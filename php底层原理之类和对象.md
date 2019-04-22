>对于PHPer来说，OOP是不可或缺的开发思维，但是你对php类和对象的底层实现又了解多少呢？本着知其然且知其所以然的思想，让我们一起来寻找答案~

类的底层实现可看作是之前我们讲过的变量、函数等的知识集合。所以想要理解更深入的同学最好查看下我之前的关于介绍变量、函数的文章

## 类的数据结构
不管是普通类还是抽象类或是接口，都存放到统一的结构体中，并且在生成中间代码时，会将此类添加到全局类列表中。当然，也是在此时，会通过类名判断该类是否已经存在，如果存在，则添加失败
```c
struct _zend_class_entry {
    char type;     // 和函数一样，类被拆分为两种类型：ZEND_INTERNAL_CLASS 内部类型和ZEND_USER_CLASS 用户自定义类型
    char *name;// 类名称
    zend_uint name_length;                  // 即sizeof(name) - 1
    struct　_zend_class_entry *parent; // 继承的父类
    int　refcount;  // 引用数
    zend_bool constants_updated;
 
    zend_uint ce_flags;	//类的类型，在编译阶段被区分是普通类，接口，抽象类
    HashTable function_table;      // 静态类方法和普通类方法存放集合
    HashTable default_properties;          // 默认属性存放集合
    HashTable properties_info;     // 属性信息存放集合
    HashTable default_static_members;// 类本身所具有的静态变量存放集合
    HashTable *static_members; // type == ZEND_USER_CLASS时，取&default_static_members;
    // type == ZEND_INTERAL_CLASS时，设为NULL
    HashTable constants_table;     // 常量存放集合
    struct _zend_function_entry *builtin_functions;// 方法定义入口
 
	/* 魔术方法 */
    //所有魔术方法单独存放，初始化时被设置为null
    union _zend_function *constructor;
    union _zend_function *destructor;
    union _zend_function *clone;
    union _zend_function *__get;
    union _zend_function *__set;
    union _zend_function *__unset;
    union _zend_function *__isset;
    union _zend_function *__call;
    union _zend_function *__tostring;
    union _zend_function *serialize_func;
    union _zend_function *unserialize_func;
    zend_class_iterator_funcs iterator_funcs;// 迭代
 
    /* 类句柄 */
    zend_object_value (*create_object)(zend_class_entry *class_type TSRMLS_DC);
    zend_object_iterator *(*get_iterator)(zend_class_entry *ce, zval *object,
        intby_ref TSRMLS_DC);
 
    /* 类声明的接口 */
    int(*interface_gets_implemented)(zend_class_entry *iface,
            zend_class_entry *class_type TSRMLS_DC);
 
 
    /* 序列化回调函数指针 */
    int(*serialize)(zval *object， unsignedchar**buffer, zend_uint *buf_len,
             zend_serialize_data *data TSRMLS_DC);
    int(*unserialize)(zval **object, zend_class_entry *ce, constunsignedchar*buf,
            zend_uint buf_len, zend_unserialize_data *data TSRMLS_DC);
 
 
    zend_class_entry **interfaces;  //  类实现的接口
    zend_uint num_interfaces;   //  类实现的接口数
 
 
    char *filename; //  类的存放文件地址 绝对地址
    zend_uint line_start;   //  类定义的开始行
    zend_uint line_end; //  类定义的结束行
    char *doc_comment;
    zend_uint doc_comment_len;
 
 
    struct _zend_module_entry *module; // 类所在的模块入口：EG(current_module)
};
```
由上面代码可以看出，类的成员变量、成员方法都是存放在各自的结构体中，而结构体的数据结构和之前讲解的变量和函数的数据结构一模一样，只不过编译后的成员变量和成员方法是存放在类结构体中而已

## 对象的生成
我们都知道，对象是new出来的，但是从底层来看，对象生成分为3步   
第一步：根据类名去全局类列表内查找该类是否存在，如果存在，则获取存储类的变量   
第二步：判断类是否是普通类（非抽象类或接口）；如果是普通类则给需要创建的对象存放的zval容器分配内存，并设置容器类型为IS_OBJECT   
第三步：执行对象初始化操作，将对象添加到全局对象列表（对象池）中

附上对象的数据结构：
```c
typedef struct _zend_object {
    zend_class_entry *ce; //对象的类结构
    HashTable *properties; //对象属性
    HashTable *guards; /* protects from __get/__set ... recursion */
} zend_object;
```

## 获取和设置成员变量

获取成员变量：   
第一步，获取对象的属性，从对象的properties查找是否存在与名称对应的属性，如果存在返回结果，如果不存在，转第二步   
第二步，如果存在get魔术方法，则调用此方法获取变量，如果不存在，则报错

设置成员变量：   
第一步，获取对象的属性，从对象的properties查找是否存在与名称对应的属性，如果存在且已有的值和需要设置的值相同，则不执行任何操作，否则执行变量赋值操作，如果不存在，转第二步   
第二步，如果存在_set魔术方法，则调用此方法设置变量，如果不存在，转第三步   
第三步，如果成员变量一直没有被设置过，则直接将此变量添加到对象的properties字段所在HashTable中。

## 总结
到今天为止，我们差不多已经将关于php的底层原理讲了一个遍了。当然，在这期间，不少同学跟我说，现在都已经逐渐开始使用php7了，你现在讲解的内容还是php5，会不会过时了？其实我讲解php5也是为讲php7作准备，php7毕竟是php5的延展，了解了php5之后，再去了解php7会更加容易些。而且php也是从php5开始才逐渐完善起来的，我们有必要了解下php5的内容。不过从下周开始，我们会开始从底层比较php7和php5的不同，敬请期待~
