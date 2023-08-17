# Learning-SimpleFrameWork
学习唐老狮基础框架课程时的记录与思考

# 唐老狮的基础框架
|框架模块|具体内容|作用|
|:---:|---|---|
|单例模式基类|BaseManager、SingletonAutoMono、SingletonMono|避免重复的声明单例
|缓存池|缓存池Mgr和单对象池类|回收重复创建的GameObject
|事件中心||
|公共Mono||
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


