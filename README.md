# zookeeper client 设计开发手册   
本手册是根据zookeeper服务特性及官方SDK特点，编写的一套开发手册。针对碰到的大量**陷阱**，提出注意点，并对编写生产可用的sdk提供参考建议。

**本设计（语言基于C++）是基于官方C版Client库而写，对于golang、java及其它语言均有借鉴意见。** 由于竞业协议，代码不便公开，请谅解。   

**实现了服务注册，服务发现，负载均衡 功能，具体包含：主从节点(主节点优先)，最优节点(负载较优)，连接池和地址池(轮训)**。

在开发sdk的过程中，浏览了很多github上的开源库（主要是C和C++），均存在bug，无法生产使用。后来参考了这个库：https://github.com/owenliang/zkclient.git 因为上面这个库，文档详细，但是bug依然很多，并没有解决zookeepeer的一些缺陷。   

名次定义：  
> 会话恢复：指在旧会话超时，重建新会话时后，恢复必要的 临时节点以及订阅。  
> 调用线程：指使用库zookeeperClient的上层程序，即开发者程序。  
> 完成回调：异步接口完成时 调用的回调函数。  
> watch回调：数据及子节点变化时 收到通知后调用的回调函数。  
> 持续订阅：订阅一次，往后持续推送变化。  
> zk: zookeeper服务器。  
> tmpnode: 临时节点。  
> 业务路径：所有服务归类，按照业务划分，例如/bigSys/smallSys/serviceType/servieName。bigSys是大系统划分，smallSys是小系统划分，serviceType是具体的服务类型的统称，servieName是具体服务（必须唯一）  


## 一、**zookeeper介绍**    
首先，zookeeper以会话作为上下文管理的手段。顺序保证，临时节点管理，watch监控都以会话作依据。  

---  
## 二、**根据zookeeper和官方库的特点，提出如下注意点**       
1. clientid只有在收到事件ZOO_CONNECTED_STATE才能获取成功，并且只在会话有效期内可用。   
   一旦会话失效的，要想利用clientid重建会话，不可行。   
2. zk的临时节点以及订阅请求 归属于会话，一旦会话失效，将无效。   
   zk只能完成单次订阅，需要再次订阅必须再次请求。   
