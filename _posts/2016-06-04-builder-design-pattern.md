---
layout: post
categories: [设计模式]
tags: [设计模式]
code: true
title: 多个构造器参数考虑用Builder模式
---

&emsp;&emsp;当一个类的构造参数过多，或者将会越来越多的时候，可以使用Builder模式来创建该类。  
&emsp;&emsp;一般工作中看到大部分人用JavaBean来作为参数解决这个问题，简单明了，但是大部分情况下这些参数可能没有关联，抽象出这么个类不优雅，可读性差。如果直接让该类提供set参数方法，将构造过程分解到调用多个set方法的过程中，在构造过程中JavaBean可能处于不一致的状态。  
用Builder模式实现:  

```
package hello;

/**
 * Person
 *
 * @author crystalChen
 * @date 16/6/4 10:13
 */
public class Person {
    private final int id;
    private final String lastName;
    private final String firstName;
    private final boolean sex;
    private final int age;

    private Person(Builder builder) {
        id = builder.id;
        lastName = builder.lastName;
        firstName = builder.firstName;
        sex = builder.sex;
        age = builder.age;
    }
    
    public static class Builder {

        private final int id;
        private final String lastName;
        private final String firstName;
        private boolean sex;
        private int age;

        //  传入必填字段      
        public Builder(int id, String lastName, String firstName) {
            this.id = id;
            this.lastName = lastName;
            this.firstName = firstName;
        }

        public Builder sex(boolean b) {
            sex = b;
            return this;
        }
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        
        public Person build() {
            return new Person(this);
        }

    }
    
    public static void main(String[] args) {
        new Person.Builder(360,"crystal","chen").sex(true).age(23);
    }
}
```