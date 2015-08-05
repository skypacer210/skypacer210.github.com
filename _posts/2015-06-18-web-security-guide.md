---
layout: post
title: "Web App Security Consideration"
categories:
- technology 
tags:
- WEB
- HTTP
- OWASP


---
  
该文摘自[What technical details should a programmer of a web application consider before making the site public?](http://programmers.stackexchange.com/questions/46716/what-technical-details-should-a-programmer-of-a-web-application-consider-before)，实际上列出了Web安全的知识体系，也包括了工程开发当中的指导准则。     

### 接口与用户体验 ###  

线程安全是多线程程序中的编程理念，一段代码线程安全意味着在多个线程执行的时候能够运行正确。因此，它必须满足： 

- 确保你的网站能够适配所有主流的浏览器。一个最小的测试集必须包括：Firefox采用的Gecko引擎，Safari以及其他手机浏览器采用的WebKit引擎，Chrome，IE(兼容VPC图片)和Opera。同时也需考虑在不同OS上的渲染问题；
- 考虑浏览器之外的访问通道，比如蜂窝电话，Sreen Reader以及搜索引擎。这里的要点是如何在不影响用户的情况下更新功能，这里需要版本管理软件和自动化编译机制；
- 永远不要呈现给用户不友好的错误提示；
- 不要将用户的邮箱地址以明文形式呈现；
- 对于用户产生的链接加入`rel="nofollow"`属性以避免受到攻击；
- 为你的网站加入深思熟虑后的限制；
- POST成功后立即重定向，从而禁止再次提交引发刷新；
- 理解并应用渐进增强；
- 多元化用户交互，不单单依赖鼠标；
- 最后一点：别让用户思考； 



### 安全 ###

- 参考OWASP development guide，其覆盖了上层到底层的安全架构和知道准则；
- 理解SQL注入并知道如何防御；
- 永远不信任用户输入或者是来自用户的请求，包括cookie以及隐藏其后的属性值；
- 对密码进行"撒盐"哈希，不同列采用不同的“盐”以防御彩虹表攻击；选择计算较慢的哈希算法(比如bcrypt或者scrypt)来存储密码，NIST和FIPS建议采用PBKDF2对密码进行哈希。避免直接使用MD5和SHA；
- 不要试图开发你自己的验证系统，相比现有成熟的验证体系，你的方法更容易受到攻击；
- 对敏感数据输入相关的页面采用SSL/HTTPS保护；
- 预防 session hijacking；
- 避免 cross site scripting (XSS)；
- 避免 cross site request forgeries (CSRF)；
- 避免 Clickjacking；
- 确保你的系统及时更新最新的补丁；
- 确保你的DB连接信息是安全的；
- 及时追踪最新的攻击方法和手段；
- 阅读谷歌浏览器的安全手册；
- 阅读Web应用攻击者的手册；
- 考虑最小权限原则，尝试以非root权限运行你的应用服务器；
  


### 性能 ###

- 如有必要实现cache，理解并使用HTTP、HTML5的cache；
- 优化图像处理，不要在需要反复使用的场合调用20KB的照片；
- 学会如何压缩和deflate；
- 组合多个脚本文件以减少浏览器的连接数量；
- 参考Yahoo的异常性能站点，包含有多个操作指导；
- 采用CSS处理小的有相关度的图片；
- 高访问量站点需要考虑跨domain的组件拆分问题；
- 静态内容()应该放置于一个单独的域，该域不使用cookie。因为一个域及其子域中的所有cookie在每次请求发送中都会被包含。一个好的方式是采用CDN，但是必须考虑CDN失效的情形；
- 最小化渲染页面所需的HTTP请求；
- 善假于物，比如Google的Closure JavaScript编译器；
- 确保在站点的根目录下有一个`favicon.ico`文件, 比如. `/favicon.ico`。浏览器会自动请求该文件，甚至在HTML中无需提及；如果没有这样的文件` /favicon.ico`，会产生多个404错误，消耗网站带宽；



### 搜索引擎优化 ###

- 采用搜索引擎友好的URLS，比如采用`example.com/pages/45-article-title` 而不是`example.com/index.php?page=45`；  
- 对于动态内容修改，将`#`替换诶`#！`；  
- 不要采用诸如`click here`的链接；
- 设置一个XML sitemap, 缺省位置` /sitemap.xml`；
- 对于多个URL指向同一个内容，采用`<link rel="canonical" ... />`；
- 善假于物，这次是Google的 Webmaster Tools 和微软Bing的 Webmaster Tools；
- 安装Google的分析器，或者是开源的Piwik；
- 知道搜索引擎爬虫的工作原理；
- 采用301做永久重定向；



### 技术 ###


- 理解HTTP相关原理，比如GET、POST、session、cookie以及无状态；
- 根据W3C标准编写HTML和CSS，确保准确；
- 理解浏览器中JavaScript如何工作；
- 理解JavaScript, style sheets以及其他所需资源的加载过程以及对性能的影响；
- 理解JavaScript sandbox的工作原理；
- 理解JavaScript已逐步被AJAX替代；
- 理解301和302重定向的区别；
- 尽可能多的理解网站所运行的平台；
- 考虑采用Reset Style Sheet 或者normalize.css；
- 考虑JavaScript框架；
- 综合考虑perceived performance 和 JS 框架；
- 不要重复发明轮子；
- 尽可能“轻便”；