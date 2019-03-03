php垃圾回收机制，对于PHPer来说是一个不陌生但是又不是很熟悉的内容。那么php是怎么实现对不需要的内存进行回收的呢？

##php变量的内部存储结构
首先还是需要了解下基础知识，便于垃圾回收原理内容的理解。大家都知道php是由C编写而成的，所以php变量的内部存储结构也会和C语言相关，即zval的结构体：
```c
struct _zval_struct {
	union {
		long lval;
		double dval;
		struct {
			char *val;
			int len;
		} str;
		HashTable *ht;
		zend_object_value obj;
		zend_ast *ast;
	} value;					//变量value值
	zend_uint refcount__gc;   //引用计数内存中使用次数，为0删除该变量
	zend_uchar type;		   //变量类型
	zend_uchar is_ref__gc;    //区分是否是引用变量
};
```
从上面结构体内容可以看出每一个php变量都会由`变量类型`、`value值`、`引用计数次数`和`是否是引用变量`四部分组成

注：上面zval结构体是php5.3版本之后的结构，php5.3之前因为没有引入新的垃圾回收机制，即GC，所以命名也没有`_gc`；而php7版本之后由于性能问题所以改写了zval结构，这里不再表述

##引用计数原理
了解了php变量的内部存储结构之后，我们再了解下php变量赋值相关的原理和早期垃圾回收机制

###变量容器
####非array和object变量
每次将常量赋值给一个变量时，都会产生一个变量容器

举例：
```php
$a = '许铮的技术成长之路';
xdebug_debug_zval('a')
```
结果：
```php
a: (refcount=1, is_ref=0)='许铮的技术成长之路'
```

####array和object变量
会产生元素个数+1的变量容器

举例：
```php
$b = [
'name' => '许铮的技术成长之路',
'number' => 3
];
xdebug_debug_zval('b')
```
结果：
```php
b: (refcount=1, is_ref=0)=array ('name' => (refcount=1, is_ref=0)='许铮的技术成长之路', 'number' => (refcount=1, is_ref=0)=3)
```

###赋值原理（写时复制技术）
了解了常量赋值之后，接下来我们从内存角度思考变量之间的赋值

举例：
```php
$a = [
'name' => '许铮的技术成长之路',
'number' => 3
]; //创建一个变量容器，变量a指向给变量容器，a的ref_count为1
$b = $a; //变量b也指向变量a指向的变量容器，a和b的ref_count为2
xdebug_debug_zval('a', 'b');
$b['name'] = '许铮的技术成长之路1';//变量b的其中一个元素发生改变，此时会复制出一个新的变量容器，变量b重新指向新的变量容器，a和b的ref_count变成1
xdebug_debug_zval('a', 'b'); 
```
结果：
```php
a: (refcount=2, is_ref=0)=array ('name' => (refcount=1, is_ref=0)='许铮的技术成长之路', 'number' => (refcount=1, is_ref=0)=3)
b: (refcount=2, is_ref=0)=array ('name' => (refcount=1, is_ref=0)='许铮的技术成长之路', 'number' => (refcount=1, is_ref=0)=3)
a: (refcount=1, is_ref=0)=array ('name' => (refcount=1, is_ref=0)='许铮的技术成长之路', 'number' => (refcount=1, is_ref=0)=3)
b: (refcount=1, is_ref=0)=array ('name' => (refcount=1, is_ref=0)='许铮的技术成长之路1', 'number' => (refcount=1, is_ref=0)=3)
```

所以，当变量a赋值给变量b的时候，并没有立刻生成一个新的变量容器，而是将变量b指向了变量a指向的变量容器，即内存"共享"；而当变量b其中一个元素发生改变时，才会真正发生变量容器复制，这就是`写时复制技术`

###引用计数清0
当变量容器的ref_count计数清0时，表示该变量容器就会被销毁，实现了内存回收，这也是`php5.3版本之前的垃圾回收机制`

举例：
```php
$a = "许铮的技术成长之路";
$b = $a;
xdebug_debug_zval('a');
unset($b);
xdebug_debug_zval('a');
```

结果：
```php
a: (refcount=2, is_ref=0)='许铮的技术成长之路'
a: (refcount=1, is_ref=0)='许铮的技术成长之路'
```

###循环引用引发的内存泄露问题
但是php5.3版本之前的垃圾回收机制存在一个漏洞，即当数组或对象内部子元素引用其父元素，而此时如果发生了删除其父元素的情况，此变量容器并不会被删除，因为其子元素还在指向该变量容器，但是由于所有作用域内都没有指向该变量容器的符号，所以无法被清除，因此会发生内存泄漏，直到该脚本执行结束

举例：
```php
$a = array( 'one' );
$a[] = &$a;
xdebug_debug_zval( 'a' );
```

由于该示例不好输出结果，用图表示，如图：
![循环引用](https://segment-xavier.oss-cn-beijing.aliyuncs.com/php%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86%E4%B9%8B%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E6%9C%BA%E5%88%B6/%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8.png)

举例：
```php
unset($a);
xdebug_debug_zval('a');
```
如图：
![循环引用的内存泄漏](https://segment-xavier.oss-cn-beijing.aliyuncs.com/php%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86%E4%B9%8B%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E6%9C%BA%E5%88%B6/%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8%E5%BC%95%E8%B5%B7%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F.png)

##新的垃圾回收机制
php5.3版本之后引入根缓冲机制，即php启动时默认设置指定zval数量的根缓冲区（默认是10000），当php发现有存在循环引用的zval时，就会把其投入到根缓冲区，当根缓冲区达到配置文件中的指定数量（默认是10000）后，就会进行垃圾回收，以此解决循环引用导致的内存泄漏问题

###确认为垃圾的准则
1、如果引用计数减少到零，所在变量容器将被清除(free)，不属于垃圾
2、如果一个zval 的引用计数减少后还大于0，那么它会进入垃圾周期。其次，在一个垃圾周期中，通过检查引用计数是否减1，并且检查哪些变量容器的引用次数是零，来发现哪部分是垃圾。

##总结
垃圾回收机制：
1、以php的引用计数机制为基础（php5.3以前只有该机制）
2、同时使用根缓冲区机制，当php发现有存在循环引用的zval时，就会把其投入到根缓冲区，当根缓冲区达到配置文件中的指定数量后，就会进行垃圾回收，以此解决循环引用导致的内存泄漏问题（php5.3开始引入该机制）
