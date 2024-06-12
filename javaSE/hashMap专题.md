## hashMap专题

### hashMap的底层数据结构

jdk8，数组+链表+红黑树

其中：

- 当数组超过加载因子时进行1.5倍扩容
- 当链表超过 8 且数据总量超过 64 时会转红黑树。

![image-20240520120034920](D:\picGo\images\image-20240520120034920.png)

### 为什么链表改为红黑树的阈值是 8?

作者在源码中的注释

> Because TreeNodes are about twice the size of regular nodes, we
> use them only when bins contain enough nodes to warrant use
> (see TREEIFY_THRESHOLD). And when they become too small (due to
> removal or resizing) they are converted back to plain bins. In
> usages with well-distributed user hashCodes, tree bins are
> rarely used. Ideally, under random hashCodes, the frequency of
> nodes in bins follows a Poisson distribution
> ([http://en.wikipedia.org/wiki/Poisson_distributionopen in new window](http://en.wikipedia.org/wiki/Poisson_distribution)) with a
> parameter of about 0.5 on average for the default resizing
> threshold of 0.75, although with a large variance because of
> resizing granularity. Ignoring variance, the expected
> occurrences of list size k are (exp(-0.5) pow(0.5, k) /
> factorial(k)). The first values are:
> 0: 0.60653066
>
> 1: 0.30326533
>
> 2: 0.07581633
>
> 3: 0.01263606
>
> 4: 0.00157952
>
> 5: 0.00015795
>
> 6: 0.00001316
>
> 7: 0.00000094
>
> 8: 0.00000006
>
> more: less than 1 in ten million

理想情况下**使用随机的哈希码**，容器中节点分布在 hash 桶中的频率遵循**泊松分布**

可以得出桶中元素个数和概率的对照表，当链表元素个数为 8 时的概率已经非常小

### 解决hash冲突的办法有哪些？HashMap用的哪种？

- 再散列法(开放地址法)
  - 比如P=H(key)冲突，则让P1=H(P)，以此类推，直到找到一个不冲突的哈希地址
  - 不能真正删除一个元素，只能标记
- 再哈希法
  - 准备多个hash函数，发生冲突就用另一个
  - 不会堆积，但增加了计算的时间
- 链地址法
  - 将哈希值相同的元素构成一个同义词的单链表，并将单链表的头指针存放在哈希表的第i个单元中

### HashMap 的扩容方式？

https://mp.weixin.qq.com/s/0KSpdBJMfXSVH63XadVdmw

### HashMap默认加载因子是多少？为什么是 0.75，不是 0.6 或者 0.8 ？

对于加载因子，= hash桶中存放的元素 / hash桶的容量

- **若过高，空间利用率高，但冲突几率变大**
- **若过低，冲突几率低，但空间利用率不高，动不动就扩容**

这个值在jdk8中是0.75，这个因子是通过**二项分布**来的

设有n次事件，s为桶的容量，我们去计算这个n个事件在存入hash桶时不发生冲突的概率为binom(n，0)

则![image-20240520121128273](D:\picGo\images\image-20240520121128273.png)

我们让这个概率必须>0.5，这样是我们可以接受的

则![image-20240520121559811](D:\picGo\images\image-20240520121559811.png)

对其对数处理，运算可得

![image-20240520121633278](D:\picGo\images\image-20240520121633278.png)

然后求这个分母的极限，得出为ln2/1=0.693.

又因为hashMap容量必须为2的n次幂，也就是说 **容量*加载因子=2的n次幂 **(公式a)

> 为什么？=> 
>
> 因为在计算该元素在数组的下标时，hash 要去 % length，即 & (length -1)。
>
> 为了更“散“地分布，我们希望结果和hash相关度更强一点。
>
> 而 a & b，若b都为1，结果就只和a相关了 
>
> https://mp.weixin.qq.com/s?__biz=MzIxNzQwNjM3NA==&mid=2247498088&idx=2&sn=7bcc4674e9054736aa1aff973d8dc511&scene=21#wechat_redirect

所以取0.75

而加载因子是0.6和0.8，带入公式a计算后，不是2的n次幂了，所以都不可以

但0.625(5/8)和0.875(7/8)都可以，然后0.75是中位数，就取它吧！

> 超过0.8，查表时的CPU缓存不命中（cache missing）会按照指数曲线上升。

### HashMap 中 key 的存储索引是怎么计算的？

```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**计算出key的hashCode为key1，key1进行右移16位为key2**

**hash(key) = key1 ⨁ key2** 

然后回到数组，通过hash&（length-1）计算得到该元素在数组中存储的位置。

### JDK 8 为什么要 hashcode 异或其右移十六位的值？

首先我们来回顾一下hashMap如何最大程度地，权衡利弊地避免hash冲突，插入数组的

> 首先对该key进行自定义hash， 也就是 key.hashCode()  ^  ( h >>> 16 ) 
>
> 然后对数组长度取模，确定要插入元素的数组下标，也就是hash( key )  &  ( length  -  1 )

![image-20240520125916171](D:\picGo\images\image-20240520125916171.png)

一句话：**保证考虑到高低Bit都参与到Hash的计算中**



举个例子：

> 如果hash=h，而不是hash=h ^ (h>>>16)，我们看看会发生什么
>
> 比如现在有一个对key进行hashCode得到 h 为 `1111 1111 1111 1111 1111 0000 1110 1010`
>
> 现在我们直接用 h & ( n - 1)，假设数组长度16，那么 n - 1 的二进制为 `1111`
>
> ok，相与一下。
>
> 你会发现根本和hashCode的高位没有任何关系。如果 h 计算为  `1111 1111 0000 1111 1111 0000 1110 1010`，相与结果还是一样的。

为什么是16位，不是8位，12位？

> hash是一个int类型，
>
> int类型是4个字节，32b，32位二进制
>
> 将key.hashCode()与它的右移16位进行 ⨁ ，高位也考虑到了，对半分嘛



&和%的关系，模运算的消耗很大，没有位运算快（+-  ≈ &^  >  */  > %） 

> 为什么hashMap容量必须为2的n次幂 => 数组长度为2的n次幂，二进制为10x0，-1刚好为0111xxx1，这样hash对数组长度的取模才有意义。
>
> (length-1) 的最后一位可能为 0，也可能为 1（这取决于 h 的值，即key.HashCode()的值）
>
> 即 & 运算后的结果可能为偶数，也可能为奇数，这样便可以保证哈希值的均匀性，降低冲突发生的频率

https://zhuanlan.zhihu.com/p/121555725

### HashMap 的put方法流程？

大致流程：

- 根据hash(key) = n - 1来确定即将插入的数组下标
- 如果数组为null
  - resize去初始化
- 是否发生hash冲突
  - 没有，直接插入数组
  - 发生冲突，
    - 发现冲突的是treeNode，挂树上
    - 发现冲突的是链表，检查链表长度>8(判断1)和当前数组容量>64(判断2)，如果1，扩容；如果12，转红黑树，挂树上；否则挂链表上，将key对应的value更新

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
     // 步骤1：tab为空则创建
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
     // 步骤2：计算index，并对null做处理 
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            // 步骤3：节点key存在，直接覆盖value
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 步骤4：判断该链为红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            // 步骤5：该链为链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //链表长度大于8转换为红黑树进行处理
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // key已经存在直接覆盖value
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
     // 步骤6：超过最大容量 就扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

https://www.cnblogs.com/captainad/p/10905184.html

### HashMap为什么线程不安全？



- **JDK 7 时多线程下扩容会造成死循环，可能会死循环**
  - JDK1.7 使用**头插法**插入元素，在多线程的环境下有可能导致**环形链表**的出现，扩容的时候会导致死循环。

- **多线程的put可能导致元素的丢失。**

  - ```java
    if ((p = tab[i = (n - 1) & hash]) == null) // 如果没有hash碰撞则直接插入元素
    		tab[i] = newNode(hash, key, value, null);
    ```

  - 注意这段代码。如果没有 hash 碰撞则会直接插入元素

  - 如果线程 A 和线程 B 同时进行 put 操作，刚好这两条不同的数据 hash 值一样，并且该位置数据为 null，所以这线程 A、B 都会进入tab[i] = newNode(hash, key, value, null);

  - 假设一种情况，线程 A 进入后还未进行数据插入时挂起，而线程 B 正常执行，从而正常插入数据，然后线程 A 获取 CPU 时间片，此时线程 A 不用再进行 hash 判断了，问题出现：线程 A 会把线程 B 插入的数据给**覆盖**，发生线程不安全。

- **put和get并发时，可能导致get为null。**

  - jdk 1.7的transfer负责将旧的哈希表重新复制到新哈希表：https://blog.csdn.net/qq_32077121/article/details/108191063

  - 这里的src[j] =  null位置存在并发问题

  - jdk 1.8后将transfer归并到了resize中，看一下源码，这里的`oldTab[j] = null`;还是没有解决并发问题。

  - ```java
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;//并发问题，put进程爽了，get进程懵了
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
    ```

待看：

[Java 8系列之重新认识HashMap(美团)](https://tech.meituan.com/2016/06/24/java-hashmap.html)

[蔚来一面：HashMap 的 hash 方法原理是什么？( 沉默王二)](https://mp.weixin.qq.com/s/aS2dg4Dj1Efwujmv-6YTBg)

## HashCode

### String的实现

jdk8以前，是这样的：

```java
public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
```

公式归纳起来是这样的：

![image-20240520142144215](D:\picGo\images\image-20240520142144215.png)

### 为什么选择31作为乘子

简答：

- 31是一个奇质数，选择奇质数作为乘子 可以 降低哈希算法的冲突率 ，这是一个传统。（数学）

- 而比起其他奇质数

  - 2，算出的哈希范围太小，分布性不佳，冲突率高
  - 101，如果编码字符有5个以上，101 ^ 5 =1000亿+，就溢出了int的最大范围 20亿+，数据丢失

- 而且在计算 h * 31 时，可以优化为 31 * h = ( h  <<  5 )  -  i，被移位和减法运算取代，而现在jvm都支持这种优化

  

> https://stackoverflow.com/questions/299304/why-does-javas-hashcode-in-string-use-31-as-a-multiplier
>
> 来自这个问题的排名第一的答案：
>
> **选择数字31是因为它是一个奇质数，如果选择一个偶数会在乘法运算中产生溢出，导致数值信息丢失，因为乘二相当于移位运算。选择质数的优势并不是特别的明显，但这是一个传统。同时，数字31有一个很好的特性，即乘法运算可以被移位和减法运算取代，来获取更好的性能：`31 * i == (i << 5) - i`，现代的 Java 虚拟机可以自动的完成这个优化**。						——节选《Effective Java》
>
> 排名第二：
>
> **正如 Goodrich 和 Tamassia 指出的那样，如果你对超过 50,000 个英文单词（由两个不同版本的 Unix 字典合并而成）进行 hash code 运算，并使用常数 31, 33, 37, 39 和 41 作为乘子，每个常数算出的哈希值冲突数都小于7个，所以在上面几个常数中，常数 31 被 Java 实现所选用也就不足为奇了。**

https://cloud.tencent.com/developer/article/1518171

### jdk9+的实现

jdk9以后，加了编码器，首先isLatin1()判断String的编码，并分为Latin和UTF16来分别处理。

下面是jdk9以后的源码

```java
public int hashCode() {
        int h = hash;
        if (h == 0 && !hashIsZero) {
            h = isLatin1() ? StringLatin1.hashCode(value)
                           : StringUTF16.hashCode(value);
            if (h == 0) {
                hashIsZero = true;
            } else {
                hash = h;
            }
        }
        return h;
    }

```

如果是Latin-1编码：h = 31 * h + (v & 0xff);

在新的hashCode实现中，它选择舍弃高八位，保留第八位的该byte的ascll编码。

```java
public static int hashCode(byte[] value) {
    int h = 0;
    for (byte v : value) {
        h = 31 * h + (v & 0xff);
    }
    return h;
}
```

如果是UTF8的

涉及到一个getChar()闲下来看



