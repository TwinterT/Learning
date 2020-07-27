# HashMap 1.7

HashMap本质是一个存储Entry类对象的数组+多个单链表

```java
//核心静态内部类Entry 
//Entry对象本质 = 1个映射（键 - 值对），属性包括：键（key）、值（value） & 下1节点( next) 
static class Entry<K,V> implements Map.Entry<K,V> {
  final K key;
  V value;
  //因为HashMap是采用链表法处理哈希冲突的，所以Entry需要有一个指向下一个节点的指针
  Entry<K,V> next;
  int hash;
  /**
  * Creates new entry.
  */
  Entry(int h, K k, V v, Entry<K,V> n) {
    value = v;
    next = n;
    key = k;
    hash = h;
  }
//......部分源码已省略.......
}
```



### 1. 容量、加载因子、扩容阈值

```java
// 1. 容量（capacity）： HashMap中数组的长度
// a. 容量范围：必须是2的幂 & <最大容量（2的30次方）
// b. 初始容量 = 哈希表创建时的容量
// 默认容量 = 16 = 1<<4 = 00001中的1向左移4位 = 10000 = 十进制的2^4=16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
// 最大容量 =  2的30次方（若传入的容量过大，将被最大值替换）
static final int MAXIMUM_CAPACITY = 1 << 30;

// 2. 加载因子(Load factor)：HashMap在其容量自动增加前可达到多满的一种尺度
// a. 加载因子越大、填满的元素越多 = 空间利用率高、但冲突的机会加大、查找效率变低（因为链表变长了）
// b. 加载因子越小、填满的元素越少 = 空间利用率小、冲突的机会减小、查找效率高（链表不长）
// 实际加载因子
final float loadFactor;
// 默认加载因子 = 0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 3. 扩容阈值（threshold）：当哈希表的大小 ≥ 扩容阈值时，就会扩容哈希表（即扩充HashMap的容量） 
// a. 扩容 = 对哈希表进行resize操作（即重建内部数据结构），从而哈希表将具有大约两倍的桶数
// b. 扩容阈值 = 容量 x 加载因子
int threshold;

// 4. 其他
// 存储数据的Entry类型 数组，长度 = 2的幂
// HashMap的实现方式 = 拉链法，Entry数组上的每个元素本质上是一个单向链表
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;  
// HashMap的大小，即 HashMap中存储的键值对的数量
transient int size;
```



### 2.向HashMap添加数据(成对放入键值对)

```java
/**
     * 源码分析：主要分析： HashMap的put函数
     */
public V put(K key, V value)
	//（分析1） 1. 若 哈希表未初始化（即 table为空) 
	// 则使用 构造函数时设置的阈值(即初始容量) 初始化 数组table  
  if (table == EMPTY_TABLE) { 
    inflateTable(threshold); 
  }  
  // 2. 判断key是否为空值null
  //（分析2） 2.1 若key == null，则将该键-值 存放到数组table 中的第1个位置，即table [0]
  // （本质：key = Null时，hash值 = 0，故存放到table[0]中）
  // 该位置永远只有1个value，新传进来的value会覆盖旧的value
  if (key == null)
    return putForNullKey(value);
	//（分析3） 2.2 若 key ≠ null，则计算存放数组 table 中的位置（下标、索引）
	// a. 根据键值key计算hash值
	int hash = hash(key);
	// b. 根据hash值 最终获得 key对应存放的数组Table中位置
	int i = indexFor(hash, table.length);

	// 3. 判断该key对应的值是否已存在（通过遍历 以该数组元素为头结点的链表 逐个判断）
	for (Entry<K,V> e = table[i]; e != null; e = e.next) {
    Object k;
    //（分析4） 3.1 若该key已存在（即 key-value已存在 ），则用 新value 替换 旧value
    if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
      V oldValue = e.value;
      e.value = value;
      e.recordAccess(this);
      return oldValue; //并返回旧的value
    }
  }
	modCount++;
	
	//（分析5） 3.2 若 该key不存在，则将“key-value”添加到table中
	addEntry(hash, key, value, i);
	return null;
}
```

#### 分析1：初始化哈希表

> 即 初始化数组（table）、扩容阈值（threshold）

