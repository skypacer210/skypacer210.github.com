---
layout: post
title: "Patterns in C-REACTOR"  
image: /image/reactor_figure_1.png  
categories:
- translation

tags:
- design pattern
- reactor

---

本文由[skypacer](http://skypacer210.github.io/)译自[Patterns in C REACTOR](http://www.adampetersen.se/Patterns%20in%20C%205,%20REACTOR.pdf)，原文作者[Adam Petersen](http://www.adampetersen.se/)。转载请注明出处！

##小引
----
本篇英文原文所发布的站点[Adam Petersen](http://www.adampetersen.se/)是一个个人网站，本文翻译的是其中的关于设计模式C实现的文章，本章主题是reactor模式。

首次正式翻译文章，水平有限，欢迎指正。


##目录
----
1、[多客户端示例](#mutil_client)    
2、[单一责任原则](#single_res)    
3、[违反Open-Closed原则](#open_closed)        
4、[从性能谈起](#performance)      
5、[问题总结](#issue_sum)    
6、[REACTOR模式](#reactor_pattern)    
&nbsp;&nbsp;&nbsp;&nbsp;6.1、[事件探测](#6.1)  
&nbsp;&nbsp;&nbsp;&nbsp;6.2、[实现机制](#6.2)   
&nbsp;&nbsp;&nbsp;&nbsp;6.3、[Reactor注册策略](#6.3)   
&nbsp;&nbsp;&nbsp;&nbsp;6.4、[Reactor实现](#6.4)       
&nbsp;&nbsp;&nbsp;&nbsp;6.5、[触发Reactor](#6.5)      
&nbsp;&nbsp;&nbsp;&nbsp;6.6、[处理注册](#6.6)    
&nbsp;&nbsp;&nbsp;&nbsp;6.7、[事件多样性](#6.7)              
7、[REACTOR vs OBSERVER](#vs)        
8、[结论](#cons)     
9、[总结](#sum)            

##正文
----
<a id='mutil_client' name='mutil_client'> </a>  

本文将研究一种适用于*event-driven*应用的模式。Reactor Pattern将应用的不同职责解耦，允许应用由多个潜在的client端分离并分发事件。


###1、多客户端示例

为了简化对大系统的维护，工程师可以单独搜集子系统的诊断信息然后再合并至中心。每个子系统利用TCP/IP连接至诊断服务器，由于TCP/IP是一个面向连接的协议，客户端（不同的子系统）不得不单独向server请求相应的连接。一旦连接建立，client可以随时发送诊断信息。  

为了解决这个问题，最容易想到的方法就是服务器依次扫描客户端的连接请求和诊断信息，如下图所示：	  

![图片](/assets/images/reactor_figure_1.png)  
<center>Figure 1: eternal loop to scan for different events</center>
  
尽管这是一个极其简化的例子，这里仍然存在不少潜在的问题。首先是该方法将应用逻辑、网络代码、事件分发代码这几个毫不相干的逻辑捆绑在一起，会带来严重的维护、测试和可扩展问题，这样的设计违反了基本的设计原则。  

<a id='mutil_client' name='mutil_client'> </a>
###2、单一责任原则

<a id='single_res' name='single_res'> </a>

单一责任原则，即一个类只能有一种原因去驱动改变，该原则的是同open-closed原则有异曲同工之妙：尽量避免现有代码的修改。当违反了单一原则的时候，一个模块有多种原因需要去修改。更糟的是，多个不同的责任集中于一个模块当中会耦合在一起，使得测试与修改变得非常复杂。  

单一责任原则本质是集中性，便于进行多层抽象，而不仅仅是一个过程上下文，简单地讲类替换为函数将有利于我们在此原则基础上分析算法。  

###3、违反Open-Closed原则

<a id='open_closed' name='open_closed'> </a>  

如果违反了Open-Closed原则，上述例子的模块将变得难以维护，或者该代码在设计之初就没打算扩展性。不幸的是，该中设计已经违反了Open-Closed原则，若不改变现有代码，新的功能将无法加入。总而言之，这种设计使得修改代码变得代价高昂。  

 
###4、从性能谈起  

<a id='performance' name='performance'> </a>  

这种方案的糟糕之处还在于整个系统的性能变得很差，当线性扫描所有事件时，尽快采用了超时机制，然而很多时间被白白浪费了。  

一个潜在的性能问题就是并发失败的处理，一种解决方案是引入多线程，诊断服务器轮询事件，但是此时的轮询代价已经只剩下处理连接请求，一旦请求建立，分配相应的线程处理该连接上的所有消息处理。  
  
然后多线程方案无法从根本上解决这种设计的问题：违反了单一责任原则和Open-Closed原则。尽管扫描代码和诊断消息处理从主循环中移除，添加一个新的服务端口仍需修改现有代码。  

从设计的角度看，线程不能改变任何事情，事实上，即使考虑到性能，这种改进导致上下文切换和同步，反而比单线程方法更糟。
  
###5、问题总结  

<a id='issue_sum' name='issue_sum'> </a>  

总结以上的经验，我们发现上述设计方南的失败之处归结于它们以三种责任为解决前提。问题即违反了Open-Closed原则，导致难于修改现有代码。  

理想的解决方案应该是易于扩展、封装和多种责任解耦，并能够同时服务多个客户端，即使不引入多线程。Reactor模式通过在事件处理中封装应用逻辑和在事件分发中隔离代码很好的解决了这些问题。  

###6、REACTOR模式  

<a id='reactor_pattern' name='reactor_pattern'> </a>  

Reactor模式定义：“reactor架构使得事件驱动型的不同应用实现分离，并将来自一个或多个客户端的服务请求分发至一个应用中”。  

![图片](/assets/images/reactor_figure_2.png)
<center>Figure 2: Structure of the REACTOR pattern</center>   

其中包含的参与者如下：  
  
-	**EventHandler**：一个EventHandler定义了一个接口，该接口由处理事件的模块实现，每个EventHandler有自己的Handle。  
-	**Handle**：**Handle** 本质是reactor模式在OS上实现的具体句柄，例如，**handle**  包含了像文件、socket和timer等系统资源。
-	**DiagnosticsServer** and **DiagnosticsClient**: 这是两种具体的事件handler，每个独立封装一个责任，为了能够接受事件通知，这些具体的事件handlers必须在 **Reactor** 中进行注册。
-	**Reactor**：**Reactor** 维护 **EventHandler** 的注册，并取出相应的 **Handles**. **Reactor** 等待已经注册 **handles** 中的事件，当 **Handle** 通知事件发生时，触发相应的 **EventHandler** 。
  
####6.1、事件探测  

<a id='6.1' name='6.1'> </a>  

在一些文献中关于 **reactor** 的描述中，定义了一个工具，即 **synchronous Event Demultiplexer **， 该分离器由 **reactor** 调用等待已经注册的 **Handles** 的事件。 

该事件分离器一般由操作系统提供，比如 *poll()* ，*select()* 和 *WaitForMutipleObjects()* 。
  
####6.2、实现机制  

<a id='6.2' name='6.2'> </a>    

**EventHandler** 和 **Reactor** 之间的合作关系类似OBSERVER模式中的observer和它的对象。    

为了将Reactor和它的事件处理器解耦，同时Reactor仍能够通知到他们，每个具体的事件处理器必须关联一个唯一的instance。这里的C实现中，采用void *作为通用类型以描述 **EventHandler** 接口。  

-	事件处理接口：EventHandler.h  

{% highlight objc %}
 14 #ifndef EVENT_HANDLER_H
 15 #define EVENT_HANDLER_H
 16 
 17 #include "Handle.h"
 18 
 19 /* All interaction from Reactor to an event handler goes 
 20 through function pointers with the following signatures: */
 21 typedef Handle (*getHandleFunc)(void *instance);
 22 typedef void (*handleEventFunc)(void *instance);
 23 
 24 typedef struct {
 25 .       void *instance;
 26 .       getHandleFunc getHandle;
 27 .       handleEventFunc handleEvent;
 28 } EventHandler;
 29 
 30 #endif
{% endhighlight %}

-	Reactor注册接口：ractor.h  

{% highlight c %}
 14 #ifndef REACTOR_H
 15 #define REACTOR_H
 16 
 17 #include "EventHandler.h"
 18 
 19 void Register(EventHandler* handler);
 20 void Unregister(EventHandler* handler);
 21 
 22 #endif
{% endhighlight %}  

-	具体的事件处理器：DiagnosticsServer.c
  
{% highlight c %}
 14 /******************************************************************************
 15 * A simulation of a diagnostics server.
 16 *
 17 * A client that wants to log a diagnostic connects to this server using TCP/IP.
 18 * The server gets notified about a connecting client through the Reactor.
 19 * Upon such a notification, the server creates a client representation.
 20 *******************************************************************************/
 21 
 22 #include "base.h"
 23 #include "DiagnosticsServer.h"
 24 #include "DiagnosticsClient.h"
 25 #include "EventHandler.h"
 26 #include "ServerEventNotifier.h"
 27 #include "Reactor.h"
 28 
 29 #include "Error.h"
 30 #include "TcpServer.h"
 31 
 32 #define MAX_NO_OF_CLIENTS 10
 33 
 34 struct DiagnosticsServer {
 35 .       Handle listeningSocket;
 36 .       EventHandler eventHandler;
 37 
 38 .       /* All connected clients able to send diagnostics messages. */
 39 .       DiagnosticsClientPtr clients[MAX_NO_OF_CLIENTS];
 40 };
 41 
 42 /************************************************************
 43 * Function declarations.
 44 ************************************************************/
 45 
 46 static void deleteAllClients(DiagnosticsServerPtr server);
 47 
 48 static int matchControlledClientByPointer(const DiagnosticsServerPtr server, const DiagnosticsClientPtr clientToMatch);
 49 
 50 static int findFreeClientSlot(const DiagnosticsServerPtr server);
 51 
 52 static int findMatchingClientSlot(const DiagnosticsServerPtr server, const DiagnosticsClientPtr client);
 53 
 54 static Handle getServerSocket(void* instance);
 55 
 56 static void handleConnectRequest(void* instance);
 57 
 58 static void onClientClosed(void* server, void* closedClient);
 59 
 60 /************************************************************
 61 * Implementation of the EventHandler interface.
 62 ************************************************************/
 63 
 64 static Handle getServerSocket(void* instance)
 65 {
 66    const DiagnosticsServerPtr server = instance;
 67    return server->listeningSocket;
 68 }
 69 
 70 static void handleConnectRequest(void* instance)
 71 {
 72 .       DiagnosticsServerPtr server = instance;
 73 
 74 .       const int freeSlot = findFreeClientSlot(server);
 75 
 76 .       if(0 <= freeSlot) {
 77 .       .       /* Define a callback for events requiring the actions of the server (for example 
 78 .       .          a closed connection). */
 79 .       .       ServerEventNotifier eventNotifier = {0};
 80 .       .       eventNotifier.server = server;
 81 .       .       eventNotifier.onClientClosed = onClientClosed;
 82 
 83 .       .       server->clients[freeSlot] = createClient(server->listeningSocket, &eventNotifier);
 84 
 85 .       .       ne_debug(REACTOR_DEBUG, "Server: Incoming connect request accepted\n");
 86 .       }
 87 .       else {
 88 .       .       ne_debug(REACTOR_DEBUG, "Server: Not space for more clients\n");
 89 .       }
 90 }
 91 
 92 /************************************************************
 93 * Implementation of the ServerEventNotifier interface. 
 94 ************************************************************/
 95 
 96 /**
 97 * This function is invoked as a callback from the client representation 
 98 * in case it detects a disconnect on TCP level.
 99 */
100 static void onClientClosed(void* server,
101 .       .       void* closedClient)
102 {
103 .       DiagnosticsServerPtr serverInstance = server;
104 .       DiagnosticsClientPtr clientInstance = closedClient;
105 
106 .       const int clientSlot = findMatchingClientSlot(serverInstance, clientInstance);
107 
108 .       if(0 > clientSlot) {
109 .       .       ne_debug(REACTOR_DEBUG, "Phantom client detected\n");
110 .       .       return;
111 .       }
112 
113 .       destroyClient(clientInstance);
114 
115 .       serverInstance->clients[clientSlot] = NULL;
116 }
117 
118 /************************************************************
119 * Implementation of the ADT functions of the server. 
120 ************************************************************/
121 
122 /**
123 * Creates a server listening for connect requests on the given port.
124 * The server registers itself at the Reactor upon creation.
125 */
126 DiagnosticsServerPtr createServer(unsigned int tcpPort)
127 {
128 .       DiagnosticsServerPtr newServer = malloc(sizeof *newServer);
129 
130 .       if(NULL != newServer) {
131 .       .       int i = 0;
132 
133 .       .       for(i = 0; i < MAX_NO_OF_CLIENTS; ++i) {
134 .       .       .       newServer->clients[i] = NULL;
135 .       .       }
136 
137 .       .       newServer->listeningSocket = createServerSocket(tcpPort);
138 .       .       if (newServer->listeningSocket < 0) {
139 .       .       .       ne_debug(REACTOR_DEBUG, "fail create sock=%d.\n", newServer->listeningSocket);
140 .       .       .       return NULL;
141 .       .       }
142 
143 .       .       /* Successfully created -> register the listening socket. */
144 .       .       newServer->eventHandler.instance = newServer;
145 .       .       newServer->eventHandler.getHandle = getServerSocket;
146 .       .       newServer->eventHandler.handleEvent = handleConnectRequest;
147 
148 .       .       Register(&newServer->eventHandler);
149 .       .       ne_debug(REACTOR_DEBUG, "register event sock=%d.\n", newServer->listeningSocket);
150 .       }
151 
152 .       return newServer;
153 }
154 
155 /**
156 * Unregisters at the Reactor and deletes all connected clients 
157 * before the server itself is disposed.
158 * After completion of this function, the server-handle is invalid.
159 */
160 void destroyServer(DiagnosticsServerPtr server)
161 {
162 .       deleteAllClients(server);
163 
164 .       /* Before deleting the server we have to unregister at the Reactor. */
165 .       Unregister(&server->eventHandler);
166 
167 .       disposeServerSocket(server->listeningSocket);
168 .       free(server);
169 }
170 
171 /************************************************************
172 * Definition of the local utility functions.
173 ************************************************************/
174 
175 static void deleteAllClients(DiagnosticsServerPtr server)
176 {
177 .       int i = 0;
178 
179 .       for(i = 0; i < MAX_NO_OF_CLIENTS; ++i) {
180 .       .       if(NULL != server->clients[i]) {
181 .       .       .       destroyClient(server->clients[i]);
182 .       .       }
183 .       }
184 }
185 
186 /**
187 * Returns the index where a client matching the given pointer is found.
188 * Returns -1 if no match was found. 
189 */
190 static int matchControlledClientByPointer(const DiagnosticsServerPtr server,
191 .       .       const DiagnosticsClientPtr clientToMatch)
192 {
193 .       int clientSlot = -1;
194 .       int slotFound = 0;
195 .       int i = 0;
196 
197 .       for(i = 0; (i < MAX_NO_OF_CLIENTS) && (0 == slotFound); ++i) {
198 
199 .       .       if(clientToMatch == server->clients[i]) {
200 .       .       .       clientSlot = i;
201 .       .       .       slotFound = 1;
202 .       .       }
203 .       }
204 
205 .       return clientSlot;
206 }
207 
208 static int findFreeClientSlot(const DiagnosticsServerPtr server)
209 {
210 .       return matchControlledClientByPointer(server, NULL);
211 }
212 
213 static int findMatchingClientSlot(const DiagnosticsServerPtr server,
214 .       .       const DiagnosticsClientPtr client)
215 {  
216 .       return matchControlledClientByPointer(server, client);
217 }
                                                                                                                                                                                                                                                                                                         {% endhighlight %}
    

####6.3、Reactor注册策略

<a id='6.3' name='6.3'> </a>  
  
当实现具体的事件处理器时定义抽象数据结构，这样做的好处是封装具体的注册处理细节，这样就隐藏了具体的信息，client甚至无需了解如何同reactor交互。  

Reactor的另外一个好处体现在server内部，比如说对handle的封装通过getServerSocket获得，这样我们为reactor提供一个方式获取handle；同时，reactor对事件处理器进行访问控制：只有注册的事件处理器方可被调用相应的handle及其相关资源。  


####6.4、Reactor实现  

<a id='6.4' name='6.4'> </a>  

Reactor的实现依赖于具体的同步事件分发器，如果OS提供了多种同步事件分发器，比如select()和poll()，Reactor就需要针对这些分发器实现多个reactor实例，并由链接器根据问题选择，即所谓的链接多态。  

每种Reactor的实现必须确定该应用下所需要的reactor实例。在多数场景中，应用只需一个reactor实例，即single reactor。如果应用需要多个reactor实例，则可以对reactor本身进行抽象。  

为了独立于具体的分离机制，Reactor必须维护一个已注册具体的事件处理器集合。简单地做法就是采用数组，适用于最大的客户端数目已知的情形。  


基于Poll()实现的Reactor：PollReactor.c  

{% highlight c %}

19 #include "base.h"
 20 #include "Reactor.h"
 21 #include "ReactorEventLoop.h"
 22 
 23 #include "Error.h"
 24 
 25 #include <sys/poll.h>
 26 #define POLLRDNORM      0x0040          /* non-OOB/URG data available */
 27 
 28 #include <assert.h>
 29 #include <stddef.h>
 30 #include <stdio.h>
 31 
 32 #define MAX_NO_OF_HANDLES 32
 33 
 34 /* POSIX requires <poll.h> to define INFTIM. 
 35    However, in my environment it doesn't -> follow the recommendations by Mr. Stevens.
 36 */
 37 #ifndef INFTIM
 38 #define INFTIM -1
 39 #endif
 40 
 41 /* Bind an event handler to the struct used to interface poll(). */
 42 typedef struct
 43 {
 44    int isUsed;
 45    EventHandler handler;
 46    struct pollfd fd;
 47 } HandlerRegistration;
 48 
 49 static HandlerRegistration registeredHandlers[MAX_NO_OF_HANDLES];
 50 
 51 /************************************************************
 52 * Function declarations.
 53 ************************************************************/
 54 
 55 /* Add a copy of the given handler to the first free position in registeredHandlers. */
 56 static int addToRegistry(EventHandler* handler);
 57 
 58 /* Identify the event handler in the registeredHandlers and remove it. */
 59 static int removeFromRegistry(EventHandler* handler);
 60 
 61 /* Add a copy of all registered handlers to the given array. */
 62 static size_t buildPollArray(struct pollfd* fds);
 63 
 64 /* Identify the event handler corresponding to the given descriptor in the registeredHandlers. */
 65 static EventHandler* findHandler(int fd);
 66 
 67 /* Detect all signalled handles and invoke their corresponding event handlers. */
 68 static void dispatchSignalledHandles(const struct pollfd* fds, size_t noOfHandles);
 69 
 70 /****************************************************************
 71 * Implementation of the Reactor interface used for registrations.
 72 *****************************************************************/
 73 
 74 void Register(EventHandler* handler)
 75 {
 76 .       assert(NULL != handler);
 77 
 78 .       if(!addToRegistry(handler)) {
 79 .       .       /* NOTE: In production code, this error should be delegated to the client instead. */
 80 .       .       ne_debug(REACTOR_DEBUG, "No more registrations possible\n");
 81 .       .       return;
 82 .       }
 83 }
 84 
 85 void Unregister(EventHandler* handler)
 86 {
 87 .       assert(NULL != handler);
 88 
 89 .       if(!removeFromRegistry(handler)) {
 90 .       .       /* NOTE: In production code, this error should be delegated to the client instead. */
 91 .       .       ne_debug(REACTOR_DEBUG,"Event handler not registered\n");
 92 .       .       return;
 93 .       }
 94 }
 95 
 96 /****************************************************************
 97 * Implementation of the reactive event loop 
 98 * (interface separated in ReactorEventLoop.h).
 99 *****************************************************************/
100 
101 void HandleEvents(void)
102 {
103 .       /* Build the array required to interact with poll().*/
104 .       struct pollfd fds[MAX_NO_OF_HANDLES] = {0};
105 
106 .       const size_t noOfHandles = buildPollArray(fds);
107 
108 .       /* Inoke the synchronous event demultiplexer.*/
109 .       if(0 < poll(fds, noOfHandles, INFTIM)){
110 //.     if(0 < poll(fds, noOfHandles, 5000)){
111 .       .       /* Identify all signalled handles and invoke the event handler associated with each one. */
112 .       .       dispatchSignalledHandles(fds, noOfHandles);
113 .       }
114 .       else{
115 .       .       ne_debug(REACTOR_DEBUG, "Poll failure\n");
116 .       .       return;
117 .       }
118 }
119 
120 /************************************************************
121 * Definition of the local utility functions.
122 ************************************************************/
123 
124 /**
125 * Add a copy of the given handler to the first free position in registeredHandlers.
126 */
127 static int addToRegistry(EventHandler* handler)
128 {
129 .       /* Identify the first free position. */
130 .       int isRegistered = 0;
131 .       int i = 0;
132 
133 .       for(i = 0; (i < MAX_NO_OF_HANDLES) && (0 == isRegistered); ++i) {
134 .       .       if(0 == registeredHandlers[i].isUsed) {
135 .       .       .       /* A free position found. */
136 .       .       .       HandlerRegistration* freeEntry = &registeredHandlers[i];
137 
138 .       .       .       /* Create a binding between the handle provided by the client and the events of interest. */
139 .       .       .       freeEntry->handler = *handler;
140 .       .       .       freeEntry->fd.fd = handler->getHandle(handler->instance);
141 .       .       .       freeEntry->fd.events = POLLRDNORM;
142 
143 .       .       .       isRegistered = freeEntry->isUsed = 1;
144 
145 .       .       .       ne_debug(REACTOR_DEBUG, "Reactor: Added handle with ID=%d\n", freeEntry->fd.fd);
146 .       .       }
147 .       }
148 
149 .       return isRegistered;
150 }
151 
152 /**
153 * Identify the event handler in the registeredHandlers and remove it.
154 */
155 static int removeFromRegistry(EventHandler* handler)
156 {
157 .       /* Identify the event handler in the registeredHandlers and remove it. */
158 .       int i = 0;
159 .       int nodeRemoved = 0;
160 
161 .       for(i = 0; (i < MAX_NO_OF_HANDLES) && (0 == nodeRemoved); ++i) {
162 .       .       if(registeredHandlers[i].isUsed && (registeredHandlers[i].handler.instance == handler->instance)) {
163 .       .       .       /* The given event handler is found -> mark it as unused and terminate the loop. */
164 .       .       .       registeredHandlers[i].isUsed = 0;
165 .       .       .       nodeRemoved = 1;
166 
167 .       .       .       ne_debug(REACTOR_DEBUG, "Reactor: Removed event handler with ID=%d\n", registeredHandlers[i].fd.fd);
168 .       .       }
169 .       }
170 
171 .       return nodeRemoved;
172 }
173 
174 /**
175 * Add a copy of all registered handlers to the given array.
176 */
177 static size_t buildPollArray(struct pollfd* fds)
178 {
179 .       /* Add all registered handlers to the given array. */
180 .       int i = 0;
181 .       size_t noOfCopiedHandles = 0;
182 
183 .       for(i = 0; i < MAX_NO_OF_HANDLES; ++i) {
184 .       .       if(registeredHandlers[i].isUsed) {
185 .       .       .       fds[noOfCopiedHandles] = registeredHandlers[i].fd;
186 .       .       .       ++noOfCopiedHandles;
187 .       .       }
188 .       }
189 
190 .       return noOfCopiedHandles;
191 }
192 
193 /**
194 * Identify the event handler corresponding to the given descriptor in the registeredHandlers.
195 */
196 static EventHandler* findHandler(int fd)
197 {
198 .       EventHandler* matchingHandler = NULL;
199 
200 .       /* Identify the corresponding concrete event handler in the registeredHandles. */
201 .       int i = 0;
202 
203 .       for(i = 0; (i < MAX_NO_OF_HANDLES) && (NULL == matchingHandler); ++i) {
204 .       .       if(registeredHandlers[i].isUsed && (registeredHandlers[i].fd.fd == fd)) {
205 .       .       .       matchingHandler = &registeredHandlers[i].handler;
206 .       .       }
207 .       }
208 
209 .       return matchingHandler;
210 }
211 
212 /**
213 * Loop through all handles. Upon detection of a handle signalled by poll(), its 
214 * corresponding event handler is fetched and invoked.
215 */
216 static void dispatchSignalledHandles(const struct pollfd* fds, size_t noOfHandles)
217 {
218 .       size_t i = 0;
219 
220 .       for(i = 0; i < noOfHandles; ++i) {
221 .       .       /* Detect all signalled handles. */
222 .       .       if((POLLRDNORM | POLLERR) & fds[i].revents) {
223 .       .       .       /* This handle is signalled -> find and invoke its corresponding event handler. */
224 .       .       .       EventHandler* signalledHandler = findHandler(fds[i].fd);
225 
226 .       .       .       if(NULL != signalledHandler){
227 .       .       .       .       signalledHandler->handleEvent(signalledHandler->instance);
228 .       .       .       }
229 .       .       }
230 .       }
231 }
232 

{% endhighlight %}


####6.5、触发Reactor  

<a id='6.5' name='6.5'> </a>  

反应事件轮询是Reactor的核心，其职责在于根据具体的事件处理器控制已注册事件的分离和分发。事件循环典型由main()函数调用。  

驱动reactor的客户端代码如下：  

{% highlight c %}  

 13 #include "base.h"
 14 #include "DiagnosticsServer.h"
 15 #include "ReactorEventLoop.h"
 16 
 17 #include "Error.h"
 18 
 19 void reactor_test(void)
 20 {
 21 .       /* 
 22 .        * Create a server and enter an eternal event loop, watching 
 23 .        * the Reactor do the rest. 
 24 .        */
 25 
 26 .       const unsigned int serverPort = 0xC001;
 27 .       DiagnosticsServerPtr server = createServer(serverPort);
 28 .       if(NULL == server) {
 29 .       .       ne_debug(REACTOR_DEBUG, "Failed to create the server\n");
 30 .       .       return;
 31 .       }
 32 
 33 .       /* Enter the eternal reactive event loop. */
 34 .       for(;;){
 35 .       .       HandleEvents();
 36 .       }
 37 }  
    
{% endhighlight %}


为了整体设计，需要单独为HandleEvents()创建一个文件实现，PollReactor.c，其中每个元素都将已注册事件的句柄和用于同poll()交互的结构绑定在一起。另外一种替代方案是维护两个不同的链表以确保二者的一致性。UNIX实现采用select()，定义为“一个以UNIX I/O句柄值为索引的数组，范围为0至FD_SETSIZE-1”。  

在本实现中，通过将poll结构和注册组合在一起，由于采用数组用于同poll()交互，因此该数组必须在每次事件轮询进入的时刻建立。  


####6.6、处理注册  

<a id='6.6' name='6.6'> </a>  

在本文的reactor实现中，server通过创建一个新的client来响应通知，这样一来，必须注册方可再次激活。  

一种解决方案是维护一个单独的数组用于同同步事件分离器进行交互, 该数组在事件循环中不能被修改。  

而句柄的实现则依赖平台，句柄ID有可能被重用；在拷贝中的一个信号句柄有可能属于一个未注册的事件句柄，但是由于注册重用了句柄ID，新的事件处理器可能被错误触发。我们可以加入哨兵数据以标识是否重用来解决该问题。  


####6.7、事件多样性  

<a id='6.7' name='6.7'> </a>  

实例代码仅仅考虑了一种类型的事件（read事件），其类型是hardcoded。Reactor本身并未受类型限制，能够很好的支持多种类型。  

两种通用的事件通知分发机制：    

-	Single-method interface：所有的事件由单个函数通知事件处理器，函数中只需传入事件类型即可(enum)，其不足之处在于加入了额外的控制逻辑，难于维护。  
-	Multi-method interface：事件处理器为每种支持的事件声明各自的函数（比如会所handleRead, handleWrite, handleTimeout）。一旦Reactor获取已发生事件的类型，它立即触发对应的事件处理函数，这样避免了通过指针重新创建事件的额外开销。  


###7、REACTOR vs OBSERVER  

<a id='vs' name='vs'> </a>  

尽管用于实现二者的机制相关，但是仍存在差异，主要的不同点在于通知机制。在OBSERVER实现中，当一个Subject改变其状态时，它的所有依赖者（observers)都被通知。而在REACTOR实现中，通知的关系式一对一，即一个已探测到的事件导致Reactor通知其对应的实例（EventHandler）。  

另外，一个典型的OBSERVER模式下的subject是低内聚的，除了服务其核心目的，一个subject也会负责管理和通知observers。相反，一个Reactor则仅仅分发已经注册的处理器。


###8、结论  

<a id='cons' name='cons'> </a>  

应用REACTOR模式的主要结论如下：  

-	遵循single-responsibility原则的好处：采用REACTOR模式，每种责任被封装并相互之间解耦，导致高内聚，因此简化了后续维护。比如在事件探测中平台相关的代码可以从应用中解耦，极大的方便了单元测试。  
-	遵循open-closed原则的好处：新的责任只需创建新的事件处理器，无需影响现有代码。  
-	统一了事件处理：尽管REACTOR模式集中于句柄，但其可以扩展至其他任务。Reactor加入timer支持（平台相关，比如基于信号或者线程），当同步事件分离器触发后可以设定一个超时，以避免重入问题和竞争条件。  
-	提供了一种并行读的方案：采用REACTOR方案可以有效避免并行读事件处理中的阻塞现象，其本质是一个非抢占式的多任务模型，每个具体的事件处理器必须确保其不能执行可能导致其余事件处理器饥饿的操作。  
-	类型安全的折中：由于所有的事件处理器抽象为void *，当由void指针转化为具体的事件处理指针时，编译器并没有相应的机制处理转化错误。同样的问题在OBSERVER模式中也存在，解决方法都是一致的：为不同类型的事件处理器定义单独的通知函数，利用EventHandler绑定事件处理器和其他函数。  


###9、总结  

<a id='sum' name='sum'> </a>  

Reactor模式通过对不同责任的解耦和不同模块的封装简化了事件驱动型应用的设计。关于该模式更多的讨论可以参考面向对象的软件模式卷2。  	
  