# isa

内存优化之ISA是什么？

大家通常是否会认为isa就是对象的指针，用来表明对象所属的类型。
但是如果isa指针仅表示类型的话，对内存显然也是一个极大的浪费。于是，就像tagged pointer一样，对于isa指针，苹果同样进行了优化。使得isa指针表示的内容变得更为丰富，除了表明对象属于哪个类之外，还附加了引用计数extra_rc，是否有被weak引用标志位weakly_referenced，是否有附加对象标志位has_assoc等信息。接下来我们一起来看看它的真面目！
###源码解析
首先，我们回顾一下isa指针是怎么在一个对象中存储的。如下图：
![isa存储.png](https://upload-images.jianshu.io/upload_images/4053175-a736cdd180d9a893.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从图中可以看出，我们所谓的isa指针，最后实际上落脚于isa_t的联合体；
####isa_t分析
```
union isa_t {
    // 两个构造函数
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    // bits 可以当做为向外提供了操作struct的对象
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};
```
从源码可以看出 isa_t是联合体，isa_t 中包含有cls，bits， struct三个变量，它们的内存空间是重叠的。在实际使用时，仅能够使用它们中的一种，你把它当做cls，就不能当bits访问，你把它当bits，就不能用cls来访问！(问：为什么用联合体呢，它的作用是什么？答：联合体的作用是在于，用更少的空间来表示了更多的类型，但是这些类型是不能够共存的！）
对联合体有兴趣的小伙伴空闲时间，可以去了解一下，但不建议用于项目中，装逼请随意！敲敲我这三米长的开山刀，请将注意力集中在isa_t 联合上：
- 有两个构造函数isa_t(), isa_t(uintptr_value)
- 有三个数据成员Class cls, uintptr_t bits, struct ，其中uintptr_t占据64位内存！

这里需要注意下，虽说是三个成员，uintptr_t bits 和 struct 其实是一个成员，它们都占据64位内存空间！因为联合体类型的成员内存空间是重叠的。由于uintptr_t bits 和 struct 都是占据64位内存，因此它们的内存空间是完全重叠的！（简单的来说，uintptr_t bits 可以当做为向外提供了操作struct的对象，而struct 本身则说明了uintptr_t bits 中各个二进制位的定义）

理解了uintptr_t bits 和 struct 关系后，则isa_t其实可以看做有两个可能的取值，Class cls或struct。如下图所示：
![isa_t.png](https://upload-images.jianshu.io/upload_images/4053175-4e48318b8ac09421.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当isa_t作为Class cls使用时，这符合了我们之前一贯的认知：isa是一个指向对象所属Class类型的指针。然而，让一个64位的指针表示一个类型，是不是有些不划算呢。因此，绝大多数情况下，苹果采用了优化的isa策略，isa_t 类型并不等同于Class cls，而是struct！

一起来看一下struct的结构 ：
```
# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
#   define ISA_BITFIELD                                                      \
      uintptr_t nonpointer        : 1;                                       \  注意：区分isa_t是否是一个真正的指针！！！
      uintptr_t has_assoc         : 1;                                       \
      uintptr_t has_cxx_dtor      : 1;                                       \
      uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ \
      uintptr_t magic             : 6;                                       \
      uintptr_t weakly_referenced : 1;                                       \
      uintptr_t deallocating      : 1;                                       \
      uintptr_t has_sidetable_rc  : 1;                                       \
      uintptr_t extra_rc          : 19
#   define RC_ONE   (1ULL<<45)
#   define RC_HALF  (1ULL<<18)

# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
#   define ISA_BITFIELD                                                        \
      uintptr_t nonpointer        : 1;                                         \
      uintptr_t has_assoc         : 1;                                         \
      uintptr_t has_cxx_dtor      : 1;                                         \
      uintptr_t shiftcls          : 44; /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/ \
      uintptr_t magic             : 6;                                         \
      uintptr_t weakly_referenced : 1;                                         \
      uintptr_t deallocating      : 1;                                         \
      uintptr_t has_sidetable_rc  : 1;                                         \
      uintptr_t extra_rc          : 8
#   define RC_ONE   (1ULL<<56)
#   define RC_HALF  (1ULL<<7)

# else
#   error unknown architecture for packed isa
# endif

// SUPPORT_PACKED_ISA
#endif
```

struct共占用64位，从低位到高位依次是nonpointer到extra_rc。成员后面的：表明了该成员占用几个bit。成员的含义如下：
![struct 结构表.png](https://upload-images.jianshu.io/upload_images/4053175-dbd28a30834b3300.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由上表可以看出，和对象引用计数相关的有两个成员：extra_rc和has_sidetable_rc。真机所采用19位的extra_rc来记录对象的引用次数，当extra_rc 不够用时，还会借助sidetable来存储计数值，这时，has_sidetable_rc会被标志为1。我们可以算一下，对于19位的extra_rc ，其数值可以表示2^19 - 1 = 524287。 52万多，相信绝大多数情况下，都够用了。

####实践验证
创建一个对象，并打印isa_t的值
```
PeopleModel *model = [[PeopleModel alloc] init];
NSLog(@"isa_t = %p", *(void **)(__bridge void*)model); // isa_t = 0x1a10286cf39
```
复制该地址粘贴到计算机中：
![计算器.png](https://upload-images.jianshu.io/upload_images/4053175-d3579ffa816ecb9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

标红区域请于上图中struct结构表结合查看！当开启了isa_t优化，nonpointer 置位为1， 这时，isa_t *其实不是一个地址，而是一个有意义的值，也就是说，苹果用isa_t * 所占用的64位空间，表示了一个有意义的值，而这64位值的定义，就符合我们上面struct的定义。
