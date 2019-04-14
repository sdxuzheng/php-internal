>对于PHPer而言，我们通常会把常用的功能封装成一个函数来进行复用，以提升开发效率。但是php到底是如何找到对应的函数的呢？今天，我们来一起寻找答案~

## 函数分类
首先，我们先回顾一下php的函数分类，php函数包含用户自定义函数、内部函数、匿名函数等多种类型。而对于用户自定义函数和内部函数，他们分别存在对应自己的数据结构，但是Zend引擎为了适配两种函数类型，所以定义了一种新的数据结构：`zend_function`联合体

## 数据结构
我们还是先看下`zend_function`联合体，了解下为什么针对用户自定义函数和内部函数要做适配
```c
typedef union _zend_function {
    zend_uchar type;    //函数类型，用来标记是用户自定义函数还是内部函数
    struct {
        zend_uchar type;  /* never used */
        char *function_name;    //函数名称
        zend_class_entry *scope; //函数所在的类作用域，用来标记作为成员方法时所属的类
        zend_uint fn_flags;     // 标记其作为类的成员方法时的访问类型，是public、protected还是private
        union _zend_function *prototype;  //函数原型，标记内部函数或者用户自定义函数所属的zend_function
        zend_uint num_args;     //函数的参数数量
        zend_uint required_num_args; //必传的参数数量
        zend_arg_info *arg_info;  //参数信息指针
        zend_bool pass_rest_by_reference;
        unsigned char return_reference;  //返回值 
    } common;
    zend_op_array op_array;   //用户自定义函数结构体
    zend_internal_function internal_function; //内部函数结构体
} zend_function;
struct _zend_op_array {
    /* Common elements (共有元素)*/
    zend_uchar type;
    char *function_name;        
    zend_class_entry *scope;
    zend_uint fn_flags;
    union _zend_function *prototype;
    zend_uint num_args;
    zend_uint required_num_args;
    zend_arg_info *arg_info;
    zend_bool pass_rest_by_reference;
    unsigned char return_reference;
    /* END of common elements */
 
    zend_bool done_pass_two;
    ....//  其它字段
}
typedef struct _zend_internal_function {
    /* Common elements (共有元素)*/
    zend_uchar type;
    char * function_name;
    zend_class_entry *scope;
    zend_uint fn_flags;
    union _zend_function *prototype;
    zend_uint num_args;
    zend_uint required_num_args;
    zend_arg_info *arg_info;
    zend_bool pass_rest_by_reference;
    unsigned char return_reference;
    /* END of common elements */
 
    void (*handler)(INTERNAL_FUNCTION_PARAMETERS);
    struct _zend_module_entry *module;
} zend_internal_function;
```
从上面介绍的内容，我们可以发现，不管是用户自定义函数还是内部函数，在底层存储时都会存在共有字段`type`和`common`，所以他们以联合体的方式共享内存，可以节省内存空间和快速获取函数的基本信息，并且如果有需要，可以在一些结构间完美的进行强制类型转换。即`zend_function`可以与`zend_op_array`互换，`zend_function`可以与`zend_internal_function`互换

## 函数注册
聊完了用户自定义函数和内部函数的数据结构存储，我们再来看下全局函数列表

全局函数列表，可以理解成函数注册表，其内部实现是一个哈希表。用户自定义函数和内部函数编译完成后会将函数名注册到全局函数列表中。也就是此时会判断是否全局函数列表中存在同名函数

那么用户自定义函数和内部函数在存储到全局函数列表时有什么不同呢？

用户自定义函数：
我们写的php函数在编译阶段会经过词法分析->语法分析->生成中间代码opcode->执行中间代码的过程，执行中间代码时，会将函数名注册到全局函数列表

内部函数：
php启动时，会加载所有扩展模块，并为扩展模块中每一个函数创建一个zend_internal_function结构，并将这个函数名注册到全局函数列表

## 函数调用
接下来，我们再来看下调用函数时，是如何执行的呢？

函数调用时，首先会根据函数名去全局函数列表内查找是否存在该函数，如果不存在，则会直接报出“Call to undefined function xxx()"的错误信息；如果存在，则获取该函数指针对应的函数结构中的type字段，判断其函数类型，如果函数类型是自定义函数，则调用zend_execute来执行函数的zend_op_array内容，而如果函数类型是内部函数，则直接调用zend_internal_function的handle指针指向的扩展模块的C函数

## 总结
到这里，大家应该可以找到开头我们提出的问题的答案了。其实就是通过函数注册到全局函数列表，然后函数调用时，再从全局函数列表中查找对应函数进行执行来实现的；只不过，对于用户自定义函数和内部函数而言，其实现方式不同，当然这也就意味着执行效率的不同（当然php内部函数执行效率更高了，因为它没有运行时的编译阶段，相当于直接执行c语言，所以能用php内部函数的尽量使用内部函数）

今天我们就聊到这里了，欢迎大家的手动点赞~
