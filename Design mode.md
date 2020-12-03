

工厂设计模式

1. 简单工厂模式
2. 工厂方法模式
3. 抽象工厂模式



  1）还没有工厂时代：假如还没有工业革命，如果一个客户要一款宝马车,一般的做法是客户去创建一款宝马车，然后拿来用。
  2）简单工厂模式：后来出现工业革命。用户不用去创建宝马车。因为客户有一个工厂来帮他创建宝马.想要什么车，这个工厂就可以建。比如想要320i系列车。工厂就创建这个系列的车。即工厂可以创建产品。
  3）工厂方法模式时代：为了满足客户，宝马车系列越来越多，如320i，523i,30li等系列一个工厂无法创建所有的宝马系列。于是由单独分出来多个具体的工厂。每个具体工厂创建一种系列。即具体工厂类只能创建一个具体产品。但是宝马工厂还是个抽象。你需要指定某个具体的工厂才能生产车出来。

  4）抽象工厂模式时代：随着客户的要求越来越高，宝马车必须配置空调。于是这个工厂开始生产宝马车和需要的空调。



## 简单工厂模式（静态工厂模式）

从命名上就可以看出这个模式一定很简单。它存在的目的很简单：定义一个用于创建对象的接口。 
   先来看看它的组成： 
     1) 工厂类角色：这是本模式的核心，含有一定的商业逻辑和判断逻辑，用来创建产品
     2) 抽象产品角色：它一般是具体产品继承的父类或者实现的接口。     
     3) 具体产品角色：工厂类所创建的对象就是此角色的实例。在java中由一个具体类实现。 

![屏幕快照 2020-09-07 下午12.56.29](/Users/meiliwu/Desktop/屏幕快照 2020-09-07 下午12.56.29.png)

```
//抽象产品类
public class BMW {
    public BMW() {
        System.out.println("宝马");
    }
}


//产品1
public class BMW320 extends BMW {
    public BMW320() {
        System.out.println("BMW 320");
    }
}

//产品2
public class BMW520 extends BMW {
    public BMW520() {
        System.out.println("BMW 520");
    }
}

//工厂类
public class Factory {
    public static BMW getCar(String brandName) {
        switch (brandName) {
            case "320":
                return new BMW320();
            case "520":
                return new BMW520();
            default:
                return null;
        }
    }
}

//客户类
public class Customer {
	public static void main(String[] args) {
		Factory factory = new Factory();
		BMW bmw320 = factory.createBMW(320);
		BMW bmw523 = factory.createBMW(523);
	}

```

## 工厂方法模式

 工厂方法模式去掉了简单工厂模式中工厂方法的静态属性，使得它可以被子类继承。这样在简单工厂模式里集中在工厂方法上的压力可以由工厂方法模式里不同的工厂子类来分担。 
工厂方法模式组成： 
    1)抽象工厂角色： 这是工厂方法模式的核心，它与应用程序无关。是具体工厂角色必须实现的接口或者必须继承的父类。在java中它由抽象类或者接口来实现。 
    2)具体工厂角色：它含有和具体业务逻辑有关的代码。由应用程序调用以创建对应的具体产品的对象。 
    3)抽象产品角色：它是具体产品继承的父类或者是实现的接口。在java中一般有抽象类或者接口来实现。 
    4)具体产品角色：具体工厂角色所创建的对象就是此角色的实例。在java中由具体的类来实现。 

​			

![屏幕快照 2020-09-07 下午1.01.19](/Users/meiliwu/Desktop/屏幕快照 2020-09-07 下午1.01.19.png)

	//抽象产品类
	public class BMW {
	    public BMW() {
	        System.out.println("宝马");
	    }
	}
	
	
	//产品1
	public class BMW320 extends BMW {
	    public BMW320() {
	        System.out.println("BMW 320");
	    }
	}
	
	//产品2
	public class BMW520 extends BMW {
	    public BMW520() {
	        System.out.println("BMW 520");
	    }
	}
	
	//抽象工厂类
	public interface BMWFactory {
	       BMW createCar();
	}
	
	
	//BMW320工厂
	public class BMW320Factory implements BMWFactory {
	    @Override
	    public BMW320 createCar() {
	        return new BMW320();
	    }
	}
	
	//BMW520工厂
	public class BMW520Factory implements BMWFactory {
	    @Override
	    public BMW520 createCar() {
	        return new BMW520();
	    }
	}
	
	
	//客户类
	public class Customer {
	    public static void main(String[] args) {
	        //工厂方法模式
	        BMW320Factory bmw320Factory = new BMW320Factory();
	        BMW520Factory bmw520Factory = new BMW520Factory();
	        System.out.println(bmw320Factory.createCar());
	        System.out.println(bmw520Factory.createCar());
	    }
	}
## 抽象工厂模式

工厂方法模式：
一个抽象产品类，可以派生出多个具体产品类。  
一个抽象工厂类，可以派生出多个具体工厂类。  
每个具体工厂类只能创建一个具体产品类的实例。
抽象工厂模式：
多个抽象产品类，每个抽象产品类可以派生出多个具体产品类。  
一个抽象工厂类，可以派生出多个具体工厂类。  
每个具体工厂类可以创建多个具体产品类的实例。  
区别：
工厂方法模式只有一个抽象产品类，而抽象工厂模式有多个。  
工厂方法模式的具体工厂类只能创建一个具体产品类的实例，而抽象工厂模式可以创建多个。

![image-20200907130753308](/Users/meiliwu/Library/Application Support/typora-user-images/image-20200907130753308.png)

