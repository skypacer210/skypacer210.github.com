---
layout: post
title: "Cloud Application设计模式之Event Sourcing Pattern"
categories:
- technology
tags:
- Cloud Design Patterns
- Event-Driven


---


本文将对Event Sourcing Pattern的设计细节做一个粗略介绍~  

原文链接[Event Sourcing Pattern](http://msdn.microsoft.com/en-us/library/dn589792.aspx/)  

------ 

## 1、问题提出： ##  

大多数的应用处理数据，典型的操作就是通过更新用户的数据维护当前的状态。比如CRUD数据操作模式，即创建、读、更新和删除数据，一般的做法是从数据仓库读取数据，对数据进行修改，然后用最新的值更新数据状态，上述操作往往通过锁实现。  
CURD方式的不足之处：  
* CRUD系统直接对数据仓库进行更新操作会导致性能降低、而且不利于系统扩展。  
* 在一个协作系统里面，多个并发用户在进行数据更新的时候会发生冲突  
* 从审计的角度考虑，每个操作记录在单独的日志中，丢失了历史信息。  

## 2、解决思路： ##  

事件源模式（event souring pattern）定义了一种对序列化事件驱动数据的操作方法，每个事件都是以累加方式（append-only）进行存储。APP代码发送一系列事件至数据存储，以命令式方式描述每一个动作。同时，数据一致性在数据存储中完成。每个事件代表了对数据的一个属性的改变。  
完成一致性处理的事件存储在event store，作为数据当前状态的信任源或系统记录（或者称之为已授权数据源中的给定元素或信息片段）。典型地，Event store发布这些事件，这样能够通知到消费者并做相应的处理。作为消费者也可以对其他系统应用执行事件群的动作，或者是完成该操作所需的关联动作。我们可以看到用于产生事件的APP代码与注册了事件的系统之前是解耦的。    
通过Event store发布事件的一个典型应用就是当APP改变目录行为后对物化视图的维护，并且能够与外部系统集成。比如一个显示了所有客户订单的物化视图，用于产生UI的部分组件。当APP添加或者删除订单时，描述上述行为的事件能够被处理，以更新物化视图。    
  
![图片](/assets/images/Event_Sourcing_Pattern_1.jpg)

The Event Sourcing pattern provides many advantages, including the following:  

* Events are immutable and so can be stored using an append-only operation. The user interface, workflow, or process that initiated the action that produced the events can continue, and the tasks that handle the events can run in the background. This, combined with the fact that there is no contention during the execution of transactions, can vastly improve performance and scalability for applications, especially for the presentation level or user interface.
* Events are simple objects that describe some action that occurred, together with any associated data required to describe the action represented by the event. Events do not directly update a data store; they are simply recorded for handling at the appropriate time. These factors can simplify implementation and management.
* Events typically have meaning for a domain expert, whereas the complexity of the object-relational impedance mismatch might mean that a database table may not be clearly understood by the domain expert. Tables are artificial constructs that represent the current state of the system, not the events that occurred.
* Event sourcing can help to prevent concurrent updates from causing conflicts because it avoids the requirement to directly update objects in the data store. However, the domain model must still be designed to protect itself from requests that might result in an inconsistent state.
* The append-only storage of events provides an audit trail that can be used to monitor actions taken against a data store, regenerate the current state as materialized views or projections by replaying the events at any time, and assist in testing and debugging the system. In addition, the requirement to use compensating events to cancel changes provides a history of changes that were reversed, which would not be the case if the model simply stored the current state. The list of events can also be used to analyze application performance and detect user behavior trends, or to obtain other useful business information.
* The decoupling of the events from any tasks that perform operations in response to each event raised by the event store provides flexibility and extensibility. For example, the tasks that handle events raised by the event store are aware only of the nature of the event and the data it contains. The way that the task is executed is decoupled from the operation that triggered the event. In addition, multiple tasks can handle each event. This may enable easy integration with other services and systems that need only listen for new events raised by the event store. However, the event sourcing events tend to be very low level, and it may be necessary to generate specific integration events instead.

## 2、适用场景 ## 
  
![图片](/assets/images/Event_Sourcing_Pattern_2.jpg)

## 3、应用实例 ##  