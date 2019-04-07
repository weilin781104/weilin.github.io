# 基本数据类型和引用类型

Java是面向对象的编程语言，一切都是对象，但是为了编程的方便还是引入了基本数据类型，为了能够将这些基本数据类型当成对象操作，Java为每一个基本数据类型都引入了对应的包装类型（wrapper class），int的包装类就是Integer，从Java 5开始引入了自动装箱/拆箱机制，使得二者可以相互转换，对应如下：

原始类型：boolean，char，byte，short，int，long，float，double

包装类型：Boolean，Character，Byte，Short，Integer，Long，Float，Double

Java中的基本数据类型只有以上8个，除了基本类型（primitive type），剩下的都是引用类型（reference type）。

三种引用类型：

类class

接口interface

数组array


# int和Integer的区别

1、Integer是int的包装类，int则是java的一种基本数据类型
2、Integer变量必须实例化后才能使用，而int变量不需要
3、Integer实际是对象的引用，当new一个Integer时，实际上是生成一个指针指向此对象；而int则是直接存储数值
4、Integer的默认值是null，int的默认值是0

延伸：

关于Integer和int的比较

1、由于Integer变量实际上是对一个Integer对象的引用，所以两个通过new生成的Integer变量永远是不相等的（因为new生成的是两个对象，其内存地址不同）。

![](assets\20190407205911.jpg)

2、Integer变量和int变量比较时，只要两个变量的值是向等的，则结果为true（因为包装类Integer和基本数据类型int比较时，java会自动拆包装为int，然后进行比较，实际上就变为两个int变量的比较）

![](assets\20190407205925.jpg)

3、非new生成的Integer变量和new Integer()生成的变量比较时，结果为false。（因为非new生成的Integer变量指向的是java常量池中的对象，而new Integer()生成的变量指向堆中新建的对象，两者在内存中的地址不同）

![](assets\20190407210536.jpg)

4、对于两个非new生成的Integer对象，进行比较时，如果两个变量的值在区间-128到127之间，则比较结果为true，如果两个变量的值不在此区间，则比较结果为

![](assets\20190407205945.jpg)

不在-128-127之间：

![](assets\20190407205956.jpg)

对于第4条的原因：

java在编译Integer i = 100 ;时，会翻译成为Integer i = Integer.valueOf(100)；，而java API中对Integer类型的valueOf的定义如下：

![](assets\20190407210005.jpg)



java对于-128到127之间的数，会进行缓存，Integer i = 127时，会将127进行缓存，下次再写Integer j = 127时，就会直接从缓存中取，就不会new了。