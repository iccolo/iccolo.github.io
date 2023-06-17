---
layout: post
title: "RoaringBitmap与Protobuf存储空间对比"
author: "iccolo"
tags: ["bitmap", "存储空间"]
---
## RoaringBitmap

给定 [0,x) 中的 N 个整数，然后 RoaringBitmap 的序列化大小（以字节为单位）不应超过此界限：

8 + 9 * (x+65535)/65536 + 2 * N

## Protobuf

使用如下结构存储 bitmap：

```protobuf
message Bitmap {
  repeated uint64 non_zero_bits = 1;
}
```

Protobuf 整数序列化大小：
- 2字节 [0,    2^7)   即 [0, 128)
- 3字节 [2^7,  2^14)  即 [128, 16,384)
- 4字节 [2^14, 2^21)  即 [16,384, 2,097,152)
- 5字节 [2^21, 2^28)  即 [2,097,152, 268,435,456)

## Custom-Encoding

如果采用 Protobuf 方式编码整数列表，但去掉编码头部，每个整数序列化大小减少1字节

## 序列化大小比较

使用 RoaringBitmap/Protobuf 存储 Bitmap 序列化大小与 x、N 有关

#### x < 128
RoaringBitmap 17+2*N Bytes
Protobuf      2*N    Bytes
Custom-Encoding 1*N  Bytes

序列化大小：RoaringBitmap > Protobuf

#### 128 <= x < 16,384
RoaringBitmap 17+2*N Bytes
Protobuf      2*N ~ 3*N Bytes
Custom-Encoding 1*N ~ 2*N Bytes

序列化大小：如果 N < 17 则 RoaringBitmap > Protobuf，否则视具体情况；RoaringBitmap > Custom-Encoding

#### 16,384 <= x <= 65,536
RoaringBitmap 17+2*N Bytes
Protobuf      2*N ~ 4*N Bytes
Custom-Encoding 1*N ~ 3*N Bytes

序列化大小：如果 N < 9 则 RoaringBitmap > Protobuf，否则视具体情况；如果 N < 17 则 RoaringBitmap > Custom-Encoding，否则视具体情况

#### 65,536 < x < 2,097,152
RoaringBitmap 26+2*N Bytes
Protobuf      2*N ~ 5*N Bytes
Custom-Encoding 1*N ~ 4*N Bytes

序列化大小：如果 N < 9 则 RoaringBitmap > Protobuf，否则视具体情况


## 附：Protobuf 整数序列化大小计算

整数编码头部
- 第 1-3bit，存放数据类型
- 第 4-7bit，存放 field tag
- 第 8bit，1表示下一个字节也属于头部，用来存放更大的 field tag 值，0表示头部结束

数据类型
- 0：int32, int64, uint32, uint64, sint32, sint64, bool, enum
- 1：fixed64, sfixed64, double
- 2：string, bytes, embedded messages, packed repeated fields
- 5：fixed32, sfixed32, float

整数编码内容
- 第 8bit，1表示下一个字节也属于当前整数编码，0表示下一个字节是新的内容

例如数字127编码是 0000 1000 0111 1111