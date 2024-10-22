# Learning-SimpleFrameWork
学习唐老狮基础框架课程时的记录与思考        
ReadMe没有目录不便查阅，丢两个有目录的地址      
[Wiki](https://github.com/GavinZ233/Learning-SimpleFrameWork/wiki)
[知乎](https://zhuanlan.zhihu.com/p/653773390)
# 唐老狮的基础框架
|框架模块|具体内容|作用|
|:---:|---|---|
|单例模式基类|BaseManager、SingletonAutoMono、SingletonMono|避免重复的声明单例
|缓存池|PoolMgr、PoolData|回收重复创建的GameObject
|事件中心|EventCenter、EventInfo、IEventInfo|作为监听者分发事件
|公共Mono|MonoMgr、MonoController|向外提供MonoBehavior的生命周期
|场景切换|ScenesMgr|同步或异步加载场景，并在异步时向外更新进度
|资源加载|ResMgr|同步或者异步加载对象，并自动实例化GameObject
|输入控制|InputMgr|监听设备输入情况并广播
|音效管理|MusicMgr|背景音乐与音效的创建和修改
|UI管理|BasePanel、UIManager|UI面板基类和UI面板的管理模块，自动获取面板控件

## 单例模式基类

### 1. 单例基类
    public class BaseManager<T> where T:new()
    {//懒汉模式，当访问时，才实例化
        private static T instance;

        public static T GetInstance()
        {   
            if (instance == null)
                instance = new T();
            return instance;
        }
    }


###  单例基类的衍生知识：
#### 1. 饿汉模式与懒汉模式
懒汉模式在访问时才实例化，避免了不常用的单例占用内存    
饿汉模式在静态变量被记入内存时，就一起实例化了，坏处是占用内存，好处是提前实例化避免了线程不安全    


### 2. 继承Mono的单例基类

#### 2.1 懒汉模式
运行最初，内存中只有该类的instance声明，类并没有实例化，当调用GetInstance（）时，会执行一系列的游戏物体创建与单例类的挂载等操作，最终返回单例类。

    public class SingletonAutoMono<T> : MonoBehaviour where T : MonoBehaviour
    {
        private static T instance;

        public static T GetInstance()
        {//当访问单例，单例为空时，创建GameObjcet并附加脚本
            if( instance == null )
            {
                GameObject obj = new GameObject();
            //设置对象的名字为脚本名
                obj.name = typeof(T).ToString();
            //让这个单例模式对象 过场景 不移除
                DontDestroyOnLoad(obj);
                instance = obj.AddComponent<T>();
            }
            return instance;
        }

    }


#### 2.2 饿汉模式
场景加载时，挂载该脚本的游戏物体被创建，脚本也会实例化，即场景加载时，该单例类就已经存在

    public class SingletonMono<T> : MonoBehaviour where T: MonoBehaviour
    {
        private static T instance;

        public static T GetInstance()
        {
         return instance;
        }

     protected virtual void Awake()
      {//在GameObject创建时，给instance赋值
          instance = this as T;//父类转子类
       }
	
    }

### Mono单例基类的衍生知识：
#### 1. 过场景不删除

DontDestroyOnLoad（obj）是将该GameObject标记为切换场景不删除，在运行项目时，可以从Hierarchy面板中的DontDestroyOnLoad场景中找到不会删除的obj。

#### 2. Mono脚本的实例创建时机

MonoBehaviour挂载的GameObject创建时，脚本会跟随创建。   
在AddComponent时，也会创建脚本类，达到实例化的目的。    
以上两种情况都会触发Awake方法


## 缓存池模块

### 1. 缓存池类


|PoolMgr|对象池管理类|PoolData|对象池|
|---|---|---|---|
|+ < string,PoolData> poolDic| 记录对象池的字典  |+ List< GameObject> poolList|具体对象的缓存池|
|- GameObject poolObj| 对象池的根物体 |+ GameObject fatherObj|对象池物体|
|+ GetObj(string name,UnityAction< GameObject> callBack)| 根据名称访问字典找寻对应对象池，无对象池就创建一个该物体给委托，有对象池就进入对象池寻找目标   |+ PoolData(GameObject obj,GameObject poolObj)|构造函数（传入的回收对象，对象池根物体）|
|+ PushObj(string name,GameObject obj)|压入目标物体，寻找对应对象池，无对象池就调用构造函数，构造函数会自动压入第一个物体 |+ PushObj(GameObject obj)|压入目标物体，记录物体并失活|
|+ Clear()|清空字典与 poolObj，在切换场景时，停止对目标的引用，让GC自动回收 |+ GameObject GetObj()|弹出物体，移出list返回给PoolMgr|

#### 1.1 缓存池调用流程

##### 1.1.1 取对象
1. 调用PoolMgr.GetObj(子弹名称，拿到子弹后执行的方法())
2. GetObj()方法内，判断是否有对应的对象池（1:有对象池，取对象池内的物体  2:无对象池说明没有该物体，创建物体  3：有对象池但为空，创建物体），激活物体并传入委托
3. 访问到PoolData时，调用到GetObj（）从list中移出，并返回到PoolMgr
##### 1.1.2 放对象
1. 调用PoolMgr.PushObj(对象名称，物体对象)
2. PushObj()方法内，判断是否有对象池（1:有对象池，调用对象池的PushObj，压入物体并失活 2:无对象池时，实例化一个对象池传入要缓存的物体，并记录对象池到字典）
3. 对象池失活物体并记入list,将物体的父物体改为自己，方便浏览

### 2. 对象池的思考
#### 2.1 对象池对应的管理类
这种对象池只负责收纳，对于不使用的物体无法自动回收，需要有个外部的类管理取出的物体进行缓存池的装入操作。    
由此可以面对不同模块，构建不同的对象管理类，进行回收操作。  
缓存池和对象管理类就像：垃圾站和环卫工，面对特效和子弹，都有不同的环卫工去定期收集并放入垃圾站。

#### 2.2 适当定制化
对象池也可以模块定制化,声明特效模块的对象池和子弹模块的对象池,只负责自己模块的回收与缓存.  
这种方式一看就会声明很多类,不利于管理,但是在面对未知时,对象池面向模块高度定制化往往比设计通用的对象池省时和灵活(都是甲方逼得)


## 事件中心

### 1. 整体结构

|类名|类的职责|
|---|---|
|EventCenter|负责记录外部传入的事件，并在触发时分发事件|
|EventInfo : IEventInfo|记录事件的类，为无参事件做一层封装|
|EventInfo< T > : IEventInfo|为有参事件做一层封装|
|IEventInfo|为两个记录事件的类提供父类，满足里氏替换原则进行转换|

#### 1.1 EventCenter

|名称|作用|操作|
|---|---|---|
|- Dictionary<string, IEventInfo> eventDic|记录事件类的字典|无
|+ AddEventListener< T >(string name, UnityAction< T > action)|添加有参事件|检查字典是否有name的事件类，无就构造一个事件类加入字典，事件类记录目标事件
|+ AddEventListener(string name, UnityAction action)|添加无参事件的重载|同上
|+ RemoveEventListener< T >(string name, UnityAction< T > action)|移除有参事件|字典查询目标得到事件类，移除改事件的监听
|+ RemoveEventListener(string name, UnityAction action)|移除无参事件的重载|同上
|+ EventTrigger<T>(string name, T info)|有参事件的触发|检查字典存在目标事件类时，不为空就执行事件
|+ EventTrigger(string name)|无参事件的触发|同上
|+ Clear()|清空记录事件的字典|清空对事件类的引用，切换场景时让GC自动回收上个场景的事件类

#### 1.2 EventInfo : IEventInfo
|名称|作用|操作|
|---|---|---|
|+ UnityAction actions|事件|无
|+ EventInfo(UnityAction action)|构造函数|记录初次传入的事件

#### 1.3 EventInfo< T > : IEventInfo
|名称|作用|操作|
|---|---|---|
|+ UnityAction actions|事件|无
|+ EventInfo(UnityAction action)|构造函数|记录初次传入的事件

#### 1.4 IEventInfo

空接口

### 2.思考

#### 2.1 监听者模式
基本思路就是实现一个事件中心，代替其他类的互相监听。好处是统一了监听对象，降低了其他模块的耦合，坏处就是所有的类都要调用事件中心，不同的事件需要声明不同的事件，模块多了之后不利于管理。    
关于事件过多，可以声明事件类，用来包装事件，再用字典统一管理，结构方面的混乱就迎刃而解了。

#### 2.2 里氏替换
里氏替换原则的经典应用，用父类装子类，好处是可以将多个不同的类使用统一的父类包装，并且避免了传统object包装后的拆装箱问题。  
此处还有一个泛型问题被一并解决：如果在字典中直接申明泛型的事件类，事件中心会需要变成泛型类，出现泛型方面的问题。泛型事件类根据每次不同的实例化确定泛型T的类型，而事件中心作为单例只会实例化一次，彼此冲突。当泛型事件类装入父类，再记录给事件中心时，事件中心就不需要变成泛型类，因为它的成员没有泛型了。


#### 2.3 结合项目思考
里氏替换在自己写过的对象池有应用过，表格模块的表类有多种，对象池为了每一种表格声明一个对象池类不现实。对象池的list选择记录表格的父类，返回到对象池管理器时，再转成对应的子类。  
曾经写过的监听者模式，是一个事件中心根据需要声明不同的Action，没有对事件进行管理


## 公共Mono模块

### 1. 类
|MonoMgr|Mono管理类|MonoController|Mono方法提供类
|---|---|---|---|
|+ MonoMgr()| 构造方法，创建一个GameObject并挂载MOnoController    |- event UnityAction updateEvent|更新Update的事件
|+ AddUpdateListener(UnityAction fun)|事件传递给MonoController    |- Start ()|  标记自身过场景不删除
|+ RemoveUpdateListener(UnityAction fun)|移除MonoController中的事件|- Update ()|  自身每帧调用updateEvent事件，通知监听模块执行方法|
|+ Coroutine StartCoroutine(IEnumerator routine)|调用MonoController执行协程|+ AddUpdateListener( UnityAction fun)| 添加帧更新事件的函数|
|+ Coroutine StartCoroutine(string methodName, [DefaultValue("null")] object value)| 同上，启动有参协程 |+ RemoveUpdateListener( UnityAction fun)|移除帧跟新事件的函数|
|+ Coroutine StartCoroutine(string methodName)|按名称执行协程，但只能执行MonoController内部的协程|||

### 2. 功能
 向外提供MonoBehavior脚本的方法，比如生命周期周期函数，或者开启协程。   
 由此可以节省不必要的脚本声明，在热更新项目中也会担任很重要的角色，热更的逻辑可能无法写在Mono脚本中，需要根据Mono生命周期执行时，可以通过将事件交给MonoMgr来执行。

### 3. MonoBehavior执行生命周期

#### 3.1 生命周期执行层顺序  


> `Initialization` => Editor => Initialiazation => `Physics` => `Input events` => `Game logic` => 
Scene rendering => Gizmo rendering => GUI rendering => End of frame => Pausing => Decommissioning   


#### 3.2 常用生命周期方法(按时间顺序排列)   

|方法|执行生命周期层|触发时机|常用场景|
|---|:---:|---|---|
|Awake|Initialization层|加载场景时初始化GameObject时、先前非活动的GameObject设置活动时、初始化使用Object.Instantiate创建之后、GameObject被Add脚本后|执行初始化操作
|OnEnable|Initialization层|该函数在对象变为启用和激活状态时调用|被执行Active时想要执行的操作，比如被回收的UI重新激活时进行Group刷新
|Start|Initialization层|在将要执行第一帧前|初始化后的操作，需要访问其他重要类的初始化操作
|FixedUpdate|Physics层|固定帧率执行，默认为0.02s/次|用于物理计算
|OnTrigger`Enter`|Physics层|挂载的触发器被触发时（Enter、Exit、Stay）|用作简单的区域判断，比如进入火海的怪物，调用collider的怪物脚本持续扣血
|OnCollision`Enter`|Physics层|挂载的碰撞体被触发时（Enter、Exit、Stay）|子弹等需要考虑精确信息的情况，collision类包含接触点和速度
|yield WaitForFixedUpdate|Physics层|物理帧的最后执行|协程想要实现物理运算时
|OnMouse`Enter`|InputEvents层|当鼠标对碰撞体操作时（Enter、Exit、Over、Up、Drag、Down、UpAsButton）|模型的选中，拖拽等
|Update|GameLogic层|每一帧执行|键盘监听，每帧更新的业务逻辑
|yield null or something|GameLogic层|协程没有返回特殊类时|多是用来暂停协程的逻辑，留给下一帧继续，降低系统压力
|yield WaitForSeconds|GameLogic层|指定秒数后执行|计时器，延迟协程逻辑
|StartCoroutine|GameLogic层||开启协程方法
|LateUpdate|GameLogic层|在一帧的最后执行|需要跟随计算的情况，如跟随相机，跟随的挂件
|OnDrawGizmos|GizmoRendering层|每帧执行|测试阶段画辅助线
|OnGUI|GUIRendering层|每帧执行|绘制GUI，也属于测试阶段
|yield WaitForEndOfFrame|EndOfFrame层|协程在一帧的最后执行|协程需要计算一些跟随的逻辑，我还没遇到 ┑(￣Д ￣)┍
|OnDisable|Decommissioning层|当挂载物体失活时|对于需要回收物体的回收操作和关闭方法
|OnDestroy|Decommissioning层|被删除前的逻辑|如怪物死亡后清除尸体时，执行血液武器等的删除

>自上表格可以看出，协程的生命周期方法是要迟于本体方法的。
#### 3.2 偶尔使用的生命周期方法
|方法|触发时机|常用场景|
|---|---|---|
|OnAnimatorMove|在物理层，每帧调用|用于处理动画移动以修改根运动的回调
|OnAnimatorIK|在物理层，每帧调用|用于设置动画 IK的回调
|OnApplicationFocus|在帧的最后检查焦点是否在程序内|程序焦点改变时暂停游戏或者改变UI等
|OnApplicationPause|在帧的最后检查程序是否暂停|程序暂停时通知游戏逻辑暂停


## 场景切换模块    

### 1. ScenesMgr类
|方法|操作|作用|
|---|---|---|
|+ LoadScene(string name, UnityAction fun)|加载场景，执行委托|同步加载场景
|+ LoadSceneAsyn(string name, UnityAction fun)|开启异步加载场景的协程|向外提供异步加载场景方法
|- ReallyLoadSceneAsyn(string name, UnityAction fun)|循环判断是否加载完毕，向事件中心触发进度条进度事件，结束执行委托|异步加载场景并向外传加载进度

### 2. 拓展知识

#### 2.1 AsyncOperation类
异步操作协同程序    

|变量|作用|使用示例|
|---|---|---|
|allowSceneActivation|允许在场景准备就绪后立即激活场景|设置为false，可以等待额外的初始化内容完成后再激活
|isDone|操作是否已完成（只读）|中断加载场景的循环条件
|priority|Priority 允许您调整执行异步操作调用的顺序|当有多个异步操作排队时，将首先执行具有 更高优先级的操作
|progress|获取操作进度（只读）|用来更新进度条，需要注意进度最多到0.9
|completed|操作完成时调用的事件|即使操作能够同步完成，也将在下一帧调用在创建它的调用所在的帧中注册的事件处理程序。如果处理程序是在操作完成后注册的，并且已调用 complete 事件，则将同步调用该处理程序


## 资源加载模块

### 1. ResMgr类
|方法|操作|作用|
|---|---|---|
|+ T Load< T >(string name) where T:Object|加载指定资源，如果是GameObject就实例化之后再返回|同步加载资源，并实例化
|+ LoadAsync< T >(string name, UnityAction< T > callback) where T:Object|开启异步协程|对外开放异步加载方法
|- ReallyLoadAsync< T >(string name, UnityAction< T > callback) where T : Object|异步加载指定资源，是GameObject也是实例化后再返回|异步加载资源

### 2. 拓展
#### 2.1 ResourceRequest
Resources包的异步加载请求，继承自**AsyncOperation**     

|变量|作用|使用示例|
|---|---|---|
|asset|正在加载的资源对象（只读）|加载完成后，直接调用实例化asset

### 3. 联想

#### 3.1 协程换成线程
目前大部分异步操作都是使用协程，并不是协程真的万能，而是Unity对于线程支持度较低。后续尝试多了解线程池等相关知识，考虑使用线程替代部分协程的工作。线程无法替代主线程的Unity操作，但是可以帮助处理网络或者寻路等复杂计算。
#### 3.2 协程与线程差异
协程的逻辑执行只是被yield分帧在主线程里执行，好处是不用担心线程安全问题，坏处就是压力依然集中在主线程。
线程是独立于主线程的，在允许操作Unity资源的前提下，加载和操作资源是不会影响到主线程的，减少了主线程的压力，但也带来了线程安全问题。

#### 3.3 线程安全问题
简言之，多个线程访问同一个目标，出现的冲突就是线程安全问题。    
如：    
同时访问主角的100血量属性，一个要加10一个要扣10，结果一定不是100，只会是90或者110。     
两个线程访问同一个没有实例化的单例类，此时就会实例化两个该类，因为都触发了类的实例化，哦吼，单例变双例。（bytheway：饿汉模式因为提前实例化，恰好避免了该问题）

  


## 输入控制模块

### 1. InputMgr类
|方法|操作|作用|
|---|---|---|
|+ InputMgr()|将自己的Update传入MonoMgr|实例化时，为自己获得Update的生命周期
|+ StartOrEndCheck(bool isOpen)|检测输入状态的变量修改为入参|提供关闭开启输入检测的方法
|- CheckKeyCode(KeyCode key)|将按键输入传递给事件中心|将输入事件广播
|- MyUpdate()|状态变量为true时，调用CheckKeyCode（）方法，并传入需要广播的KeyCode|记录需要广播的按键

### 2. 拓展
新旧版输入系统  
旧版输入控制是声明在哪就在哪修改。  
新版集中化处理，在配置文件中声明并且兼容了多平台的输入，封装了自己的事件中心，并且事件内容也更加详细。

鉴于游戏公司多数使用旧版Unity，使用的都是旧版输入系统，暂不深入了解InputSystem

## 音效管理模块

### 1. MusicMgr类
仅适用于没有距离要求的音效场景


|成员|操作|作用|
|---|---|---|
|- AudioSource bkMusic||背景音乐播放器
|- float bkValue||背景音乐大小
|- GameObject soundObj||音效播放器挂载物体
|- List< AudioSource > soundList||音效列表
|- float soundValue||音效大小
|+ MusicMgr()|将自己的Update传给MonoMgr|实例化时，时获得Update的触发时机
|- Update()|遍历soundList，删除已经停止播放的AudioSource|清除无用的播放器
|+ PlayBkMusic(string name)|如果背景音乐播放器为空，创建一个物体并挂载播放器，记录播放器。异步加载背景音乐并初始化。|播放背景音乐
|+ PauseBKMusic()|如果背景音乐不为空，暂停播放器|暂停背景音乐
|+ StopBKMusic()|如果背景音乐不为空，停止播放器|关闭背景音乐
|+ ChangeBKValue(float v)|修改背景音乐音量值，当播放器不为空，向播放器赋值|修改背景音乐音量大小
|+ PlaySound(string name, bool isLoop, UnityAction< AudioSource > callBack = null)|当音效挂载物体为空时，创建并记录，异步加载音效并加入新的播放器，播放器挂载到物体上并初始化|播放音效
|+ ChangeSoundValue( float value )|遍历当前音效数组，赋值目标音量值|修改音效音量
|+ StopSound(AudioSource source)|检查列表是否有该播放器，移出列表并停止播放删除播放器|停止音效


## UI模块

### 1. BasePanel类
|成员|操作|作用|
|---|---|---|
|- Dictionary<string, List< UIBehaviour >> controlDic||通过里氏转换来记录所有控件的字典
|- Awake ()|寻找所有的子控件|保护虚函数，提供给子类获取控件的初始化
|+ ShowMe()|虚函数等子类写逻辑|显示自己
|+ HideMe()|虚函数等子类写逻辑|隐藏自己
|- OnClick(string btnName)|虚函数等子类写逻辑|按钮点击时的委托，子类可以根据传入的btnName分辨具体按钮
|- OnValueChanged(string toggleName, bool value)|虚函数等子类写逻辑|Toggle被点击时的委托，子类根据入参执行逻辑
|- T GetControl< T >(string controlName) where T : UIBehaviour|当字典有该名称的对象时，遍历对象的控件List得到该控件并返回|得到对应名称物体的指定类型控件
|- FindChildrenControl< T >() where T:UIBehaviour|获取对象下所有子控件T，遍历控件计入字典，当T为Button或者Toggle等有交互的控件时，传入委托|找到子对象的对应控件加入字典

#### 1.1 知识点

##### 1.1.1 里氏替换
老生常谈的经典原则，父类装子类，常常伴随泛型一起使用。此处是为了满足同时接受所有的UI控件类，使用共同的父类UIBehaviour接收，使用时再as成指定的类。

##### 1.1.2 活用FindChildren避免手动拖拽
平时的UI控件都是Public或者SerializeField序列化到Inspector面板上手动拖拽，此处唐老师采用了全部Find，按照对象名称分别记录到字典。     
访问时通过字典里的对象名与对应控件类可以找到，声明特殊事件委托的情况也可以通过提前编写带名称的委托封装方法实现。（ps：当前只写了Btn和Tog两种，其他Slider等用到的时候再加）


### 2. UIManager类
|成员|操作|作用|
|---|---|---|
|+ Dictionary<string, BasePanel> panelDic||记录Panel的字典
|- Transform bot||UI层级中的底层
|- Transform mid||UI层级中的中层
|- Transform top||UI层级中的顶层
|- Transform system||UI层级中的系统层
|+ RectTransform canvas||记录UI的Canvas方便外部访问
|+ UIManager()|创建Canvas并记录四个层级的Transform，保留Canvas于EventSystem过场景不移除|构造函数，初始化
|+ Transform GetLayerFather(E_UI_Layer layer)|外部访问层级时，返回对应Transfrom|向外提供Canvas的层级对象
|+ ShowPanel< T >(string panelName, E_UI_Layer layer = E_UI_Layer.Mid, UnityAction< T > callBack = null) where T:BasePanel|访问字典有Panel时，调用面板的ShowMe（）并执行委托结束函数，无面板时异步加载面板资源，并在加载结束后根据入参设定层级父节点，初始化面板的Transfrom，显示面板记入字典并执行委托|显示面板并设定层级
|+ HidePanel(string panelName)|有面板时，调用面板HideMe（）并删除面板，移出字典|删除面板
|+ T GetPanel< T >(string name) where T:BasePanel|检查字典是否有该key返回T，无返回空|外部获取目标面板
|+ AddCustomEventListener(UIBehaviour control, EventTriggerType type, UnityAction< BaseEventData > callBack)|获取控件的EventTrigger，没有就添加，事件类型设置为入参的type，事件的calback传入入参的委托|给控件添加事件监听的静态方法

#### 2.1 知识点

##### 2.1.1 Hierarchy的渲染顺序
算是常识，UI正常情况下是按照Hierarchy的上下顺序来渲染的（World模式的UI是按照距离渲染）。如：A、B图片，A在上B在下，因为渲染顺序是自上向下，先渲染A再渲染B，所以在屏幕上就像先画了A再画了B，B就会掩盖在A上。      
此处唐老师借此衍生出了UI的四个层级对象，底层，中层，顶层，系统层。      
使用场景：打开背包界面，一个黑色的背景属于底层，背包栏和装备栏属于中层，被拖拽的装备临时属于顶层这样就不会被背包或者装备栏遮挡，而装备拖拽失败时弹出的警告必然属于系统层，渲染在所有层级之上。

##### 2.1.2 EventTrigger


Unity文档原话：    
>接收来自 EventSystem 的事件，并为每个事件调用已注册的函数。    
EventTrigger 可用来指定您想为每个 EventSystem 事件调用的函数。 您可以将多个函数分配给单个事件，每当 EventTrigger 收到该事件时，都会按照这些函数的提供顺序调用它们。 
注意：将此组件附加到游戏对象将导致该对象拦截所有事件，并且所有事件都不会传播到父对象。  
您可以通过两种方式拦截事件：    
一是扩展 EventTrigger，并覆盖您想拦截的事件的函数   
二是指定单个委托

经测试，加入EventTrigger的Click并不会影响原本Btn的Click触发。

此处唐老师使用EventTrigger是为了额外拓展一些触发，比如图片也需要监听鼠标点击，与此类似的操作是在图片上挂载一个脚本并继承IPointerClickHandler接口。相较之下加入EventTrigger确实会更方便。（ps：以前真的是直接挂载脚本专门负责监听，真的太浪费了）

既然提到接口，就继续说说，IPointerClickHandler类似的接口，与EventTrigger都属于UnityEngine.EventSystems命名空间，EventTrigger就是Unity为我们提供的集事件大成者，与我刚才提到的挂在脚本做监听类似，不过是封装好的。

挖个坑`EventSystems`也可以写一篇笔记，结构参照Unity文档的结构