```java
/**
* 函数使用原型
*/
if (table == EMPTY_TABLE) { 
  inflateTable(threshold); 
}  

/**
* 源码分析：inflateTable(threshold); 
*/
private void inflateTable(int toSize) {  
    
    // 1. 将传入的容量大小转化为：>传入容量大小的最小的2的次幂
    // 即如果传入的是容量大小是19，那么转化后，初始化容量大小为32（即2的5次幂）
    int capacity = roundUpToPowerOf2(toSize);->>分析1   

    // 2. 重新计算阈值 threshold = 容量 * 加载因子  
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);  

    // 3. 使用计算后的初始容量（已经是2的次幂） 初始化数组table（作为数组长度）
    // 即 哈希表的容量大小 = 数组大小（长度）
    table = new Entry[capacity]; //用该容量初始化table  

    initHashSeedAsNeeded(capacity);  
}  

    /**
     * 分析1：roundUpToPowerOf2(toSize)
     * 作用：将传入的容量大小转化为：>传入容量大小的最小的2的幂
     * 特别注意：容量大小必须为2的幂，该原因在下面的讲解会详细分析
     */

private static int roundUpToPowerOf2(int number) {  
   //若 容量超过了最大值，初始化容量设置为最大值 ；否则，设置为：>传入容量大小的最小的2的次幂
  return number >= MAXIMUM_CAPACITY  ? 
            MAXIMUM_CAPACITY  : (number > 1) ? Integer.highestOneBit((number - 1) << 1)
}
```

**真正初始化哈希表（初始化存储数组table）是在第1次添加键值对时，即第1次调用put（）时**



#### 分析2：当key==null时

```java
/**
* 函数使用原型
*/
if (key == null)
  return putForNullKey(value);

/**
* 源码分析：putForNullKey(value)
*/
private V putForNullKey(V value) {  
  // 遍历以table[0]为首的链表，寻找是否存在key==null 对应的键值对
  // 1. 若有：则用新value 替换 旧value；同时返回旧的value值
  for (Entry<K,V> e = table[0]; e != null; e = e.next) {  
    if (e.key == null) {   
      V oldValue = e.value;  
      e.value = value;  
      e.recordAccess(this);  
      return oldValue;  
    }  
  }  
  modCount++;  
  // 2 .若无key==null的键，那么调用addEntry（），将空键 & 对应的值封装到Entry中，并放到table[0]中
  addEntry(0, null, value, 0); 
  // 注：
  // a. addEntry（）的第1个参数 = hash值 = 传入0
  // b. 即 说明：当key = null时，也有hash值 = 0，所以HashMap的key 可为null
  // c. 对比HashTable，由于HashTable对key直接hashCode（），若key为null时，会抛出异常，所以HashTable的key不可为null
  // d. 此处只需知道是将 key-value 添加到HashMap中即可，关于addEntry（）的源码分析将等到下面再详细说明，
  return null;  
}
```



#### 分析3：计算存放数组 table 中的位置

```java
/**
* 函数使用原型
* 主要分为2步：计算hash值、根据hash值再计算得出最后数组位置
*/
// a. 根据键值key计算hash值 ->> 分析1
int hash = hash(key);
// b. 根据hash值 最终获得 key对应存放的数组Table中位置 ->> 分析2
int i = indexFor(hash, table.length);

/**
* 源码分析1：hash(key)
* 该函数在JDK 1.7 和 1.8 中的实现不同，但原理一样 = 扰动函数 = 使得根据key生成的哈希码（hash值）分布更加均匀、更具备随机性，避免出现hash值冲突（即指不同key但生成同1个hash值）
* JDK 1.7 做了9次扰动处理 = 4次位运算 + 5次异或运算
* JDK 1.8 简化了扰动函数 = 只做了2次扰动 = 1次位运算 + 1次异或运算
*/

// JDK 1.7实现：将 键key 转换成 哈希码（hash值）操作  = 使用hashCode() + 4次位运算 + 5次异或运算（9次扰动）
static final int hash(int h) {
  h ^= k.hashCode(); 
  h ^= (h >>> 20) ^ (h >>> 12);
  return h ^ (h >>> 7) ^ (h >>> 4);
}

// JDK 1.8实现：将 键key 转换成 哈希码（hash值）操作 = 使用hashCode() + 1次位运算 + 1次异或运算（2次扰动）
// 1. 取hashCode值： h = key.hashCode() 
//  2. 高位参与低位的运算：h ^ (h >>> 16)  
static final int hash(Object key) {
  int h;
  return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  // a. 当key = null时，hash值 = 0，所以HashMap的key 可为null      
  // 注：对比HashTable，HashTable对key直接hashCode（），若key为null时，会抛出异常，所以HashTable的key不可为null
  // b. 当key ≠ null时，则通过先计算出 key的 hashCode()（记为h），然后 对哈希码进行 扰动处理： 按位 异或（^） 哈希码自身右移16位后的二进制
}

/**
* 函数源码分析2：indexFor(hash, table.length)
* JDK 1.8中实际上无该函数，但原理相同，即具备类似作用的函数
*/
static int indexFor(int h, int length) {  
  return h & (length-1); 
  // 将对哈希码扰动处理后的结果 与运算(&) （数组长度-1），最终得到存储在数组table的位置（即数组下标、索引）
}
```



