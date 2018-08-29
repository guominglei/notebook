# java 锁机制
对象锁
类锁
synchronized 
## 原理jvm角度 
jvm 需要对两类线程共享的数据进行协调  

		a.保存在堆中的实例变量  
		b.保存在方法区中的类变量  

这两类数据对所有线程共享。只需要添加synchronized块或者synchronized方法即可实现锁机制。

## 实现方法

### 1、代码块同步
	
		public void run(){
			synchronized(lock_item){
				for(int i=0 ;i<10 ; i++){
					System.out.println("No"+num+":"+i);
				}
			}
		}

代码块同步方式，依赖于lock_item是否是唯一的

### 2、类实例同步方法synchronized 修饰的方法
	public synchronized void abc(int num){
		for(int i=1; i< 0; i++){
				System.out.println("No"+num+":"+i);
			}
		
		}

这种情况下，只有单例模式下才能实现同步。多实例之间不能实现同步。

### 3、类同步静态方法。形式如：  
	
	public static synchronized void abc(int num){
		for(int i=1; i< 0; i++){
				System.out.println("No"+num+":"+i);
			}
		
		}

对于同步静态方法，对象锁就是该静态方法所在的类的Class实例。所以这个类的多个实例对象，可以通过同步静态方法实现锁同步。因为实例对象使用的锁对象都是一个就是这个类的 类实例。