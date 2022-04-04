---
title:      "LevelDB代码阅读：Varint"
date:       "2017-11-15 14:00:00"
tags: ["LevelDB"]
---

C++一直是我想要学习的编程语言之一，但是拖延症，使我始终都没有学个明白。所以借LevelDb代码阅读之际，复习一下，随带学习一下KV数据库

## Varint介绍
Varint是Leveldb中的一种表示数字的方法，他用一个或多个字节表示一个数字，值越少的数字，所占用的字节数越少。比如对于int32类型的数字，一般需要4个byte来表示。但是采用Varint，对于很小的int32类型的数字，则可以用1个byte来表示。当然凡事都有好的也有不好的一面，采用Varint表示法，大的数字则需要5个byte来表示。从统计的角度来说，一般不会所有的消息中的数字都是大数，因此大多数情况下，采用Varint 后，可以用更少的字节数来表示数字信息。

Varint每个byte的最高位bit有特殊含义，如果该位为1，表示后续的byte也是该数字的一部分，如果该位为0，则结束。其他的7 个bit都用来表示数字。因此小于128的数字都可以用一个byte表示。大于 128的数字，比如300，会用两个字节来表示：1010 1100 0000 0010。

## Varint源代码分析
Varint的源代码位于LevelDB的[util/coding.cc，util/coding.h]()内


我们来看一下[util/coding.cc](https://github.com/google/leveldb/blob/master/util/coding.cc),
正常情况下，int需要32位，varint用一个字节的最高为做标识位，所以，一个字节只能存储7位，如果整数特别大，可能需要5个字节才能存放{5 * 8 - 5(标识位) > 32}，下面的if语句有5个分支，正好对应varint占用1到5个字节的情况。
```c++
char* EncodeVarint32(char* dst, uint32_t v) {
  // Operate on characters as unsigneds
  unsigned char* ptr = reinterpret_cast<unsigned char*>(dst);
  static const int B = 128;
  if (v < (1<<7)) {
    *(ptr++) = v;
  } else if (v < (1<<14)) {
    *(ptr++) = v | B;
    *(ptr++) = v>>7;
  } else if (v < (1<<21)) {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = v>>14;
  } else if (v < (1<<28)) {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = v>>21;
  } else {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = (v>>21) | B;
    *(ptr++) = v>>28;
  }
  return reinterpret_cast<char*>(ptr);
}
```

看了编码，我们再来看看解码[util/coding.h](https://github.com/google/leveldb/blob/master/util/coding.h)

```c++
inline const char* GetVarint32Ptr(const char* p,
                                  const char* limit,
                                  uint32_t* value) {
  if (p < limit) {
    uint32_t result = *(reinterpret_cast<const unsigned char*>(p));
    if ((result & 128) == 0) { 
      *value = result;
      return p + 1;
    }
  }
  return GetVarint32PtrFallback(p, limit, value);
}
```

这里的limit是从GetVarint32传入，看下面的代码，Slice是LevelDB的自定义string，类似于redis的SDS。首先得到Slice数据的起始指针，再用`p + input->size()`来获得结束指针。value用于存储返回的int值。

如果`result & 128 `等于0，则直接解码。128对应二进制为 1000 0000，只有最高位为1，对应 varint的字节中最高特殊位，如果特殊位为0，代表该字节直接可以表示给数字。

```
bool GetVarint32(Slice* input, uint32_t* value) {
  const char* p = input->data();
  const char* limit = p + input->size();
const char* q = GetVarint32Ptr(p, limit, value);
```

否者调用GetVarint32PtrFallback获得uint32

```c++
const char* GetVarint32PtrFallback(const char* p,
                                   const char* limit,
                                   uint32_t* value) {
  uint32_t result = 0;
  for (uint32_t shift = 0; shift <= 28 && p < limit; shift += 7) {
    uint32_t byte = *(reinterpret_cast<const unsigned char*>(p));
    p++;  //p指针向前移动
    if (byte & 128) {
      // More bytes are present
      result |= ((byte & 127) << shift);  //取7位到result
    } else {
      result |= (byte << shift); //获得最后一位
      *value = result;
      return reinterpret_cast<const char*>(p);
    }
  }
  return NULL;
}
```
现在我们再来看看Varint64的编译和解析，在原理上和Varint如出一辙。但是编译上和32位略有不同，而是使用了一个循环来解决

```c++
char* EncodeVarint64(char* dst, uint64_t v) {
  static const int B = 128;
  unsigned char* ptr = reinterpret_cast<unsigned char*>(dst);
  while (v >= B) {
    *(ptr++) = (v & (B-1)) | B;
    v >>= 7;
  }
  *(ptr++) = static_cast<unsigned char>(v); 
  return reinterpret_cast<char*>(ptr);
}
```

解析Varint64，也和Varint32如出一则

```c++
const char* GetVarint64Ptr(const char* p, const char* limit, uint64_t* value) {
  uint64_t result = 0;
  for (uint32_t shift = 0; shift <= 63 && p < limit; shift += 7) {
    uint64_t byte = *(reinterpret_cast<const unsigned char*>(p));
    p++;
    if (byte & 128) {
      // More bytes are present
      result |= ((byte & 127) << shift);
    } else {
      result |= (byte << shift);
      *value = result;
      return reinterpret_cast<const char*>(p);
    }
  }
  return NULL;
}
```

## reinterpret_cast有何作用
reinterpret_cast用在任意指针（或引用）类型之间的转换；以及指针与足够大的整数类型之间的转换；从整数类型（包括枚举类型）到指针类型，无视大小。但是这种随意的转换必然会带来需要的问题，这个的讨论在之后的Blog，再来论述。


## 参考文章

- [LevelDB源码剖析之Varint](http://mingxinglai.com/cn/2013/01/leveldb-varint32/)
- [C++标准转换运算符reinterpret_cast](https://www.cnblogs.com/ider/archive/2011/07/30/cpp_cast_operator_part3.html)