* 为什么不直接采用经过hashCode()处理的哈希码 作为 存储数组table的下标位置？

  容易出现哈希码与数组大小范围不匹配的情况，即计算出来的哈希码可能不在数组大小范围内，从而导致无法匹配存储位置

  

* 为什么采用 哈希码 与运算(&) （数组长度-1） 计算数组下标？

  根据HashMap的容量大小（数组长度），按需取 哈希码一定数量的低位作为存储的数组下标位置，从而解决 “哈希码与数组大小范围不匹配” 的问题

  

* 为什么在计算数组下标前，需对哈希码进行二次处理：扰动处理？

  加大哈希码低位的随机性，使得分布更均匀，从而提高对应数组存储下标位置的随机性 & 均匀性，最终减少Hash冲突



#### 分析4：若对应的key已存在，则 使用 新value 替换 旧value

**注：当发生 Hash冲突时，为了保证 键key的唯一性哈希表并不会马上在链表中插入新数据，而是先查找该 key是否已存在，若已存在，则替换即可**

```java
/**
     * 函数使用原型
     */
// 2. 判断该key对应的值是否已存在（通过遍历 以该数组元素为头结点的链表 逐个判断）
for (Entry<K,V> e = table[i]; e != null; e = e.next) {
  Object k;
  // 2.1 若该key已存在（即 key-value已存在 ），则用 新value 替换 旧value
  if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
      V oldValue = e.value;
      e.value = value;
      e.recordAccess(this);
      return oldValue; //并返回旧的value
  }
}
modCount++;

// 2.2 若 该key不存在，则将“key-value”添加到table中
addEntry(hash, key, value, i);
return null;
```



#### 分析5：若对应的key不存在，则将该“key-value”添加到数组table的对应位置中

> 此处有2点需特别注意：键值对的添加方式 & 扩容机制

##### 1.键值对的添加方式：单链表的头插法

即将该位置（数组上）原来的数据放在该位置的（链表）下1个节点中（next）、在该位置（数组上）放入需插入的数据-> 从而形成链表



##### 2.扩容机制

![扩容](img/hashmap_resize.png?raw=true)

扩容过程中的转移数据示意图如下:

![转移](img/hashmap_transfer.png?raw=true)

**此时若（多线程）并发执行 put（）操作，一旦出现扩容情况，则 容易出现 环形链表，从而在获取数据、遍历链表时 形成死循环（Infinite Loop），即 死锁的状态 = 线程不安全**



# 与 JDK 1.8的区别

* JDK1.7用的是头插法，而JDK1.8及之后使用的都是尾插法，那么他们为什么要这样做呢？因为JDK1.7是用单链表进行的纵向延伸，当采用头插法时会容易出现逆序且环形链表死循环问题。但是在JDK1.8之后是因为加入了红黑树使用尾插法，能够避免出现逆序且链表死循环的问题。

* 扩容后数据存储位置的计算方式也不一样：
  1. 在JDK1.7的时候是直接用hash值和需要扩容的二进制数进行&（这里就是为什么扩容的时候为啥一定必须是2的多少次幂的原因所在，因为如果只有2的n次幂的情况时最后一位二进制数才一定是1，这样能最大程度减少hash碰撞）（hash值 & length-1）
  2. 而在JDK1.8的时候直接用了JDK1.7的时候计算的规律，也就是扩容前的原始位置+扩容的大小值=JDK1.8的计算方式，而不再是JDK1.7的那种异或的方法。但是这种方式就相当于只需要判断Hash值的新增参与运算的位是0还是1就直接迅速计算出了扩容后的储存方式。

![区别](img/hashmap_diff.png?raw=true)



* JDK1.7的时候使用的是数组+ 单链表的数据结构。但是在JDK1.8及之后时，使用的是数组+链表+红黑树的数据结构（当链表的深度达到8的时候，也就是默认阈值，就会自动扩容把链表转成红黑树的数据结构来把时间复杂度从O（n）变成O（logN）提高了效率）
* 为什么在JDK1.8中进行对HashMap优化的时候，把链表转化为红黑树的阈值是8,而不是7或者不是20呢？

  1. 如果选择6和8（如果链表小于等于6树还原转为链表，大于等于8转为树），中间有个差值7可以有效防止链表和树频繁转换。假设一下，如果设计成链表个数超过8则链表转换成树结构，链表个数小于8则树结构转换成链表，如果一个HashMap不停的插入、删除元素，链表个数在8左右徘徊，就会频繁的发生树转链表、链表转树，效率会很低。
  2. 还有一点重要的就是由于treenodes的大小大约是常规节点的两倍，因此我们仅在容器包含足够的节点以保证使用时才使用它们，当它们变得太小（由于移除或调整大小）时，它们会被转换回普通的node节点，容器中节点分布在hash桶中的频率遵循泊松分布，桶的长度超过8的概率非常非常小。所以作者应该是根据概率统计而选择了8作为阀值
     