```
//抽象产品1
public abstract class Engine {
}

//具体产品1A
public class EngineA extends Engine {
    public EngineA() {
        System.out.println("高端车专用引擎");
    }
}

//具体产品1B
public class EngineB extends Engine {
    public EngineB() {
        System.out.println("低端车专用引擎");
    }
}

//抽象产品B
public abstract class AirConditioner {
}

//具体产品1
public class AirConditionerA extends AirConditioner {
    public AirConditionerA() {
        System.out.println("高端车专用空调");
    }
}

//具体产品2
public class AirConditionerB extends AirConditioner {
    public AirConditionerB() {
        System.out.println("低端车专用空调");
    }
}

//消费者
public class Customer {  
    public static void main(String[] args){  
        //生产宝马320系列配件
        FactoryBMW320 factoryBMW320 = new FactoryBMW320();  
        factoryBMW320.createEngine();
        factoryBMW320.createAircondition();
          
        //生产宝马523系列配件  
        FactoryBMW523 factoryBMW523 = new FactoryBMW523();  
        factoryBMW320.createEngine();
        factoryBMW320.createAircondition();
    }  
}
```



## 总结： 

无论是简单工厂模式，工厂方法模式，还是抽象工厂模式，他们都属于工厂模式，在形式和特点上也是极为相似的，他们的最终目的都是为了解耦。在使用时，我们不必去在意这个模式到底工厂方法模式还是抽象工厂模式，因为他们之间的演变常常是令人琢磨不透的。经常你会发现，明明使用的工厂方法模式，当新需求来临，稍加修改，加入了一个新方法后，由于类中的产品构成了不同等级结构中的产品族，它就变成抽象工厂模式了；而对于抽象工厂模式，当减少一个方法使的提供的产品不再构成产品族之后，它就演变成了工厂方法模式。

​    所以，在使用工厂模式时，只需要关心降低耦合度的目的是否达到了。





# 中介设计模式

中介者模式（Mediator Pattern）是用来降低多个对象和类之间的通信复杂性。这种模式提供了一个中介类，该类通常处理不同类之间的通信，并支持松耦合，使代码易于维护。中介者模式属于行为型模式。



**意图：**用一个中介对象来封装一系列的对象交互，中介者使各对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。

**主要解决：**对象与对象之间存在大量的关联关系，这样势必会导致系统的结构变得很复杂，同时若一个对象发生改变，我们也需要跟踪与之相关联的对象，同时做出相应的处理。

**何时使用：**多个类相互耦合，形成了网状结构。

**如何解决：**将上述网状结构分离为星型结构。

**关键代码：**对象 Colleague 之间的通信封装到一个类中单独处理。

**应用实例：** 1、中国加入 WTO 之前是各个国家相互贸易，结构复杂，现在是各个国家通过 WTO 来互相贸易。 2、机场调度系统。 3、MVC 框架，其中C（控制器）就是 M（模型）和 V（视图）的中介者。

**优点：** 1、降低了类的复杂度，将一对多转化成了一对一。 2、各个类之间的解耦。 3、符合迪米特原则。

**缺点：**中介者会庞大，变得复杂难以维护。

**使用场景：** 1、系统中对象之间存在比较复杂的引用关系，导致它们之间的依赖关系结构混乱而且难以复用该对象。 2、想通过一个中间类来封装多个类中的行为，而又不想生成太多的子类。

**注意事项：**不应当在职责混乱的时候使用。

```
//客户
public class Client {
    private int number;
    private Mediator mediator;
    private House house;

    public Client(int number, Mediator mediator) {
        this.number = number;
        this.mediator = mediator;
    }

    public Client(House house, Mediator mediator) {
        this.house = house;
        this.mediator = mediator;
    }

    public void buyHouse(int number) {
        this.number -= number;
        mediator.buyHouse(number);
        System.out.println("当前余额是 " + number);
    }

    public void sellHouse(int money) {
        mediator.sellHouse(money);
    }

    public int getNumber() {
        return number;
    }

    public void setNumber(int number) {
        this.number += number;
    }

    public Mediator getMediator() {
        return mediator;
    }

    public void setMediator(Mediator mediator) {
        this.mediator = mediator;
    }

    public House getHouse() {
        return house;
    }

    public void setHouse(House house) {
        this.house = house;
    }

    @Override
    public String toString() {
        if (house != null) {
            return "通过中介" + mediator.toString() + "购入一套房子" + house.toString() + "当前余额是" + number;
        } else {
            return "通过中介" + mediator.toString() + "卖出一套房子" + "当前余额是" + number;
        }
    }

}

//中介
public class Mediator {
    private Client buyer;
    private Client seller;

    public Mediator() {

    }

    public Mediator(Client A, Client B) {
        this.buyer = A;
        this.seller = B;
    }


    public void buyHouse(int money) {
        handleBusiness(money);
    }

    public void sellHouse(int money) {
        handleBusiness(money);
    }

    public void handleBusiness(int money) {
        this.seller.setNumber(money);
        this.buyer.setNumber(this.buyer.getNumber() - money);
        House house = seller.getHouse();
        this.buyer.setHouse(house);
        this.seller.setHouse(null);
    }
}

//交易场景
 
        Client buyer = new Client(10000,null);
        House house = new House("天安门");
        Client seller = new Client(10000, house);

        Mediator mediator = new Mediator(buyer, seller);
        mediator.setName("张三");
        buyer.setMediator(mediator);
        seller.setMediator(mediator);

        buyer.buyHouse(1000);
        System.out.println(buyer.toString());
        System.out.println(seller.toString());
        
        
        //通过中介张三购入一套房子天安门当前余额是9000
				//通过中介张三卖出一套房子当前余额是11000
```

