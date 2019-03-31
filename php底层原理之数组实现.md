数组是PHPer最常用的数据类型，同时php容易上手也得益于其强大的数组，但是数组在php中是如何实现的呢？

首先，我们还是先了解下相关的数据结构，为下面的内容打好基础

## 哈希表
`哈希表`，顾名思义，即将不同的关键字映射到不同单元的一种数据结构。而将不同关键字映射到不同单元的方法就叫做`哈希函数`

理想情况下，经过哈希函数处理，关键字和单元是会进行一一对应的；但是如果关键字值足够多的情况下，就容易出现多个关键字映射到同一单元的情况，即出现`哈希冲突`

哈希冲突的解决方案，要么使用`链接法`，要么使用`开放寻址法`

**链接法**   
即当不同的关键字映射到同一单元时，在同一单元内使用链表来保存这些关键字

**开放寻址法**   
即当插入数据时，如果发现关键字被映射到的单元存在数据了，说明发生了冲突，就继续寻找下一个单元，直到找到可用单元为止

而因为开放寻址法方案属于占用其他关键字映射单元的位置，所以后续的关键字更容易出现哈希冲突，因此容易出现性能下降

## 链表
既然上面提到了链表，这里我们简单聊一下链表的基础知识。链表分为很多种类型，常用的数据结构包括：队列，栈，双向链表等

链表，就是由不同的链表节点组成的一种数据结构。链表节点一般由`元素`+`指向下一节点的指针`组成。而双向链表，顾名思义，则是由`指向上一节点的指针`+`元素`+`指向下一节点的指针`组成

对于数据结构的内容，我们不过多展开，我们之后会有专门的内容去详细介绍数据结构

## php数组
php解决哈希冲突的方式是使用了链接法，所以php数组是由`哈希表`+`链表`实现，准确来说，是由`哈希表`+`双向链表`实现

### 内部结构-哈希表

HashTable结构体主要用来存放哈希表的基本信息

```c
typedef struct _hashtable { 
    uint nTableSize;        // hash Bucket的大小，即哈希表的容量，最小为8，以2x增长。
    uint nTableMask;        // nTableSize-1 ， 索引取值的优化
    uint nNumOfElements;    // hash Bucket中当前存在的元素个数，count()函数会直接返回此值 
    ulong nNextFreeElement; // 下一个可使用的数字键值
    Bucket *pInternalPointer;   // 当前遍历的指针（foreach比for快的原因之一）
    Bucket *pListHead;          // 存储整个哈希表的头元素指针
    Bucket *pListTail;          // 存储整个哈希表的尾元素指针
    Bucket **arBuckets;         // 存储hash数组
    dtor_func_t pDestructor;    // 在删除元素时执行的回调函数，用于资源的释放
    zend_bool persistent;       //指出了Bucket内存分配的方式。如果persisient为TRUE，则使用操作系统本身的内存分配函数为Bucket分配内存，否则使用PHP的内存分配函数。
    unsigned char nApplyCount; // 标记当前hash Bucket被递归访问的次数（防止多次递归）
    zend_bool bApplyProtection;// 标记当前hash桶允许不允许多次访问，不允许时，最多只能递归3次
#if ZEND_DEBUG
    int inconsistent;
#endif
} HashTable;
```

Bucket结构体则用于保存数据的具体内容

```c
typedef struct bucket {
    ulong h;            // 对char *key进行hash后的值，或者是用户指定的数字索引值
    uint nKeyLength;    // hash关键字的长度，如果数组索引为数字，此值为0
    void *pData;        // 指向value，一般是用户数据的副本，如果是指针数据，则指向pDataPtr
    void *pDataPtr;     // 如果是指针数据，此值会指向真正的value，同时上面pData会指向此值
    struct bucket *pListNext;   // 指向整个哈希表的该单元的下一个元素
    struct bucket *pListLast;   // 指向整个哈希表的该单元的上一个元素
    struct bucket *pNext;       // 指向由于哈希冲突导致存放在同一个单元的链表中的下一个元素
    struct bucket *pLast;       // 指向由于哈希冲突导致存放在同一个单元的链表中的上一个元素
    // 保存当前值所对于的key字符串，这个字段只能定义在最后，实现变长结构体
    char arKey[1];              
} Bucket;
```
其中Bucket结构体内有指向用户数据的pData元素，其实是指向了之前我们介绍的变量zval结构体，这也是为什么当创建数组时，会出现数组元素+1的变量容器。不了解变量底层相关知识的，请查看我之前的文章：

[php底层原理之变量（一）](https://github.com/sdxuzheng/php-internal/blob/master/php%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86%E4%B9%8B%E5%8F%98%E9%87%8F%EF%BC%88%E4%B8%80%EF%BC%89.md)   
[php底层原理之变量（二）](https://github.com/sdxuzheng/php-internal/blob/master/php%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86%E4%B9%8B%E5%8F%98%E9%87%8F%EF%BC%88%E4%BA%8C%EF%BC%89.md)   

**哈希表内部结构关系图**

![哈希表内部结构关系图](https://segment-xavier.oss-cn-beijing.aliyuncs.com/php%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86%E4%B9%8B%E6%95%B0%E7%BB%84%E5%AE%9E%E7%8E%B0/Zend%E5%BC%95%E6%93%8E%E5%93%88%E5%B8%8C%E8%A1%A8%E7%BB%93%E6%9E%84%E5%92%8C%E5%85%B3%E7%B3%BB.png)   
注：图片来源于网络   

从上图我们可以看出，Bucket在存放数据的时候，如果存在哈希冲突，则将多个关键字映射到链表中，由此组成了双向链表

## 总结
今天，我们以数组作为切入点，简单了解了下基本的数据结构：哈希表和链表；并且了解了数组的底层实现，即`哈希表`+`双向链表`。其实哈希表作为php中最重要的数据结构，用处很广。变量的符号表，函数列表等都是用哈希表来存储的，感兴趣的同学可以看我之前的文章来了解相关知识
