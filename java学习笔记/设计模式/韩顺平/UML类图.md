# UML类图

UML统一建模语言(Undified modeling languange)



## 1.关系

Dependency 依赖使用

Assoication 表示关联，一对一

Generalization 表示泛化，继承关系

Aggregation 表示聚合，关联关系的特例；比如A类中有一个成员变量是B类，B类是通过一个set方法进行传递的。

Composite 表示组合



### 1.1类图--依赖关系（dependency）

只要是在类中用到了对方，那么他们之间就存在依赖关系。

可以是类的成员属性

可以是方法的方法类型

可以是方法接受的参数类型

可以是方法中的变量

```java
public class PersonServiceBean{
    //类
    private PersonDao personDao; 

    public void save(Person person){}
    
    public IDCard getIDCard(Integer personid){return null;}
    
    public void modify(){
        Department deparment=new Department();
    }
}
public class PersonDao{}
public class IDCard{}
public class Person{}
public class Department{}

```

![依赖](E:\笔记\java学习笔记\设计模式\韩顺平\依赖.png)



### 1.2 类图--泛化关系(generalization)

泛化关系实际上就是继承关系，它是**依赖关系的特例**

```java
public abstract class DaoSupport{
    public void save(Object entity){}
    
    public void delete(Object id);
}
public class PersonServiceBean extends Daosupport{}
```

![泛化](E:\笔记\java学习笔记\设计模式\韩顺平\泛化.png)

####  小结：

泛化关系实际上就是继承关系

如果A类继承了B类，我们就说A和B存在泛化关系



### 1.3 类图--实现关系(Implementation)

实现关系实际上就是A类实现B类，他是**依赖关系的特例**

```java
public interface PersonService{
    void delete(Integer id);
}
public class PersonServiceBean implements PersonService{
	@Override
    public void delete(Integer id){
        
    }
}
```



![实现](E:\笔记\java学习笔记\设计模式\韩顺平\实现.png)

### 1.4 类图--关联关系(Association)

关联关系实际上就是类与类之间的联系，它是**依赖关系的特例**

关联具有导航型：即双向关系或单向关系；

关系有多重性：如"1"（表示有且仅有一个）"0..."表示0个或者多个，

"0,1"表示0个或者一个，"n...m"表示n到m个都可以，"m...*"表示至少m个



#### 1.4. 1单向一对一关系

```java
public class Person{
    private IDCard card;
}
public class IDCard{}
//双向一对一关系
public class Person{
    private IDCard card;
}
public class IDCard{
    private Person person;
}
```



![一对一关联关系](E:\笔记\java学习笔记\设计模式\韩顺平\一对一关联关系.png)



### 1.5 类图--聚合关系(Aggregation)

聚合关系表示的是整体和部分的关系，整体与部分可以分开，**聚合关系是关联关系的特例**,所以它具有关联的导航型与多重行；

导航型，是A聚合B还是B聚合A。A聚合了几个B(多重聚合)

如一台电脑有键盘、显示器、鼠标组成。组成电脑的各个配件是可以从电脑上分离出来的，使用带空心棱形的实现来表示

鼠标可以分开是聚合，不可以分开是组合的关系

```java
public class Computer{
    private Mouse mouse;
    private Monitor monitor;
    
    public void setMouse(Mouse mouse){
        this.mouse=mouse;
    }
    public void setMonitor(Monitor monitor){
        this.monitor=monitor;
    }
}
```

![聚合关系](E:\笔记\java学习笔记\设计模式\韩顺平\聚合关系.png)

### 1.6 类图--组合(Composition)

如果我们从Mouse、Monitor和Computer是不可分离的，则升级为组合关系

```java
public class Computer{
    private Mouse mouse=new Mouse();
    private Monitor monitor=new Monitor();
}
public class Client{
    main{
        Computer computer =new Computer();
    }
}
```

组合关系：也是整体与部分的关系，**但是整体与部分不可以分开。**

在程序中我们定义实体：Person与IDCard、Head，那么Head和Person就是组合，IDCard和Person是聚合

```java
public class Person{
    private IDCard card;
    private Head head=new Head();
}
public class IDCard{}
public class Head{}
```

![组合](E:\笔记\java学习笔记\设计模式\韩顺平\组合.png)

**小结：**

如果在程序中Person定义了对IDCard进行级联删除，即删除Person时连同IDCard一起删除，那么IDCard和Person就是组合了。

级联删除，删除一个对象，连同IDCard也删除。