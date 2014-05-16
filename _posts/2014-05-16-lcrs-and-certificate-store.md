---
layout: post
title: "LCRS and certificate store"
categories:
- data structure
tags:
- LCRS
- certificate


---
## How it works  

###1. LCRS  

LCRS, 即left-child, right-sibling representation is a way of encoding a multi-way tree using a binary tree  

-	树的本质在于用继承性的关系描述数据结构，只需理解继承性即可理解树
-	Binary Tree：意味着每个节点最多有两个孩子
-	Multi-way tree: 意味着每个节点有任意数目的孩子
-	LCRS仅仅是一种编码形式  

###2. 为什么要用到LCRS Tree
	
传统的Mutil-way Tree
<pre><code>
 	               A       
                 //|\ \
               / / | \  \
              B C  D  E  F
              |   /|\   / \
              G  H I J K   L
</code></pre>  

数据结构表达：  

<pre><code>
struct Node {
    DataType value;
    std::vector<Node*> children;  /*用到多个孩子指针*/
};
</code></pre> 

每个节点存储一个是leftmost child，另外一个指针指向其右兄弟节点，形式变化如下：  
	   
<pre><code>
            A
           /
          /
         /
        B -> C -> D -> E -> F
       /         /           /
      G         H->I->J   K->L
</code></pre>

层次和结构不变，变的是存储方式。可以理解为double chained  tree:

 - 每个节点的leftmost child都链接在单链表里， 用child域存储下一个child，在tree的层次逐层增加；
 - 每个节点的兄弟节点都链接在另外一个单链表里，用sibling域存储该节点的兄弟节点，在tree中的层次是一样的

遍历

From a parent node to its kth child (zero-indexed):

Descend至当前节点的leftmost child,即child向右遍历该节点的兄弟节点k次返回节点,
数据结构描述相应变为：

<pre><code>  
struct Node {	
	DataType data;
	Node* leftChild;/**/
	Node* rightSibling; /*存储着*/
};  
</code></pre> 

Multi-way tree VS LCSR tree  

* 内存空间：后者比前者节省大约40%    
* 时间：如果分叉因子很多，即每层的节点很多，前者查询节点只需找到对应的子节点指针即可，后者则需进行单链表遍历（树退化为链表），导致时间变长  

###3. LCRS适用场景:  

 - Memory is extremely scarce: 如果在主存中存储一颗巨大多路树，LCRS会合适
 - Random access of a node’s children is not required  

在构建某些特定的数据结构是，比如Heap data structures ，采样LCRS可以优化空间，而最常见操作集中如下：删除tree的根节点和处理一个孩子；合并两颗树，而这二者对于LCRS易于完成，对于certificate存储系统，该操作常见。

## LCRS在证书系统中的应用  

###1. 数据结构

<pre><code>
#define CTREE_CLASS_CERT.       0
#define CTREE_C_SCLASS_PUBLIC.  0x0001.
#define CTREE_C_SCLASS_PRIVATE. 0x0002.
#define CTREE_C_SCLASS_BOTH.    0x0003.
#define CTREE_CLASS_ROOT.       1
#define CTREE_CLASS_FOLDER.     2.     
#define CTREE_F_SCLASS_COMMON.  0x0020.
#define CTREE_F_SCLASS_UNKNOWN. 0x0021

struct tree {
	struct tree *parent;
	struct tree *child;	/*leftmost child, as the sibling’s list head*/
	struct tree *sibling;	/*link to child in single list structure*/
	uchar.  name[64];		/*file name, eg certificate name*/
	uint .  id;	/* Must be > 0 */
	ushort. type, subtype;	/* both for reserved */
	ushort .iclass, subclass;	/*cert/root/folder; private key/public key*/
	uchar.  num_child, level, unused[2];	/*tree level*/
	uint .  capability;	/*reserved*/
	pBmp.   EBmp, CBmp;	/*Icon*/
	void.   *iprivate;	/*for unkown CA folder and private info*/
};   

</code></pre>   

###2. API

###2.1、Insert node  

###2.2、remove node  
   
<pre><code>
STATIC void

ctree_remove_node(struct tree *troot, uchar *filename)
{
        struct tree *delte, *parent, *child, *lost, *lastchild;
        delte = tree_find_node_by_name(troot, filename, 1);/*获取要删除的节点*/

        if (delte == 0)
                return;

        parent = delte->parent;         /*取得被删节点的父指针*/
        child = parent->child;          /*取得leftmost child指针*/

        if (parent->child == delte)     /*Case1: 被删节点就是leftmost child*/
                parent->child = delte->sibling; /* 修改leftmost child指向被删节点的兄弟*/
        else {  /*Case1: 被删节点是sibling节点*/
                child = parent->child;  /*以leftmost child为单链表头遍历，获取要删除的sibling节点 */
                while (child->sibling != delte)
                        child = child->sibling;
                ASSERT(child->sibling == delte);
                if (child->sibling != delte)
                        return;
                child->sibling = delte->sibling;
        }

        delte->parent = 0;
        delte->sibling = 0;
        if (delte->child == 0)  /*如果被删节点没有孩子，则直接返回*/
                return;

        lost = (struct tree *) troot->iprivate; /*重新连接整个子树至unkown CA*/

        //re-link child
        child = delte->child;

        do {    /*将该层所有的孩子节点的父指针修改为lost,即unkown CA*/
                child->parent = lost;
                lastchild = child; /*记录该层最后一个孩子节点*/
                child = child->sibling;/*单链表遍历*/
        } while (child);

        lastchild->sibling = lost->child; /*最后一个孩子节点的next指针(sibling)指向lost的leftmost child*/

        lost->child = delte->child; /*修改lost的leftmost child为被删节点的leftmost child*/

        delte->child = 0;

        ctree_free_node(delte);

        tree_update_order(lost); /*更新以lost为root的子树*/
}

</code></pre>
   
###2.3、update subtree:     

<pre><code>  
void tree_update_order(struct tree *parent)
{	
	struct tree *child;
	uint.   previd = 0;.   
		
	PRINTF("TW: parent update [%s] (lv %d, id %x)\n", parent->name, parent->level, parent->id);
	
	parent->num_child = 0;
	
	while (child = tree_next_child(parent, previd)) { /*获取该层的所有孩子*/
		child->level = parent->level + 1; /*孩子的层是父亲层加1*/
		child->id = tree_calc_orderid(parent);
		PRINTF("TW: child [%s] lv %d id %x\n", child->name,  child->level, child->id);

		tree_update_order(child); /*child作为子树root递归调用*/
		previd = child->id; /*更新前一个id*/
		child = child->sibling; /*单链表遍历*/
	}

	PRINTF("TW: [%s] return \n", parent->name);
}  

</code></pre>  

----
## 总结