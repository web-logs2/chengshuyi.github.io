---
title: "redis源码分析五：Hyperloglog"
date: 2020-08-12T16:35:57+08:00
description: ""
draft: false
tags: [redis源码分析]
categories: [redis源码分析]
---



Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定的、并且是很小的。在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 $2^64$ 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

接下来我们将从两个方面来分析hyperloglog，一个是hyperloglog的添加数据，另外一个是hyperloglog的合并过程。

### hll添加数据

具体见注释：

```c
int hllAdd(robj *o, unsigned char *ele, size_t elesize) {
    struct hllhdr *hdr = o->ptr;									// 1. hll头部信息
    switch(hdr->encoding) {
    case HLL_DENSE: return hllDenseAdd(hdr->registers,ele,elesize); // 2. 密集模式添加元素
    case HLL_SPARSE: return hllSparseAdd(o,ele,elesize); 			// 3. 稀疏模式添加元素
    default: return -1; /* Invalid representation. */ 		
    }
}

int hllDenseAdd(uint8_t *registers, unsigned char *ele, size_t elesize) {
    long index;
    uint8_t count = hllPatLen(ele,elesize,&index);					// 4. 计算该元素第一个1出现的位置
    return hllDenseSet(registers,index,count);
}

int hllPatLen(unsigned char *ele, size_t elesize, long *regp) {
    uint64_t hash, bit, index;
    int count;
    
    hash = MurmurHash64A(ele,elesize,0xadc83b19ULL);				// 5. 利用MurmurHash64A哈希函数来计算该元素的hash值
    index = hash & HLL_P_MASK;										// 6. 取低14位作为桶的编号
    hash >>= HLL_P;
    
    hash |= ((uint64_t)1<<HLL_Q); 									// 7. 为了保证循环能够终止，将第50（从0计数）位置1
    bit = 1;
    count = 1;														// 8. 存储第一个1出现的位置
    while((hash & bit) == 0) {
        count++;
        bit <<= 1;
    }
    *regp = (int) index;
    return count;
}

int hllDenseSet(uint8_t *registers, long index, uint8_t count) {
    uint8_t oldcount;
    HLL_DENSE_GET_REGISTER(oldcount,registers,index);				// 9. 得到第index个桶内的count值，因为hll每个桶的大小是6bit，所以需要采用特别的方法获取；
    if (count > oldcount) {
        HLL_DENSE_SET_REGISTER(registers,index,count);				// 10. 如果比现有的值还大，则替换；也就是每个桶内统计的是最大值
        return 1;
    } else {
        return 0;
    }
}
```

### hll统计结果

下图是hll统计的公式，其中：

* m表示分成多少个桶，在redis里低14位作为桶的编号，也就是$2^14$；
* const表示修正因子，一个常量；
* $R_j$表示桶j统计的数值（也就是高50位第一次出现1的位置）；

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200815203758.png)



```c
uint64_t hllCount(struct hllhdr *hdr, int *invalid) {
    double m = HLL_REGISTERS;
    double E;
    int j;
    int reghisto[64] = {0}; 												// 1. 实际使用到的只有[0,51]

    /* Compute register histogram */
    if (hdr->encoding == HLL_DENSE) {
        hllDenseRegHisto(hdr->registers,reghisto);                          // 2. 密集存储的hll统计，统计频次，也就是桶内值为x的桶的个数；
    } else if (hdr->encoding == HLL_SPARSE) {
        hllSparseRegHisto(hdr->registers,
                         sdslen((sds)hdr)-HLL_HDR_SIZE,invalid,reghisto);
    } else if (hdr->encoding == HLL_RAW) {
        hllRawRegHisto(hdr->registers,reghisto);
    } else {
        serverPanic("Unknown HyperLogLog encoding in hllCount()");
    }
    double z = m * hllTau((m-reghisto[HLL_Q+1])/(double)m);                // 5. 特殊处理hash值高50位全为0的情况
    for (j = HLL_Q; j >= 1; --j) {                                         // 6. 计算调和级数分母部分
        z += reghisto[j];
        z *= 0.5;
    }
    z += m * hllSigma(reghisto[0]/(double)m);                              // 7. 特殊处理空挡，也就是桶内数值为0
    E = llroundl(HLL_ALPHA_INF*m*m/z);                                     // 8. 上图的公式
    return (uint64_t) E;
}

void hllDenseRegHisto(uint8_t *registers, int* reghisto) {
    int j;
    if (HLL_REGISTERS == 16384 && HLL_BITS == 6) {
        uint8_t *r = registers;
        unsigned long r0, r1, r2, r3, r4, r5, r6, r7, r8, r9,
                      r10, r11, r12, r13, r14, r15;
        for (j = 0; j < 1024; j++) {                            			// 3. 循环1024次，每次计算16个桶，16*6=96也就是12字节
            r0 = r[0] & 63;
            r1 = (r[0] >> 6 | r[1] << 2) & 63;
            r2 = (r[1] >> 4 | r[2] << 4) & 63;
            r3 = (r[2] >> 2) & 63;
            r4 = r[3] & 63;
            r5 = (r[3] >> 6 | r[4] << 2) & 63;
            r6 = (r[4] >> 4 | r[5] << 4) & 63;
            r7 = (r[5] >> 2) & 63;
            r8 = r[6] & 63;
            r9 = (r[6] >> 6 | r[7] << 2) & 63;
            r10 = (r[7] >> 4 | r[8] << 4) & 63;
            r11 = (r[8] >> 2) & 63;
            r12 = r[9] & 63;
            r13 = (r[9] >> 6 | r[10] << 2) & 63;
            r14 = (r[10] >> 4 | r[11] << 4) & 63;
            r15 = (r[11] >> 2) & 63;
            reghisto[r0]++;													// 4. 频次增加
            reghisto[r1]++;
            reghisto[r2]++;
            reghisto[r3]++;
            reghisto[r4]++;
            reghisto[r5]++;
            reghisto[r6]++;
            reghisto[r7]++;
            reghisto[r8]++;
            reghisto[r9]++;
            reghisto[r10]++;
            reghisto[r11]++;
            reghisto[r12]++;
            reghisto[r13]++;
            reghisto[r14]++;
            reghisto[r15]++;
            r += 12;
        }
    } else {
        for(j = 0; j < HLL_REGISTERS; j++) {
            unsigned long reg;
            HLL_DENSE_GET_REGISTER(reg,registers,j);
            reghisto[reg]++;
        }
    }
}

```






### 源码文件

1. `src/hyperloglog.c`

