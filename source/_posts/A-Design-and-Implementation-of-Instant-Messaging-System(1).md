---
title: A Design and Implementation of Instant Messaging System(1)
top: false
cover: false
toc: true
mathjax: true
date: 2021-11-01 21:30:56
password:
summary:
tags:
categories: Systems Architecture
---
In this article, I want to share my experience in designing the architecture of an instant message system. A system using this design had already been deployed and put in use, supporting several hundred thousand concurrent connections. 

## Challenges
There are mainly **four** challenges in all message systems:
1. **High-Reliability**: Make sure not to lose messages
2. **High Availability**: One or more servers hangs up will not affect the service
3. **Real-Time**: Fase respond, Online user messages can reach to client end within 1 second
4. **Orderliness**: To ensure the orderliness of user messages, there will be no disorder of sending and receiving messages.

## Overall Design

![overall.png](https://s2.loli.net/2021/12/16/feQ3LPaVE6BUlnr.png)

Many ancient architectures tend to aggregate all the features into one service, which leads to two major problems:
 1. *Code is so **complex**, which makes it extremely to develop a new feature or even to maintain.*
 2. ***Low availability**, a problem with a single business logic may affect other logic, leading to the comprehensive unavailability of services.*

In order to avoid those potential problems, while I was building the architecture, I tried to follow the best practices of microservices, broke the overall service into several relatively small three sub-services, and then into smaller microservice modules.

The responsibilities of the **three subsystems** are described as follows:
> + **Message Service**:  Implementation of business logic, for example, group message, friend list maintenance. 
> + **Push Service**: Responsible for online push and offline push of messages to clients. 
> + **Storage Service**: Persistence and cache of all the messages, both online and offline messages. Provide simple API to other services.

Next, I will illustrate those subsystems in detail.

# Subsytems
## Message Service
![Message Service](https://s2.loli.net/2021/12/20/IQeZPyVjpmbuOdW.png)
A successful message service must have two features, first is the **ability of horizontal expanding**, the second one is **flexibility**, which means new features can be added or removed easily. 
To achieve those demands, I divided the whole service into 4 parts:
1. **Service router**
2. **Message hub---Signalr**
3. **OnlineStatus Service**
4. **Feature microservices**

I will start with Message Hub

#### Message Hub --- Signalr
The company I worked in is a subsidiary split from Microsoft, which means that many other applications and services are developed using ASP.NET, such as our authentication service. 

So in order to be better compatible with previous services, I chose **ASP.NET Core Signalr**, an open-source library that simplifies adding real-time web functionality to apps.Signalr supports the **Websocket protocol** very well, it can automatically send heartbeat messages to keep the connection alive. Signalr is also integrated with ASP.NET's identity mechanism, making it convenient to manipulate the user's connection context.  

![signalr.png](https://s2.loli.net/2021/12/16/upthTJgsFLU7Pnd.png)

All messages that come from clients must be pre-processed by Signalr message hub. Signalr hub will decide what to do with this message depending on its content or header, either forward to other feature microservice modules or save to persisted databases.

The problem now is that the performance of signalr message hub is the bottleneck of the whole message service, a better method to avoid this problem is to make the message hub **scalable horizontally**, that's where *Service router* kicks in.

#### Server router
The server router's job is to **balance the load** between all the message hub instances by keeping listening to the heartbeat messages send from instances and randomly choosing a hub to process the client's connection request.

To establish a WebSocket connection, both parties to the communication must know each other's IP addresses. So, when a client login, it will *first* send a **"Get Hub Address" HTTPS request** to the server router, server router will respond to an IP of one available hub to the client. *Then* the client will try to connect to the specific hub, if failed, it will ask the server router for another available IP.

How to update the available server list in the server router? If we just simply set those IPs in the static file in the server router, when we add new hub instances or remove old ones, we may have to redeploy server routers. Also, if one of those hub instances hangs up, the server router can not remove its IP from the available list automatically. The method we use to overcome those disadvantages is to let message hub instances report their status to the server router the moment they started, after that, the server router will send heartbeat messages periodically to all the hub instances.

#### OnlineStatus Service
SignalR message hub is distributed, each of the message hub instances connects to different clients. So, to record the global online status of every user, we build a database using Redis and save the **mapping of account UUID and connection's id, IP address of the hub** to it. So in Redis, the hashmap looks like this:
```
"ac123456": //Account UUID
{
  "ConnectionId":"cn89890890",
  "HubIP":"10.2.3.45"
}
```
We also implemented **stateless online status manager** to help manipulate this Redis database. 


#### Feature microservices
Feature microservices are differents services in **Kubernetes**, they provide APIs to each other. For example, we can have *Group Manager* as a service, providing APIs like *create new group* or *add new members to the existing group*. When a message reaches the message hub, the hub will decide what microservices to call depending on its header or content. In this way, we prevent the code of the message hub from being too complicated to understand or modify in the future.


## Storage Service
![Storage Service](https://s2.loli.net/2021/12/20/eUwspo2AdPgSHc5.png)
Our goal in designing storage services is to find the balance between performance and cost. The storage service classifies and treats data differently according to the frequency of accessing the data.

With that in mind, we store offline, unreached messages and history messages separately in Redis and MongoDB. To be more precise:
>*Recent messages*: In Redis, expire in 7 days
>*History messages*: In MongoDB shared cluster 

There are basically 4 types of components in the storage service:
+ *External Accessor*: provide API to manipulate MongoDB or Redis
+ *Rabbit MQ*: as a message queue, all the requests sent to storage service will first enqueue and wait to be processed
+ *Database Reader*: receive and process request of reading data
+ *Database Writer*: receive and process request of writing data

Rabbitmq plays an extremely important role here, since we may need to support some high concurrency scenarios like hundreds of users in one group sending messages to each other, we need a message queue to shave peak so our whole system will not collapse.

To ensure every message client sends will eventually be received by destnation receiver, one of the approachs we take is to persist the messages into MongoDB before sending. If fails, client will be asked to resend this message.

## Push Service
The core task of the push system is to use **offline push** (such as iOS APNS, Huawei push, Xiaomi push, etc.) for message notification when the user is not online after receiving a request to send a downlink message to the user.
![Push Service](https://s2.loli.net/2021/12/20/jJtcub35IzphlUy.png)

Because push services may have large-scale concurrent flocks, such as when a large group of heated discussions, it will trigger a billion-level TPS. Therefore, the push service uses RabbitMQ to do peak shaving.

Here too, every module except RabbitMQ is stateless, because parallel expansion and fault tolerance can also be achieved, and any service failure does not affect the overall service availability

# Summary of this article
This article mainly summarizes the overall architecture design of this IM system that can carry 100000 DAU(Daily active users). For high performance and horizontal scalability, the entire architecture is divided into 3 subsystems based on the concept of WeChat, namely: message service, push system, storage system.

Here is a picture that roughly illustrate the process of sending a message:
![Send A Message](https://s2.loli.net/2021/12/20/eqkWKPQxJhIS26f.png)

For these 3 subsystems, at the actual technical application layer, further service splitting and refinement have been carried out, which greatly enhances the scalability of the entire architecture.

In the next article, I will elaborate the methods we use to solve some other more detailed but very important hot issues, such as: message reliability, message order, data security, mobile terminal weak network issues, etc.

If you have any suggestions or questions about this article, welcome to write your thoughts in the comment area!!!