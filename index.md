>最近将之前做过的一个游戏服务器框架整理了一遍，一来是将之前做过的技术过一遍，加深自己的印象，二来换了个行业整理下也算留作纪念了。
 #### 1. 整体框架结构图
 ![框架图](http://chuantu.biz/t6/26/1503997649x2728309531.jpg)
 >项目通讯采用http post的短连接的方式，数据格式用json。采用Nginx+Nodejs+Redis+MongoDB的架构,其中管理服和节点服是游戏的逻辑服务器，采用的是Nodejs。
- ##### Nginx
>Nginx是一款轻量级的Web 服务器/反向代理服务器。在这个项目主要有两个作用：1、文件服务器，升级文件以及配置文件等给客户端访问。2、路由分发，客户端的网络请求经过Nginx来分发给各个节点服务器。分发采用的哈希路由的方法来保证各个节点之间承载均匀。
- ##### MongoDB
>MongoDB是一个NoSQL非关系数据库，采用BSON存取，BSON即二进制的JSON,也可以理解为JSON存取，这意味着MongoDB更加灵活。MongoDB存取的key和value不是固定的数据类型和大小，所以在使用MongoDB时无须预定义关系型数据库中的”表”等数据库对象，设计数据库将变得非常方便，可以大大地提升开发效率。
- ##### Redis
>Redis 是一个高性能的key-value数据库。它是一个内存数据库，存取效率非常高。项目中主要有两个作用：1、补足MongoDB的不足。2、在节点服之间当共享内存用。Redis除了简单的key-value数据结构外还提供有丰富set、map等数据结构，可以非常方便快速的在不同的节点服务器之间维护一些公用的表。
- ##### 节点服
>节点服务器是游戏的主逻辑服务器，采用的是多进程分布式部署的结构。每个节点服都是个独立的进程，节点数量可以动态增减。
- ##### 管理服务器
>管理服只有一个，用来执行一些全局的作业等任务。

#### 2. 节点服务器
>节点服务器是游戏的主逻辑服务器。项目采用的Nodejs的Express框架。Express 是一个基于Node的极简、灵活的web应用开发框架。
- ##### 主结构图
![UML结构图](http://chuantu.biz/t6/34/1504491155x2728309382.jpg)
- ##### ServiceMgr
>ServiceMgr主要是用来服务器Service的注册和启动。是整个服务器的入口。
- ##### 路由RouteService
>RouteService负责整个服务器的路由管理，项目中主要存在3种路由请求，1、第三方回调如充值回调等web服务。2、游戏角色操作请求。3、游戏全局操作请求。游戏全局请求是service请求，RouteService将直接根据路径找到对应的方法进行调用，比如user/login这条请求，将会直接路由到UserService中的login调用。
```js
let handler = global.serviceMgr.getService(svc_name)[method_name];
if(handler){
    handler(msg);
}
```
>游戏角色的操作请求需要先定位到不同的角色对象，因此会先解析出请求数据中的角色id值，然后再根据id取得对应的actor对象，如果没有找到还需要从db中取得角色的数据再创建actor对象，而如果对应的actor一段时间都没有请求的话还需要定期的清理，ActorService就主要负责actor对象的管理。路由找到actor之后就是要对actor进行操作了，actor下又细分了很多Controller，实际的操作逻辑都是在Controller中完成，路由也是直接定位到Controller中的实现。如商店购买物品shop/buy请求将直接调用注册名为shopController中的buy函数。
```js
let handler = actor.getController(controller_name)[method_name];
if(handler){
    handler(msg);
}
```
>这种路由直接绑定类和函数的方法可以省去大段的路由定义，方便又简洁。
- ##### 属性自动同步
>这个框架的一个最大的亮点就是实现了属性的完全自动同步。服务器上角色的属性的客户端-服务器-DB端三端完全自动同步。如购买物品shop/buy操作，服务器处理完购买物品的逻辑之后，只需发送成功或失败的信息到客户端即可，客户端的物品和金钱变动以及DB端物品和金钱的变动框架会自动同步和更新，不需要额外的网络请求或DB存储调用。这样就大大的减少了客户端的逻辑以及服务器DB存储的逻辑，也大大减少bug发生的概率。代码实现：

```js
let handler = actor.getController(controller_name)[method_name];
if(handler){
    let ret = {};
    actor.startChangeSet();
    handler(msg,ret);
    ret.change_set = actor.getChangeSet();
    actor.saveChangeSet(ret.change_set);
    res.send(ret);
}
```
>所有在Controller中的属性操作都需要通过actor中封装的propset来完成，属性操作会自动记录所做的更改，从而实现自动同步的功能。
- ##### 单元测试框架
>项目框架还实现了一整套单元测试框架。所有的service和controller中都有一个test函数，用来写整个类的单元测试代码。初始在ServiceMgr中根据启动配置通过startServiceTest来启动测试流程。startServiceTest将会遍历执行所有注册的service中的test函数。而在ActorService中将创建2个测试Actor并遍历Actor中所有Controller的test函数。\
>test函数为每个类的测试单元，测试范围会覆盖类中的每一个函数，如下面是shopController中的部分单元测试代码：

```js
test()
{
    actor.getPakageController().removeAllGoods();
    actor.setMoney(money);
    this.buyGoods(id,num);
    assert.equal(this.getMoney(),0);
    assert.equal(actor.getPakageController().getGoodsNum(id),num);
    //...
}
```
- ##### 快/慢同步功能
>由于应用了属性自动同步的功能，所以一些非网络请求带来的actor数据变动将不被允许，比如我需要在每个线上的玩家每隔5分钟给他发个金币，如果直接修改角色金币数据将无法自动同步到客户端。因此加入了_quickSync和_slowSync
的功能，_quickSync每20秒一次，_slowSync每3分钟一次。由客户端发起网络请求，服务器会遍历每个controller中_quickSync/_slowSync的方法，每个controller中都可以根据自身的逻辑在函数中添加代码。
- ##### 聊天功能
>由于节点服务器是多进程的，每个节点都相对独立，互相之间也没有通讯，聊天之类的功能就无法实现，这时候就需要借助redis了。运用redis的List功能可以轻松实现chatService的功能。首先需要在redis中建立一个聊天的List和一个全局的sn号，当有一条新的聊天记录时,sn号自增（redis提供原子自增操作incr,用来防止并发出现数据错误），同时在List中插入一条带sn号的数据。每个节点服务器都会根据sn号来定时同步最新List，这样每个节点都维护有一份最新的聊天数据。客户端在_quickSync每次拉去最新的数据，客户端中也会有一份上次拉取的最新sn号，服务器会根据客户端的sn号来推送新的数据。
- ##### 排行榜功能
>排行榜功能在sql关系型数据库中一般会放在DB的作业中实现，通过半夜空闲时间点来执行作业来提取表中的排行信息。但是在项目中用的是MongoDB，MongoDB没有作业功能，因此需要换种方式来实现。项目中用的是Redis的SortedSet（有序集合）来实现。通过Redis的SortedSet，排行功能完全交给Redis来做，服务器只需在数据有改动时同步给Redis即可。
#### 3. 管理服务器
>由于节点服是多个同时启动的，很多单独的操作是不方便放在节点服去操作的，例如一些活动的开关，排名奖励的发放等等，如果放节点服，在哪个节点服执行是个问题，因此需要单独一个服务器来完成一些功能操作。将这些功能从节点服中抽离出来单独启动一个服务器来实现，有助于实现业务间的分离，减少错误发生的概率。
