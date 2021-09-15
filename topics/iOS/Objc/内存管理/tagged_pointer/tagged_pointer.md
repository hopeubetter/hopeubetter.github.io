Apple 在 iPhone5s 推出64位系统时，引入了 `Tagged Pointer` 技术。

##### 引入原因？

![](https://raw.githubusercontent.com/hopeubetter/hopeubetter.github.io/master/topics/iOS/Objc/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/tagged_pointer/tagged_pointer_1.png)

![](https://raw.githubusercontent.com/hopeubetter/hopeubetter.github.io/master/topics/iOS/Objc/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/tagged_pointer/tagged_pointer_2.png)

引入 `Tagged Pointer` ，给64位系统，节省内存、提高运行效率（读取有3倍效率，创建比以前快106倍）。


##### `Tagged Pointer` 的特点？

只有64位机支持。

专门用来存储 `NSString`、 `NSNumber`、 `NSDate`等小对象。

`Tagged Pointer`指针的值不再是对象的地址了，而是对象的值。所以它不再是对象了（不存储 isa 指针），只是一个披着对象皮的普通变量而已。

指针 64位，分为两部分，一部分标记位，一部分为数据位。

MBS（高位优先 iOS采用）：

系统自带的 `Tagged Pointer` 类型，左侧第一位为 `Tagged Pointer` 标记位 - `1` （如果是`0`为普通对象类型）。左侧2-4位为类类型位，剩下的60位用来存储数据。

自定义扩展的 `Tagged Pointer` 类型，左侧第一位为 `Tagged Pointer` 标记位 - `1` （如果是`0`为普通对象类型）。左侧2-4位都为1，左侧5-12位为扩展类类型位，剩下的52位用来存储数据。

LBS （低位优先 MacOS采用）：跟 MBS 正好相反，从右侧开始排。


```text
/***********************************************************************
* Tagged pointer objects.
*
* Tagged pointer objects store the class and the object value in the
* object pointer; the "pointer" does not actually point to anything.
*
* Tagged pointer objects currently use this representation:
* (LSB)
*  1 bit   set if tagged, clear if ordinary object pointer
*  3 bits  tag index
* 60 bits  payload
* (MSB)
* The tag index defines the object's class.
* The payload format is defined by the object's class.
*
* If the tag index is 0b111, the tagged pointer object uses an
* "extended" representation, allowing more classes but with smaller payloads:
* (LSB)
*  1 bit   set if tagged, clear if ordinary object pointer
*  3 bits  0b111
*  8 bits  extended tag index
* 52 bits  payload
* (MSB)
*
* Some architectures reverse the MSB and LSB in these representations.
*
* This representation is subject to change. Representation-agnostic SPI is:
* objc-internal.h for class implementers.
* objc-gdb.h for debuggers.
**********************************************************************/

```

##### `Tagged Pointer` 不存储 `isa` 是怎么向他发送消息（调用方法）的？

`Tagged Pointer` 不存储 `isa`，而是存储类类型索引（系统自带类型，为最左4位，自定义扩展的为最左12位），objc_msgSend 函数会自动进行判断，根据第一位（最左）是否为1来判断是 `Tagged Pointer` 对象还是普通对象，普通对象直接通过对象 isa 获取类对象，`Tagged Pointer`对象根据索引到指定的数组中（两个数组，一个是系统自带类的，一个是自定义扩展的）获取类对象。 

##### 数值直接保存在指针中，怎么保证安全？

采用数据混淆，将 `Tagged Pointer` 值混淆称普通地址。可以设置环境变量 `OBJC_DISABLE_TAG_OBFUSCATION` 为 YES 来关闭数据混淆。

##### 怎么关闭 `Tagged Pointer`？

可以通过设置环境变量 `OBJC_DISABLE_TAGGED_POINTERS` 为 YES 来关闭。

##### `Tagged Pointer` 的源码在objc/runtime/objc-internal.h中，[下载地址](https://opensource.apple.com/tarballs/objc4/)

##### `Tagged Pointer` 相应的函数


```c
// 判断是否为 Tagged Pointer 指针

static inline bool
_objc_isTaggedPointer(const void * _Nullable ptr) 
{
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}

```


```c
// 创建 Tagged Pointer 指针

static inline void * _Nonnull
_objc_makeTaggedPointer(objc_tag_index_t tag, uintptr_t value)
{
    if (tag <= OBJC_TAG_Last60BitPayload) {
        return (void *)
            (_OBJC_TAG_MASK | 
             ((uintptr_t)tag << _OBJC_TAG_INDEX_SHIFT) | 
             ((value << _OBJC_TAG_PAYLOAD_RSHIFT) >> _OBJC_TAG_PAYLOAD_LSHIFT));
    } else {
        return (void *)
            (_OBJC_TAG_EXT_MASK |
             ((uintptr_t)(tag - OBJC_TAG_First52BitPayload) << _OBJC_TAG_EXT_INDEX_SHIFT) |
             ((value << _OBJC_TAG_EXT_PAYLOAD_RSHIFT) >> _OBJC_TAG_EXT_PAYLOAD_LSHIFT));
    }
}

```


```c
// 从 Tagged Pointer 指针中取出值

static inline uintptr_t
_objc_getTaggedPointerValue(const void * _Nullable ptr) 
{
    // assert(_objc_isTaggedPointer(ptr));
    uintptr_t basicTag = ((uintptr_t)ptr >> _OBJC_TAG_INDEX_SHIFT) & _OBJC_TAG_INDEX_MASK;
    if (basicTag == _OBJC_TAG_INDEX_MASK) {
        return ((uintptr_t)ptr << _OBJC_TAG_EXT_PAYLOAD_LSHIFT) >> _OBJC_TAG_EXT_PAYLOAD_RSHIFT;
    } else {
        return ((uintptr_t)ptr << _OBJC_TAG_PAYLOAD_LSHIFT) >> _OBJC_TAG_PAYLOAD_RSHIFT;
    }
}

```


```c
// 为了安全需要对 Tagged Pointer 指针进行数据混淆（编码）

///TaggedPointer编码函数
static inline void * _Nonnull
_objc_encodeTaggedPointer(uintptr_t ptr)
{
    return (void *)(objc_debug_taggedpointer_obfuscator ^ ptr);
}
///TaggedPointer解码函数
static inline uintptr_t
_objc_decodeTaggedPointer(const void * _Nullable ptr)
{
    return (uintptr_t)ptr ^ objc_debug_taggedpointer_obfuscator;
}

```


```c
// 注册 Tagged Pointer 类型，这就解释了为什么有的类支持 Tagged Pointer 指针，有的不支持

/***********************************************************************
* _objc_registerTaggedPointerClass
* Set the class to use for the given tagged pointer index.
* Aborts if the tag is out of range, or if the tag is already 
* used by some other class.
**********************************************************************/
void
_objc_registerTaggedPointerClass(objc_tag_index_t tag, Class cls)
{
     //OBJC_DISABLE_TAGGED_POINTERS就是判断开发者有没有关闭Tagged Pointer
    if (objc_debug_taggedpointer_mask == 0) {
        _objc_fatal("tagged pointers are disabled");
    }
    Class *slot = classSlotForTagIndex(tag);
    if (!slot) {
        _objc_fatal("tag index %u is invalid", (unsigned int)tag);
    }
    Class oldCls = *slot;
    if (cls  &&  oldCls  &&  cls != oldCls) {
        _objc_fatal("tag index %u used for two different classes "
                    "(was %p %s, now %p %s)", tag, 
                    oldCls, oldCls->nameForLogging(), 
                    cls, cls->nameForLogging());
    }
    *slot = cls;
    // Store a placeholder class in the basic tag slot that is 
    // reserved for the extended tag space, if it isn't set already.
    // Do this lazily when the first extended tag is registered so 
    // that old debuggers characterize bogus pointers correctly more often.
    if (tag < OBJC_TAG_First60BitPayload || tag > OBJC_TAG_Last60BitPayload) {
        Class *extSlot = classSlotForBasicTagIndex(OBJC_TAG_RESERVED_7);
        if (*extSlot == nil) {
            extern objc_class OBJC_CLASS_$___NSUnrecognizedTaggedPointer;
            *extSlot = (Class)&OBJC_CLASS_$___NSUnrecognizedTaggedPointer;
        }
    }
}

```

```c
// 根据索引获取类对象

// Returns a pointer to the class's storage in the tagged class arrays, 
// or nil if the tag is out of range.
static Class *
classSlotForTagIndex(objc_tag_index_t tag)
{
    if (tag >= OBJC_TAG_First60BitPayload && tag <= OBJC_TAG_Last60BitPayload) {
        return classSlotForBasicTagIndex(tag);
    }
    if (tag >= OBJC_TAG_First52BitPayload && tag <= OBJC_TAG_Last52BitPayload) {
        int index = tag - OBJC_TAG_First52BitPayload;
        uintptr_t tagObfuscator = ((objc_debug_taggedpointer_obfuscator
                                    >> _OBJC_TAG_EXT_INDEX_SHIFT)
                                   & _OBJC_TAG_EXT_INDEX_MASK);
        return &objc_tag_ext_classes[index ^ tagObfuscator];
    }
    return nil;
}

// Returns a pointer to the class's storage in the tagged class arrays.
// Assumes the tag is a valid basic tag.
static Class *
classSlotForBasicTagIndex(objc_tag_index_t tag)
{
    uintptr_t tagObfuscator = ((objc_debug_taggedpointer_obfuscator
                                >> _OBJC_TAG_INDEX_SHIFT)
                               & _OBJC_TAG_INDEX_MASK);
    uintptr_t obfuscatedTag = tag ^ tagObfuscator;
    // Array index in objc_tag_classes includes the tagged bit itself
#if SUPPORT_MSB_TAGGED_POINTERS //高位优先
    return &objc_tag_classes[0x8 | obfuscatedTag];
#else
    return &objc_tag_classes[(obfuscatedTag << 1) | 1];
#endif

}

```













