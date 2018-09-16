# 缓存Caffeine

[TOC]

## 一、缓存的计数策略Min-Count Sketch

### 0、概述

大家熟知的缓存淘汰方法有LFU（Least Frequently Used） 和LRU（Least Recently Used）

LRU的特点为简洁，但命中率低

LFU主要的缺点有二：一是要维护一份很大的列表来记录频次；二是通常热度会随时间承指数衰减，某一天的热点可能在这几日已经没什么搜索热度了，但是因为昨天的搜索频次很高，导致在很长的一段时间都会淘汰不掉。

Min-Count Sketch则结合了频次和时间

* 其记录频次的一个粗略画像
  * 正常做法为：一个元素对应到一个hash，该hash&tableMask +1
  * 理想情况是hash不发生碰撞；但是如果池子太小，碰撞难以避免会发生
  * Min-Count做法为：一个元素映射到N个hash，每个hash&tableMask + 1，最后计算频次的时候，取这四个hash频次中最小的那个
  * 虽然碰撞仍会发生，但是N个hash都碰撞的可能性相对会变低
* 在计数频次到达一定阈值后，对所有计数频次对半衰减一次



### 1、变量

* 四个SEED（Hash） 		

  * A mixture of seeds from FNV-1a, CityHash, and Murmur3

  * ```java
    SEED = new long[] {
        0xc3a5c85c97cb3127L, 0xb492b66fbe98f273L, 0x9ae16a3b2f90404fL, 0xcbf29ce484222325L
    };
    ```

* long RESET_MASK = 0x7777777777777777L;

* long ONE_MASK = 0x1111111111111111L;

* long[] table;

* int tableMask;

* int sampleSize;

* int size; 


```java
  public void ensureCapacity(@Nonnegative long maximumSize) {
    requireArgument(maximumSize >= 0);
    int maximum = (int) Math.min(maximumSize, Integer.MAX_VALUE >>> 1);
    if ((table != null) && (table.length >= maximum)) {
      return;
    }

    table = new long[(maximum == 0) ? 1 : ceilingNextPowerOfTwo(maximum)]; // 2的幂次方
    tableMask = Math.max(0, table.length - 1); // mask的存在好像就是为了截断用的，hash && mask就能得到index了
    sampleSize = (maximumSize == 0) ? 10 : (10 * maximum); // 当计数达到sampleSize，则要发生对半衰减
    if (sampleSize <= 0) {
      sampleSize = Integer.MAX_VALUE;
    }
    size = 0; // 计数数量
  }
```




### 2、主要结构

#### 1）流程图

![mark](http://ovnu0w2k5.bkt.clouddn.com/blog/180913/ADAF2lfAG8.png?imageslim)

#### 2）理解

一个大格子实际是一个long，8字节，64位，每个大格子是long[] table中的一项

可以存储的频次的最大计数为16 = 2^4，占4位

一个大格子一共可分为 64/4 = 16 个小格子，存16个频次

因为一共有四个seed，所以我们把16个小格子分为4份中格子，每一份中格子中有4个小格子

根据 fun(hash', seed[i]) = index 确定 大格子的位置 table[index]

根据 hash'&3 确定 中格子[0-3]位置

根据 第i个seed的i 确定 小格子[0-3]位置



### 3、计算频次

元素item根据4个seed分别对应到4个小格子，取4个小格子中最小的count



### 4、增加increment

#### 1）计数增加

由上面的分析可以知道，每个小格子其实是一个long的一部分，所以虽然是计数加一，但是对long来说从右到左，每个元素对应分别加 1 = 2^0， 16 = 2^4， 256 = 2^8...

其实就是定位到具体的小格子，然后算出这个小格子的1在long中的对应的数

offset = 4 * position

真要加的数为 accu = 1 << offset

```java
  public void increment(@Nonnull E e) {
    if (isNotInitialized()) {
      return;
    }

    int hash = spread(e.hashCode());
    int start = (hash & 3) << 2;

    // Loop unrolling improves throughput by 5m ops/s
    int index0 = indexOf(hash, 0);
    int index1 = indexOf(hash, 1);
    int index2 = indexOf(hash, 2);
    int index3 = indexOf(hash, 3);

    boolean added = incrementAt(index0, start);
    added |= incrementAt(index1, start + 1);
    added |= incrementAt(index2, start + 2);
    added |= incrementAt(index3, start + 3);

    if (added && (++size == sampleSize)) {
      reset();
    }
  }
```



#### 2）计数判断是否满15

和计数增加类似，定位到具体的小格子，判断这个小格子是否满15

实现为：计算mask = 1111（二进制） << offset

mask的形式为：0000...1111...0000

table[i] & mask != mask 表示没到15

```java
  boolean incrementAt(int i, int j) {
    int offset = j << 2;
    long mask = (0xfL << offset);
    if ((table[i] & mask) != mask) {
      table[i] += (1L << offset);
      return true;
    }
    return false;
  }
```



#### 3）对半缩减

```java
  /** Reduces every counter by half of its original value. */
  void reset() {
    int count = 0;
    for (int i = 0; i < table.length; i++) {
      count += Long.bitCount(table[i] & ONE_MASK); // ONE_MASK = 00010001...0001 累计所有的小格子尾数为1的个数
      table[i] = (table[i] >>> 1) & RESET_MASK; // RESET_MASK = 01110111...0111 这个写法比较巧妙，将小格子里的所有的数都/2
    }
    size = (size >>> 1) - (count >>> 2); // 因为整除会在/2时导致一些计算误差，因为四个小格子才是真的一个数，所以count/4被记做多抹去的一个数
  }
```

