---
title: 重写equals()方法的时候一定要重写hashCode()方法吗？
date: 2020-05-29 20:20:24
tags:
  - Java
---

我们知道，当我们用`HashMap`存入自定义的类时，如果不重写这个自定义类的`equals`和`hashCode`方法，得到的结果会和我们预期的不一样。

下面以一个Person类为基础，观察其在：

- 不重写`equals`和`hashCode`方法
- 重写`equals`但不重写`hashCode`方法
- 重写`equals`和`hashCode`方法

三种情况下的表现。

Person定义如下：

```java
public class Person {

    // 身份证号
    private String id;
    // 名字
    private String name;

    // getter...
    // setter...

}
```

### 不重写equals和hashCode方法

测试代码：

```java
public static void main(String[] args) {
    Person p1 = new Person();
    p1.setId("123456");
    p1.setName("zhangsan");

    Person p2 = new Person();
    p2.setId("123456");
    p2.setName("lisi");

    System.out.println(p1.equals(p2));
    System.out.println(p1.hashCode());
    System.out.println(p2.hashCode());

    HashMap<Person, String> map1 = new HashMap<>();
    map1.put(p1, "哈哈哈");
    map1.put(p2, "呵呵呵");
    System.out.println(map1.size());
}
```

输出结果：

```
false
1627674070
1360875712
2
```

很明显可以看到，同一个类型不重写`equals`和`hashCode`方法，两个实例的私有属性值也不一样，两个实例肯定不同。所以其`hash`值肯定也不一样，放到`HashMap`中当然也会被认为是两个不同的k，所以`map1`的大小为2。这个不难理解吧。

### 重写equals但不重写hashCode方法

`equals`方法重写如下：

```java
@Override
public boolean equals(Object obj) {
    // 以身份证号为准
    return this.id.equals(((Person) obj).getId());
}
```

当两个人身份证号一样的时候，我们就认为其是同一个人。即两个实例相同。

同样的测试代码下输出结果：

```java
true
1627674070
1360875712
2
```

可以看到，重写了`equals`方法之后，由于我们是以`id`为基准的，所以两个实例的身份证号只要一样，我们就认为他们是同一个人，也就是两个实例相同。但是两个实例的`hash`值却不同，又因为`HashMap`是以k的的`hash`值进行比较的，所以`map1`的大小仍然是2。

### 重写equals和hashCode方法

`hashCode`方法重写如下：

```java
@Override
public int hashCode() {
    // 取身份证号的hash值
    return getId().hashCode();
}
```

输出结果：

```
true
1450575459
1450575459
1
```

可以看到，两个实例就算名字不相等，只要其身份证号一样，那么我们就可以认为他俩是同一个人；由于其`hash`值（取身份证号的`hash值`）也相等，`HashMap`也认为其同一个实例，尽管其名字不一样。

## 总结

当两个方法都不重写的时候，两个实例的身份证号一样，但名字不一样，会被认为不是同一个人；当只重写`equals`方法的时候，只要两个实例的身份证号一样，我们就单方面认为其是同一个人，但是，以`hash`表为底层实现的一类容器却不认为他们是同一个人；只有两个方法都经过重写的时候，只要保证其`hash`值相同，就算其它属性不相同，这类容器也还会认为他们是一个人。

所以，重写equals()方法的时候一定要重写hashCode()方法吗？答案是**不一定**。我认为是分场景的，如果能确保这个类型不会被用在任何以其实例`hash`值作对比的情况下，就可以只重写`equals`方法。如果不能确保，还是老老实实重写吧！