3. zk的会话超时检测发生在服务器端。客户端只有在重连上服务器时，才被告知会话超时，并强制关闭连接。    
4. 会话失效后zhandle必须调用zookeeper_close释放资源，防止资源泄漏。并且使用已关闭的zhandle会报错。     
   zhandle存在并发问题。如果想在回调线程中处理重连问题，调用线程和回调线程需要加锁(会话超时时会触发回调函数，依次调用是zookeeper_close，zookeeper_init，并重新得到新的zhandle）。   
5. 由于zookeeper_init是异步的，所以返回ZOK并不代表连接成功。  
   程序启动时首次调用zookeeper_init时，应该用带超时的信号量等待异步事件ZOO_CONNECTED_STATE。  
   会话超时后启动回调函数重连，调用zookeeper_init后无法立即“恢复会话”，必须在收到事件ZOO_CONNECTED_STATE才能恢复会话(包括临时节点及订阅)。  
6. zk的所有回调在一个线程内调度，不允许阻塞等待。  
   zookeeper_init返回zhandle，但是可能在返回前触发回调函数，所以回调函数中必须用参数zhandle，而不是zookeeper_init返回的全局zhandle。  
   >**同步接口**：调用线程及watch回调线程 的时序不定，使用者必须自己同步。  
   **异步接口**：调用线程，完成回调，watch回调(其中 完成回调和watch回调在一个线程内，并且完成回调总是早于watch回调)，完成回调和watch回调不存在时序问题。  
   要是调用接口返回失败，完成回调和watch回调 都不会被触发。  
   因此，在设计持续订阅接口时，可以先调用同步接口(加锁)，然后之后在watch回调中调用异步接口(无需加锁)。  
7. 两次会话之间（老会话创建，新会话还未创建），数据发生变化(客户端在订阅了数据变化的操作后会话超时，此时若节点数据变化了，在客户端重现建立新会话并且重新订阅watch后，客户端无法感知该数据变化)。  
8. zk不支持持续订阅。  
   要实现支持持续订阅，必须考虑再次订阅时订阅失败的问题(客户端资源不足等)，以及由于服务器资源而主动取消订阅的问题(此时收到NoWatch事件)。这个可以由单独的线程定期处理。  
   对同一path订阅多次(exist和get等同)，只推送一次通知。这个可以客户端自己实现。  
9. zk集群节点越多，同步时间越长，写性能越差。zk连接数越多，心跳检测即会话检测越多，性能越差。observer可以解决后者。  
10. zk订阅子节点变化，在收到子节点变化，获取子节点列表，获取子节点数据之间，存在不一致问题，尤其是获取子节点列表成功，而获取子节点内容失败，此时不会再次通知。  
11. 回调分成sessionWatcher和otherWatcher两种，前者主要收到state事件，后者收到seesion_event和type事件。  
   前者用来处理会话问题，后者用来处理订阅事件(node和 children)。  
   无法在otherWatcher处理 持续订阅：   
    >**原因一是**，所有回调的在一个线程里调度，所以在otherWatcher阻塞等待直到成功为止，这种方式时不正确的，会阻碍别的回调的进行。  
    **原因二是**，otherWatcher收到session_event时，老会话超时新会话未知，这种情况下再次订阅无意义。  
12. 由sessionWatcher处理重连以及持续订阅。  
   在sessionWatcher回调中zookeeper_init失败可以循环阻塞等待，因为这个操作失败，往后的一切回调无意义，所以可以阻塞。  
13. 临时节点一般分为 两种用途。  
    >第一种是代表进程的状态，这种临时节点必须在进程生命周期内保持永活，进程推出时自动删除，且不能被其它原因干扰。  
   第二种是普通用法，会话失效或者节点已存在都可能 会使临时节点消失。  
   对于第一种用途，在临时节点创建时，要检查节点存在与否以及归属。如果存在并且属于自己不必创建直接返回成功，若不属于则必须先删除后创建。这样做时为了防止程序程序快速重启这种情况下，临时节点创建失败问题。（依据clientid）  
14. 对node和children的订阅操作以及临时节点的创建操作，一旦成功就应该在map中保存Node(path, context), Children((path, context)，(tmpNode)。若会话超时后“恢复会话时”（即收到connected事件后），重新订阅并建临时节点。  
   这里的临时节点是指 第一种用途。
15. 持续订阅过程中，可能会出现操作失败问题。失败操作保存失败map中，定时再操作。当收到seesion_expired事件时清空失败map。  
16. context是void*类型，由上层应用负责释放(回调函数中)，若是碰到归属不明确的，应该用shared_ptr  
17. 响应时间:socket连接 < zk会话时间(临时节点) < 缓存(如果配置了)。  
   server1发现与server2之间的socket连接断开后，server1重新去zk上获取可用服务列表，依然有可能获取到server2.  
   所以在发现连接断开后，直接去取相应子节点，可能会取到原先的节点(节点挂了，但是会话未超时)  
18. zoo_get接口，数据为空时，buffer_len参数时而返回0，时而返回-1。官方文档上写返回-1。这个得统一处理，不能直接返回给上层。  
19. 由于会话在服务端维护，在设计分布式锁时(临时节点方式)存在一个问题，A获得锁，A网络异常，B获得锁，在A网络正常后A才能获知自己丢失锁。这就存在短暂的时间内多人同时得到锁的可能性。临时序列节点也同样存在这个问题。  
   由于以上问题存在，释放锁时同样存在问题，A可能会释放B的锁，因此在主动释放锁(主动删节点前)必须判断节点的归属(owner or id or ACL权限)。  
20. 分时式锁与主从选举的异同：  
    >实现方法相似：竞争创建单临时节点(监听当前节点或者父节点) 或者 最小序列临时节点(监听父节点) 或者其它方式。  
   不同：主从选举的目的主要是供其它应用(非竞争者)使用，该应用获取主的结果：无法获取到（可能网络故障）或者 获取成功。  
    
    分布式锁的目的是竞争者自己使用，在zk中竞争者存在自认为获得锁的可能性(自己获得锁，网络故障后锁被别人获得，但自己任然坚信自己拥有锁)，即多人获得锁而出现问题。  
21. ZOO_CONNECTED_STATE状态存在歧义，无法获知是否新建了会话。  
   状态变化介绍：
    >第一次启动时              未知 ->  connected   
   会话内网络短时间故障  connecting -> connected    
   会话超时到重新建立会话 connecting -> expired ->connected    
   CLOSED_STATE库未定义，但是库内代码使用0.    
22. zk为强一致性的，所以不要用来处理要求性能的场合。

---
## 三、**SDK需要实现的功能**
1. 会话失效后重建，以及订阅和临时节点的恢复。由sessionWatcher回调线程中完成。session_expired开启会话重建，session_connected事件重新订阅和临时节点恢复。  
2. 支持持续订阅。   
   > 一次订阅，持续收到变化通知。      
   两次会话之间（老会话超时，新会话未建立）数据变化通知不丢失。不可依赖原始的通知机制，故采用版本号(数据内容判断效率太低)作为数据变化推送的依据。保证即没有无效通知，也不会丢失通知。   
   当然两次会话之间多次数据变化，只能处理最后一次变化的通知。
3. 重建会话时和持续订阅中 的失败操作watch和 tmpnode 均保存在队列中，由单独线程定时再执行。
4. 服务注册功能：
   每个系统均有自己的根路径，例如level1，root=/hq/level1。服务(例如realtime_1)启动时，sdk自动在目录/hq/level1/realtime/realtime_1下创建临时节点(也可是有序临时节点)。临时节点由sdk维护，数据内容采用json格式(更直观)，具体命名规范由使用者定义，建议采用{"addr":{"rep":"localhost:8070","http":"http://localhost:8080"}}  
5. 服务发现功能：（目前实现3种）   
   **最优节点**：这是一种静态方式，不需要订阅，只在需要时(例如连接异常时)获取最优节点地址。例如，获取/hq/level1/realtime下某个性能较好的服务。   
   **地址池(连接池)**：这是一种动态方式，sdk在获取某一服务的所有子节点外，还必须订阅这种服务的变化（新增或者删除子节点），来实时更新本地的连接池（或者地址池）。例如订阅了/hq/level1/realtime，一旦有新的服务(例如realtime_n)启动时，sdk自动收到通知，并更新连接池或者地址池。 这种模式使用时一般采用轮训。  
   **主从节点**：当必须顾及到数据不一致的问题，那么所有服务都必须选择同一节点(即主节点)，当主节点有变更时所有服务同步切换。   
   **总结**：获取所有子节点用于连接池，获取最优节点用于直连单个节点, 主从模式中始终选择主节点。      
        最优节点是下游选择上游；主从节点是上游决定下游。      
6. 负载均衡(最优节点方式)：      
   最优节点的选择问题，涉及到负载均衡的策略问题。一般负载均衡策略包括负载收集，定时上报，最优节点选择。并且存在一些问题：负载实时性和额外负担问题(负相关)，负载不实时到导致的负载失衡问题。在实际生产环境中，绝对的均衡无意义，并且单一的均衡策略无法满足需求。  
   **负载收集**：读取/proc负载信息，按照权值计算负载值。并且提供负载收集第三方接口。   
   **定时上报**：考虑到zk的其它业务用途及zk性能问题（zk写操作QPS 5k左右，服务数量增大时无法通过新增zk节点来提升写性能），定时收集负载值，并且和阈值结合，阈值范围内的不上报。   
   **最优节点选择**：单个负载均衡策略不可用，绝对均衡无意义。考虑到负载上报的非实时性，在此采用最小负载加随机结合的方式。在负载最小的n个节点中，随机选择一个，避免因负载不实时导致的 负载失衡问题。    
   在服务刚启动的一段时间内，服务不参与上述负载均衡。    
   考虑到会话时间问题，所以优先选择非lastSon节点。lastSon就是上次选择出的最优节点，因为他的网络故障而触发这次选择。   
   **负载均衡(主从模式)**    
   适用于两种情况：程序数据一致性问题；程序计算耗用大量cpu，io及网络资源，而其它从节点可以共享主节点的数据结果。   
   选主节点采用有序临时节点，而不是单临时节点。原因单节点在节点删除，节点再次被创建，需要两步才能被其它人获取新主节点数据；而有序临时节点在发现节点变化的时候能立刻获取主节点数据，这个和分布式锁不一样因为它还要去获取节点数据。   
   主从模式存在一个问题：当主挂掉后，由于会话超时机制，此时系统中的主节点任然是已经挂掉的老节点。直到会话超时时才会更新主节点。也就是说，在这段时间内，无法提供服务。   
   解决方法是 记录这种临时节点，在程序退出时主动删除 该进程所记录的所有临时节点。   
   **负载均衡(地址池(连接池))**   
   > 拉模式：这种方式适用于client端（tcp意义上的）作为发起方，server端作为接收方。   
            客户端主动连接多个同业务类型的server，产生n条数据 连接句柄，连接池清晰可见。   
            如果server端无状态，可以采用轮训方式，最简单高效。   
            如果server端有状态，可以采用一致性hash。（本程序还未实现）。   
            轮训或者hash，均围绕池子中的句柄集合展开。   
   > 推模式：这种方式是server端作为发起方，client作为接收方。例如pub/sub。   
            普通连接：普通连接方式，server端均可以拿到所有的client连接的句柄，连接池清晰可见（类似于拉模式）。   
            pub/sub连接（nannomsg或者zeromq）：这种方式的也是一种连接池（池子比较隐晦），sub端按照topic分类。所以应该先定义好topic，轮训或者一致性hash，均围绕topic展开。   
          router/dealer连接（nannomsg或者zeromq）：这种方式也是连接池，router会将dealer与某个ID绑定，轮训或者hash，均围绕ID展开。   
   > 以上都是讲连接池的，地址池其实是存放短连接地址的池子，例如http地址，这种方式 与拉模式相同。有一种特殊情况，在短连接请求非常少的时候，不适合使用池子，因为池子的维护成本也很高。   
7. 实现同path重复订阅。   
   首先zk不支持重复订阅，所以应该在客户端记录多次订阅，并且在收到zk通知的时候分发订阅。   
   当同时发起两个订阅请求时，如果都向服务器请求数据，由于订阅要记录版本号(也就是说订阅请求是有状态的)，两次订阅的版本更新又是并发的，并且回调线程的并发性，所以版本的记录将是一个难以处理的问题，容易出现多推送或者少推送的问题。   
   解决方法是: 只让第一次订阅请求服务器，并且记录版本及缓存数据，往后请求不走服务器直接从缓存获取，这样避免了版本更新的问题。   
8. 获取节点数据可用于配置管理。   
9. 提供业务故障切换(zk本身提供的是网络故障的切换)。   
   主从模式：由于下游服务会根据主节点变化自动切换，所以上游节点只需要调用LocalAbandon::Hide()即可。   
   最优节点模式：该模式没有订阅功能，上游节点调用可以调用LocalAbandon::Hide()，但是下游节点必须主动调用GetBestSon(path,bestSon，lastSon)   

---
## 四、**未实现功能**
a) 未提供异步接口。
b) 未实现递归订阅功能。订阅父节点后，无法同时获取父节点数据，子节点个数以及子节点数据变化的通知。   
c) 未实现exist接口。get和exist行为不同，但是只能订阅一次，在sdk中实现path重复订阅时，难实现。   
c) 只实现地址池，未实现连接池。并且未实现hash分片功能。  

