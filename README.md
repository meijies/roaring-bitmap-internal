
# 1. 前言

Roaring Bitmap 是一个高性能的压缩 Bitmap 实现，与传统的基于 RLE (Run Length 编码) 的 Bitmap 实现 (WAH、Concise、EWAH) 相比，在计算性能和压缩效率上都有成倍的提升，并被广泛应用于许多生产平台，有很大的学习研究价值。

然而如果直接看论文或代码的话，可能会像我一样遇到很多困难，特别是在理解向量执行(SIMD)算法的时候。因此，这篇文章最主要的目标就是由浅入深的帮助大家理解 Roaring Bitmap 的设计与实现。不过在此之前，还是有必要对 bitmap 有一个基本的了解。

Bitmap 是一种使用 N 个 bit 表示在给定 [0, N) 区间，如果存在第 i 个元素，则第 i 个 bit 位为 1 的数据结构。例如当 N = 10 的时候，在给定 [0, 10) 的区间，存在 1, 3, 5, 6, 7, 8, 9 七个元素，那么对应的 Bitmap 为: 0101011111。相比于使用有序数组 [1, 3, 5, 6, 7, 8, 9]，Bitmap 使用一个 bit 位就能表示 32 位整数是否存在，这将有效节省内存并且大大提高运算效率。然而当 bitmap 中存在的元素比较稀疏，特别是当其数量 S < N/32 的时候，未经压缩的 bitmap 将不再有此优势。

# 2. 总体设计
![overall_architecture](overall_architecture.png "总体设计")

暂时无法在飞书文档外展示此内容
Roaring Bitmap 最基本的设计思路是将一个 32 位无符号整型拆分成高 16 位和低 16 位的 key、value 对，相同高 16 位 key 的 value 将存储到同一个 value 容器，每个 value 容器最多存储 2 的 16 次方个元素，共享相同的高 16 位 key，使用 value 容器存储，而 key 则使用 16 bit 的无符号整型的有序数组存储。

Roaring Bitmap 支持三种 value 容器，每个容器使用的内存不超过 8 kB (大B，表示 byte)，能够全部放入一级缓存(i7-6700 的 L1 数据缓存为 128 kB)。其中数组容器是元素低 16 位组成的有序数组，最多存储 4096 个元素。当需要存储的元素数量超过 4096 的时候，数组容器将会转换成 bitmap 容器。此外数组容器和 bitmap 容器都可以转换成 run 容器，它使用 run length 编码来减少内存的使用。

Bitmap 需要支持访问操作和逻辑运算。访问操作作用于单个元素，比如增加或者删除一个元素；逻辑运算则通常作用于整个 bitmap，比如求交集，并集，差集等。由于 Roaring Bitmap 支持三种不同的 Value 容器，不同的 Value 容器之间进行逻辑运算需要使用不同的算法，这无疑增加了设计的复杂度。

# 3. Value 容器

## 3.1 数组容器

## 3.2 Bitmap 容器

## 3.3 Run 容器

## 3.4 容器的互相转换

# 4. 向量执行基础

# 5. 访问操作

# 6. 逻辑计算

``` 
input: an integer w 
output: an array S containing the indexes where a 1-bit can be found in w 
Let S be an initially empty list 
while w != 0 do 
  t ← w AND − w # 保留最右侧为 1 的 bit 位，其他 bit 位置 0
  append bitCount(t − 1) to S # t-1，
  w ← w AND (w − 1) # 将最右侧为 1 的 bit 位置 0
return S
```

``` c
pos = 0 
while (w != 0) {
 uint64_t temp = _blsi_u64(w); // 等价于 w AND -w
 out[pos++] = _mm_tzcnti_64(w); // 等价于 bitcout(t - 1)
 w ˆ= temp; 
}
uint64_t old_w = words[pos >> 6]; // 定位到 64 位 integer
uint64_t new_w = old_w | (UINT64_C(1) << (pos & 63)); // 设置比特位
// 是否增加了数据，更新基数，这里使用了无分支的算法
 cardinality += (old_w ˆ new_w) >> (pos & 63);
 words[pos >> 6] = new_w;
 ```

