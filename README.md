# Learning-SimpleFrameWork
学习唐老狮基础框架课程时的记录与思考

# 唐老狮的基础框架
|框架模块|具体内容|作用|
|:---:|---|---|
|单例模式基类|BaseManager、SingletonAutoMono、SingletonMono|避免重复的声明单例
|缓存池|缓存池Mgr和单对象池类|回收重复创建的GameObject
|事件中心|事件中心类，事件封装类和事件接口|作为监听者分发事件
|公共Mono|12|
|场景切换||
|资源加载||
|输入控制||
|音效管理||
|UI控制||
|数据管理||

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
|Dictionary<string, IEventInfo> eventDic|记录事件类的字典|无
|AddEventListener< T >(string name, UnityAction< T > action)|添加有参事件|检查字典是否有name的事件类，无就构造一个事件类加入字典，事件类记录目标事件
|AddEventListener(string name, UnityAction action)|添加无参事件的重载|同上
|RemoveEventListener< T >(string name, UnityAction< T > action)|移除有参事件|字典查询目标得到事件类，移除改事件的监听
|RemoveEventListener(string name, UnityAction action)|移除无参事件的重载|同上
|EventTrigger<T>(string name, T info)|有参事件的触发|检查字典存在目标事件类时，不为空就执行事件
|EventTrigger(string name)|无参事件的触发|同上
|Clear()|清空记录事件的字典|清空对事件类的引用，切换场景时让GC自动回收上个场景的事件类

#### 1.2 EventInfo : IEventInfo
|名称|作用|操作|
|---|---|---|
|UnityAction actions|事件|无
|EventInfo(UnityAction action)|构造函数|记录初次传入的事件

#### 1.3 EventInfo< T > : IEventInfo
|名称|作用|操作|
|---|---|---|
|UnityAction actions|事件|无
|EventInfo(UnityAction action)|构造函数|记录初次传入的事件

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

 


