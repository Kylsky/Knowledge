1.下面的代码哪些地方会产生编译错误？

```java
class Outer {
	
	class Inner {}
	
	public static void foo() { new Inner(); }
	
	public void bar() { new Inner(); }
	
	public static void main(String[] args) {
		new Inner();
	}
}

```

Java中非静态内部类对象的创建要依赖其外部类对象，上面的面试题中foo和main方法都是静态方法，静态方法中没有this，也就是说没有所谓的外部类对象，因此无法创建内部类对象，如果要在静态方法中创建内部类对象，可以这样做：

```java
	new Outer().new Inner();
```



2.finally 是不是一定会执行？在finally中return会怎样？

```
https://www.jianshu.com/p/30d355b39a51
```