```Java
public static int bitCount(int i) {
    // HD, Figure 5-2
    i = i - ((i >>> 1) & 0x55555555);
    i = (i & 0x33333333) + ((i >>> 2) & 0x33333333);
    i = (i + (i >>> 4)) & 0x0f0f0f0f;
    i = i + (i >>> 8);
    i = i + (i >>> 16);
    return i & 0x3f;
}
```

``` java
public static int bitCount(int i) {
    i = (i & 0x55555555) + ((i >>> 1) & 0x55555555);
    i = (i & 0x33333333) + ((i >>> 2) & 0x33333333);
    i = (i & 0x0f0f0f0f) + ((i >>> 4) & 0x0f0f0f0f);
    i = (i & 0x00ff00ff) + ((i >>> 8) & 0x00ff00ff);
    i = (i & 0x0000ffff) + ((i >>> 16) & 0x0000ffff);
    return i;
}
```

```c
int32_t intersect(uint16_t *A, size_t lengthA , 
                  uint16_t *B, size_t lengthB , 
                  uint16_t *out) { 
  size_t count = 0; // size of intersection 
  size_t i = 0, j = 0;
  int vectorlength = sizeof(__m128i) / sizeof(uint16_t);
  __m128i vA, vB , rv , sm16; 
  while (i < lengthA) && (j < lengthB) {
    // 从数组 A，同时加载 8 个 16 位数字到向量 vA
    vA = _mm_loadu_si128((__m128i *)&A[i]);
    // 从数组 B，同时加载 8 个 16 位数字到向量 vB
    vB = _mm_loadu_si128((__m128i *)&B[j]); 
    // 逐对比较 16 位数字，并将比较的结果通过位掩码的方式返回。
    // 假设：
    //     A[8] = {1, 15, 24, 32, 74, 86, 96, 106}
    //     B[8] = {2, 15, 24, 33, 75, 86, 96, 106}
    // 则得到 bitmask：
    //     rv = 0b11100110, 最左侧的 1 表示 A，B 最右侧的数字对匹配
    // 之后基于 rv 得到 shuffle control mask：
    //     由于使用了 _mm_shuffle_epi8，一个小标表示 8 位，表示一个整数需要两个下标
    //     [02,03] [04,05] [10,11] [12,13] [14,15]
    //     15的下标 24的下标
    rv = _mm_cmpistrm(vB , vA ,    
       _SIDD_UWORD_OPS // 无符号 16 位字符
      | _SIDD_CMP_EQUAL_ANY // 
      | _SIDD_BIT_MASK); // 只返回位掩码
    // 抽取第一个 32 位数字
    int r = _mm_extract_epi32(rv , 0);
    // mask16 是一个搜索表，存放 cmpistrm 指令得到的 bitmask 到 shuffle control mask
    // 的一个映射，使用一维数组存，通过 (const __m128i *)mask16 + r 来取得对应的
    // shuffle controll mask
    sm16 = _mm_load_si128(mask16 + r); 
    // 基于 02,03,04,05,10,11,12,13,14,15, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
    // 的 shuffle control mask 提取出交集的结果，15,24,86,96,106,0,0,0，
    __m128i p = _mm_shuffle_epi8(vA, sm16); 
    _mm_storeu_si128((__m128i *)(out + count),p); 
    count += _mm_popcnt_u32(r); 
    uint16_t a_max = A[i + vectorlength - 1]; 
    uint16_t b_max = B[j + vectorlength - 1];
    if (a_max <= b_max) i += vectorlength; 
    if (b_max <= a_max) j += vectorlength; 
  } 
  return count;
}
```

# 7. 参考文档