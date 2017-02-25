---
layout: post
categories: [Java]
tags: [Java]
code: true
title: 由增加一个类的属性对Java序列化的思考
---

 

​	因为一次业务升级，需要对一个Javabean FeedTemplate增加一个属性canClick，这个类是是个模板，类似配置，存在MySQL表中，程序读取的时候先从Redis中读，若Redis过期则读库，然后将对象序列化更新到Redis中。上线步骤应该是先给MySQL feed_template表增加字段，然后更新各个节点机器上的代码，最后清除Redis缓存。这个上线过程有没有问题呢？

​	首先feed_template表这个过程中只有查询，而且can_click字段默认值为0，所以即使有插入语句也没有；其次，先给MySQL增加了字段，程序select操作是无法感知的，因为程序的dao指明了select表的具体字段，所以Redis和Java里面都是原来的老数据；最后，在上线代码过程中，程序里面FeedTemplate类增加了属性，如果从Redis里面读取序列化的FeedTemplate对象反序列的话时候会不会有问题呢？如果第一个节点代码更新上线后，其他节点还没有更新，这个间隙中Redis刚好失效了，从MySQL里面读取了新的数据，更新到Redis当中，还未上线的节点反序列化会不会有问题呢？

​	问题归根结底，就是Java对象新增或删除属性会不会对反序列化有影响。当时马上在项目上演练了一番，没有问题。然而，我觉得问题不像表面那么简单。立刻上网查找资料，[Java对象的序列化和反序列化](http://www.cnblogs.com/dubo-/p/5608641.html)

一文中指出，是否声明serialVersionUID对于对象序列化的向上向下的兼容性有很大的影响。如果serialVersionUID去掉，序列化保存。反序列化的时候，增加或减少个字段，会报java.io.InvalidClassException。因为据Java API上描述，不显示定义serialVersionUID，Java会根据该类的各方面计算生成一个serialVersionUID，根据编译器实现的不同可能千差万别，这样在反序列化过程中可能会导致意外的 InvalidClassException。所以强烈建议显式声明serialVersionUID，并用private修饰。

​	晚上从公司回到房间，翻阅《Thinking in Java》，发现我知道的序列化只是冰山一角。Java对象序列化的出现主要的一个原因是为了RMI，Java对象序列化提供了几种方法：

​	1.implements Serializable接口

实现Serializable接口即可，不需重写什么方法，这样就可以自动序列化。注意，如果不想让某个特定的字段被Java序列化自动保存与恢复。比方密码，即使是private修饰，序列化成字节后还是可以通过读取文件或者拦截网络传输被识别的。可以使用transient（瞬时）关键字关闭序列化。示例：

`

```java
import java.io.*;
import java.util.Date;
import java.util.concurrent.TimeUnit;


public class Logon implements Serializable {

    private static final long serialVersionUID = 1L;
	
    private Date date = new Date();
    private String name;
    private transient String password;
    public Logon(Date date, String name, String password) {
        this.date = date;
        this.name = name;
        this.password = password;
    }

//    private void writeObject(ObjectOutputStream out) throws IOException //{
//        out.defaultWriteObject();
//        out.writeObject(password);
//    }
//
//    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
//        in.defaultReadObject();
//        this.password = (String) in.readObject();
//    }
  
  
    @Override
    public String toString() {
        return "name:" + name + ", password:" + password + ", date:" + date;
    }

    public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
        Logon logon = new Logon(new Date(), "Julia", "123aaa");
        System.out.println(logon.toString());
        ObjectOutputStream o = new ObjectOutputStream(new FileOutputStream("logon.out"));
        o.writeObject(logon);
        o.close();
        TimeUnit.SECONDS.sleep(1);
        ObjectInputStream in = new ObjectInputStream(new FileInputStream("logon.out"));
        logon = (Logon)in.readObject();
        System.out.println(logon.toString());
    }
}
```

`

被transient关键字修饰，反序列化得到的password是null。其实有个小魔法，添加，注意是添加被注释的两个方法，被transient关键字修饰的password也被序列化进去了。这两个方法是约定名称被ObjectOutputStream的writeObject()和ObjectInputStream的readObject()调用了。混乱不堪。



2.implements Externalizable接口

Externalizable接口继承了Serializable接口，同时增添了两个方法：writeExternal()和readExternal()。这两个方法会在序列化和反序列化的过程中被自动调用，以便执行一些特殊操作。Externalizable不会自动序列化，需要那两个增添的方法里面写实现，控制空间比较大，不需要序列化的可以不写。注意到，实现Externalizable的类需必须要有public的无参数的构造方法，否则报InvalidClassException，因为在反序列化的时候会先调用默认构造器，然后再调用readExternal()。例如：

`

```java
import java.io.*;
import java.util.Date;
import java.util.concurrent.TimeUnit;


public class Logon implements Externalizable {
    private Date date = new Date();
    private String name;
    private transient String password;
    private byte age;

    public Logon() {
        System.out.println("valid constructor");
    }

    public Logon(Date date, String name, String password, byte age) {
        this.date = date;
        this.name = name;
        this.password = password;
        this.age = age;
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeObject(date);
        out.writeObject(name);
        out.writeByte(age - 16);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        this.date = (Date)in.readObject();
        this.name = (String)in.readObject();
        this.age = (byte) (in.readByte() + 16);
    }

    @Override
    public String toString() {
        return "name:" + name + ", password:" + password + ", date:" + date + " age:" +age;
    }

    public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
        Logon logon = new Logon(new Date(), "Julia", "123aaa", (byte) 20);
        System.out.println(logon.toString());
        ObjectOutputStream o = new ObjectOutputStream(new FileOutputStream("logon.out"));
        o.writeObject(logon);
        o.close();
        TimeUnit.SECONDS.sleep(1);
        ObjectInputStream in = new ObjectInputStream(new FileInputStream("logon.out"));
        logon = (Logon)in.readObject();
        System.out.println(logon.toString());
    }

}

/* Output:
name:Julia, password:123aaa, date:Sat Feb 25 22:07:37 CST 2017 age:20
valid constructor
name:Julia, password:null, date:Sat Feb 25 22:07:37 CST 2017 age:20
*/
```

`

可以看到，这里序列化和反序列都是自己实现的，也没有序列化密码，而且对年龄做了简单加密处理。活动空间较大。

3.XML
XML格式具有跨语言性。具体查看《Thinking in Java》

4.Preferences
Preferences API可以自动存储和读取信息。不过受限于存储基本类型和字符串，并且字符串长度不超过8K。Preferences是一个K-V集合，类似Map。具体查看《Thinking in Java》


#### 总结：
1.实现对象序列化强烈建议声明private static final long serialVersionUID
2.Serializable接口自动序列化和反序列化，用transient修饰的变量不参与
3.Externalizable接口需要自己输出序列化和输入反序列化的字段
4.被序列化的对象的属性都需要支持序列化
5.Java序列化可以保持对象的"全景图"，能跟踪对象内所包含的所有引用，实现了深度复制。意味着对象A引用对象C，对象B引用同一个对象C，反序列也是引用同一个
6.序列化多个对象的时候，注意"原子"操作进行序列化，可以将所有对象置入单一容器内，并在一个操作将该容器直接写出
7.考虑各个版本序列化的时间和空间复杂度
8.有些底层实现，注意各版本编译器是否兼容，比方要修改序列化类的版本
9.需要迁移序列化的类，比方HashTable变成HashMap，网上有讨论
10.序列化的字节是可以根据规范可读的。参见[Java的序列化](http://blog.csdn.net/silentbalanceyh/article/details/8183849)