---
## 五、**注意点**   
1. 当某一个服务提供多个接口时，如{"http":"", "ws":"", "rep":""}。如果多个接口提供不同业务服务，无问题。   
   当多个接口提供一种业务服务时，就必须考虑一致性问题。当使用者要与该服务建立多个连接时，务必要让这些连接练到一个服务上。当其中一条连接出现问题时，应该默认所有连接均出问题，一同切换到另一台服务器上。   
2. 订阅父节点个数变化，但子节点数据变化，无法感知。服务迁移时(临时节点数据有变化，一般是ip变化)，若此时会话异常，可能无法感知。正确做法是，通知子节点有变化，但是获取的子节点后发现无任何增删，此时应该重新加载所有子节点。   
3. watch回调不应该阻塞太久。   


-----------
## 六、**接口设计**

###  6.1 **基本接口**  
```
 /***********在启动之前设置好参数*****************************/
 SdsResult SetRoot(const std::string& path);                                           //设置根路径，默认是无，跨系统调用时最好不要设置该接口，例如 /hq/level2 调用／hq/level1的服务
 void SetCollectInterval(int itvl);                                                               //上报负载的时间间隔(default 3min)
 void SetLoadWay(std::shared_ptr<Load> load);                                    // 第三方负载收集接口
 /***********网络连接及自动创建临时节点*******************************/
 SdsResult Start(const std::string& servicePath, Json::Value serviceData, const std::string& zkHosts, int zkTimeout);
 SdsResult Stop()
 /***********业务接口**********************************************/
 SdsResult Create(const std::string& path, const std::string& data, bool isTmp = false, bool tmpAlive = false);//非序列临时永活节点(tmpAlive=true)特点：若节点存在则删除后创建
 SdsResult Create(const std::string& path, const std::string& data, bool isTmp, bool tmpAlive, bool isSeq, std::string& seqPath);
 SdsResult CreateRecursion(const std::string& path);//递归创建节点，数据无
 SdsResult Set(const std::string& path, const std::string& data, int32_t version = -1);
 
 //获取节点数据，可支持持续订阅，SdsWatcher只需实现OnDeleted和OnChanged接口(注意不同的请求不可共用SdsWatcher,因为SdsWatcher记录了状态)
 SdsResult GetNode(const std::string& path, std::string& data, int32_t& version);
 SdsResult GetNode(const std::string& path, std::string& data);
 SdsResult GetNode(const std::string& path, std::string& data, std::shared_ptr<SdsWatcher> watcher);
 
 //获取孩子节点变化，可持续订阅，SdsWatcher只需实现OnDeleted和OnChildren接口(注意不同的请求不可共用SdsWatcher)
 SdsResult GetSons(const std::string& path, std::vector<std::string>& sons); //获取孩子，不订阅
 SdsResult GetSons(const std::string& path, std::vector<std::pair<std::string, std::string> >& sons); //获取孩子即孩子数据，不订阅
 SdsResult GetSons(const std::string& path, std::vector<std::string>& sons, std::shared_ptr<SdsWatcher> watcher); //获取孩子，持续订阅，watcher的归属不明所以用智能指针

 //获取孩子节点中，相对性能较好的一个孩子(简单负载均衡)。
 //lastSon: 考虑到会话时间过长(son程序已停止而会话未超时), 有可能再次选取已经停止的son。故该参数用来优先选择非lastSon的节点。
 SdsResult GetBestSon(const std::string& path, std::pair<std::string, Json::Value>& bestSon, const std::string lastSon ="");
 
 //批量获取子节点的数据，GetSons的回调中可能须要用到
 SdsResult GetSonsData(const std::string& fatherPath, std::vector<std::string>& sons, std::vector<std::pair<std::string, std::string> >& sonsData);
 
 //选择主节点(主节点是名字排序最小的节点，可使用有序节点)
 SdsResult GetMaster(const std::string& fatherPath, std::string& masterSon, std::shared_ptr<SdsWatcher> watcher);
 SdsResult GetMaster(const std::string& fatherPath, std::pair<std::string, std::string>& masterSon);
 
 SdsResult ReFreshServicePath();   //强制刷新业务节点，删除+创建.序列节点重新排序。可能失败。
 SdsResult HideServicePath();      //隐藏 本程序, 保证成功。
 SdsResult RecoverServicePath();   //恢复 本程序，保证成功。
```

### 6.2 **SdsWatcher介绍**
```
class SdsWatcher {
public:
  virtual ~SdsWatcher() {}
  virtual void OnDeleted(const std::string& path) = 0;
  virtual void OnChanged(const std::string& path, const std::string& data) {
 
  }
  virtual void OnSonChanged(const std::string& path, std::vector<std::string>& allSons, std::vector<std::string>& addSons, std::vector<std::string>& delSons) {
   /*
    * 1. 一旦addSons.empty() && delSons.empty() ==true，应该重新加载所有allSons
    * 2. addSons得到的只是节点名称，获取数据可能存在不一致问题。一旦获取出错，存储标记，下次加载全量数据。
    */
  }
};
```

### 6.3 **扩展接口**   
```
class AddrBest{ 	//最优节点，封装好的
     AddrBest(std::string askPath, std::string elem);
     SdsResult GetAddr(std::string& ip, int& port);
     SdsResult GetAddr(std::string& addr);
}
class AddrMatser{ //主节点，订阅功能封装好的 
     AddrMaster(std::string askPath, std::string elem, bool needWatch = false);
     SdsResult GetAddr(std::string& ip, int& port);
     SdsResult GetAddr(std::string& addr);
     virtual void OnChangedMaster(); // master变化了通知
}
class AddrPool{ // 地址池，订阅及轮训功能已经封装好
     AddrPool(std::string askPath, std::string elem);
     bool Init();
     virtual std::string GetAddr(); //轮训获取地址池中的数据
}
class LocalAbandon{   //根据业务问题 主动隐藏服务
     void Hide();     
     void Recover();
}
```
---
## 七、 **依赖库**
jsoncpp: https://github.com/open-source-parsers/jsoncpp
zookeeper C API: 官方3.4.10库版本。
LOG 日志库自己找一个。这里会报错，将日志库的头文件去掉就行，然后用constant.h 中有个宏LOGGER打开就行。

## 八、 **记得打赏**
各位码农哥，如果好用的话，记得打赏下，失业在家不容易。   
微信支付   
<image src="https://github.com/pingzilao/zksdk/blob/master/images/webchatpay.jpg" width="290" />  
支付宝   
<image src="https://github.com/pingzilao/zksdk/blob/master/images/alipay.png" width="290"/>


