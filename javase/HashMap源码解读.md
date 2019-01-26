![](https://img.hacpai.com/bing/20180602.jpg?imageView2/1/w/960/h/520/interlace/1/q/100)

# HashMap解读

在面试的过程中,面试官经常会向面试者提问关于HashMap的问题,今天我将在这篇文章中仔细介绍一下HashMap.

## jdk7中的HashMap

### 介绍一下HashMap及其put和set方法实现
HashMap是由数组加上链表的数据结构书写的,它使用key-value键值对形式存储数据,每一个键值对也叫做Entry。这些个键值对（Entry）分散存储在一个数组当中，这个数组就是HashMap的主干。HashMap数组每一个元素的初始值都是Null。
![初始化结构](http://qiniuyun.indispensable.cn//file/2018/07/898937b5c6fb450facbc2d60b2011553_image.png) 



下面是几个比较重要的参数
> static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; //数组的默认长度,16
> static final float DEFAULT_LOAD_FACTOR = 0.75f;//默认的加载因子0.75
> static final int MAXIMUM_CAPACITY = 1 << 30;//数组的最大长度 2的30次方(1073741824)
> static final Entry<?,?>[] EMPTY_TABLE = {};
> transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;//`table'`用来存放数据的位置
> transient int size; // 存放的键值对的数量大小,Entry的数量
> int threshold;//桶(bucket)的大小，可在初始化时显式指定
> final float loadFactor;//加载因子，可在初始化时显式指定。
> transient int modCount;//修改的次数,用于fail-fast机制

这个数组的默认长度为16,默认加载因子为0.75,数组里面的每一个值初始化的时候默认为null.当桶中总的键值对(Entry)的数量达到`capacity` *  `loadFactor`的大小时,数组就会扩容,第一次扩容发生在数组中Entry数量为16*0.75f=12时.每次扩容都会使得数组的容量变为原来的两倍.这两个参数在创建HashMap对象的时候都可以指定,但我们一般不指定.

对于HashMap我们最常用的两个方法就是<strong>put</strong>和<strong>get</strong>
#### put
jdk7中的源码如下
```
    public V put(K key, V value) {
	//如果此时的table仍旧为初始化时的EMPTY_TABLE(空数组)的话,就对其进行初始化扩容
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
	//如果想要存放的数据的key值为空的话,那么就调用putForNullKey方法.
	//putForNullKey会覆盖掉原先的null值对应的Entry的value(如果存在的话)
        if (key == null)
            return putForNullKey(value);
	//对于非空的key存取方法如下
	//1.计算hash值
        int hash = hash(key);
	//2.计算该hash值在table中的位置
        int i = indexFor(hash, table.length);
	//3.判断此位置中是否有Entry存在,使用equals方法判断新插入的键是否等于原有的键, 
	//相同的话就覆盖原有Entry的值,不同的话就插入链表的最上方
	//并使新插入Entry指向原先的Entry
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
		//如果前面return的话,也就不会来到这里调用addEntry方法了
        addEntry(hash, key, value, i);
        return null;
    }
```
上面代码的注释看懂了吗?没看懂也没关系,我再用画图的方式解释一下.
当我们调用put方法时,会首先判断插入数据的键是否为空,如果为空的话就调用putForNullKey方法,在HashMap中只能够存放一个空键的数据,且这个数据一定存放在`table[0]`的位置.
如果插入的数据键不为空的话,那么就会计算键(key)对应的hash值并算出该hash值在table中的位置,如果此时该位置没有数据的话,那么就addEntry,为此键值对创建一个Entry并插入table中.
![空位置插入Entry](http://qiniuyun.indispensable.cn//file/2018/07/4fcc3775fdcc4e9a8013ab8443a3d335_image.png) 



但是此时如果此时的位置已经有Entry的话,就会再次判断,如果hash相同并且Entry的话就将原先的值取而代之,而键不做改变.如下图所示
![Entry值覆盖](http://qiniuyun.indispensable.cn//file/2018/07/92a82472d58b464da3b3e59ecda189ea_image.png) 

如果hash值不同的话,就会将数据插入新的原先的Entry位置,并指向于原先的Entry.在jdk7的HashMap中,同一个位置中后插入的数据一定在先插入数据的前面,因为HashMap的代码书写者认为后插入的数据比先插入的数据更有可能被使用.
注:在下图中省去了一个指针没有画出来,实际上在Entry这个内部类的定义中有一个指针,代码如下
```
static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;//此处定义了一个指针
        int hash;
```
![桶中插入Entry](http://qiniuyun.indispensable.cn//file/2018/07/38c7b696d64f4fad9d7d8fe0b7a9d669_image.png) 

#### get

jdk7中的源码如下
```   
public V get(Object key) {
	
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);
        return null == entry ? null : entry.getValue();
    }

```
get方法首先进行判断key是否为null,如果对应的键为null的话,就调用getForNullKey查询方法,直接去table[0]的位置查找有无键为null的Entry.如果键不为零的话那么就调用getEntry方法,getEntry的源码如下:
```
 final Entry<K,V> getEntry(Object key) {
	//首先判断Entry的数量是否为零,为零就不用查找了,直接返回null
        if (size == 0) {
            return null;
        }
	//计算hash值并找到该hash值在table中对应的位置,之后顺着链表一个个的比较hash值
	//hash值一致的话再比较键是否一致,一致则取出数据并返回
        int hash = (key == null) ? 0 : hash(key);
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
	//如果查找不到此key则返回null
        return null;
    }

```
下图是我在网上找的关于jdk7中HashMap的结构图,希望你通过这个图能够自行回忆出HashMap的结构和实现方法.
![HashMap结构图](http://qiniuyun.indispensable.cn//file/2018/07/003891b6ac1342ffb011d5105e34a4df__20180727153800.jpg) 

到了这里jdk7中HashMap的get和set方法基本就就讲完了,但是不知道读者们有没有发现HashMap在数据插入和读取时存在的一个问题,当HashMap中同一个桶中的键值对越多的时候,就越有可能发生hash冲突问题,每次对table里同一个桶的Entry的Hash值进行比较的话,时间复杂度为O(n),大量的hash冲突会使得数据的读写性能下降,这个问题在jdk8中作出了优化,你继续读下去就会得到答案.

## jdk7和jdk8中HashMap实现的不同之处

  jdk7 中使用Entry 来代表每个 HashMap 中的数据节点，Java8 中使用 Node，基本没有区别，都是 key，value，hash 和 next 这四个属性，不过，Node 只能用于链表的情况，红黑树的情况需要使用 TreeNode。下面的图片是我在网上找的关于jdk8中HashMap的实现.

![jdk8中HashMap的实现](http://qiniuyun.indispensable.cn//file/2018/07/ed029fad966042dcad753efbcda055c3_image.png) 


  jdk8中的HashMap代码个人认为易读性不够好,博主水平有限,希望大家轻喷.
那么现在就开始解读代码吧,首先我觉得需要进行解释的是下面两个字段

> static final int TREEIFY_THRESHOLD = 8;
> static final int UNTREEIFY_THRESHOLD = 6;

  当数组中某一个位置中所包含键值对的数目大于TREEIFY_THRESHOLD时,比如说我们在table[2]的位置插入第九个元素的时候,这个桶的数据结构就会从链表向红黑树转换,此时如果在数组此位置进行数据的存取的话,那么时间复杂度就变为了O(logn),较之前的O(n)得到了一定的速度提升.
而当HashMap进行resize的时候,每一个桶中的键值对的数目势必要下降,如果这个桶中,也就是数组的某个位置它所对应的结点(键值对)数目小于UNTREEIFY_THRESHOLD时,数据结构就会又变回链表结构.

#### put 
  下面我们来看一下put的源码

```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
 
// 第三个参数 onlyIfAbsent 如果是 true的话，那么只有在不存在这个 key 时才会进行 put 操作
//我们传递过来的值是false
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 第一次 put 值的时候，会触发下面的 resize()，类似 jdk7 的第一次 put操作 也要初始化数组长度
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 找到具体的数组下标，如果这个位置上没有node,那么就直接创建
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
 
    else {// 说明数组该位置有数据
        Node<K,V> e; K k;
        // 首先，判断该位置的第一个键值对(node)和我们要插入的数据，key 是不是"相等"，
		  //
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果该节点是代表红黑树的节点，调用红黑树的插值方法
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 到这里else中来，说明数组该位置上是一个链表
            for (int binCount = 0; ; ++binCount) {
                // 插入到链表的最后面(Jdk7 是插入到链表的最前面)
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // TREEIFY_THRESHOLD 为 8，所以，如果新插入的值是链表中的第 9 个
                    // 会触发下面的 treeifyBin，也就是将链表转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果在该链表中找到了"相等"的 key(== 或 equals)
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 此时 break，那么 e 为链表中[与要插入的新值的 key "相等"]的 node
                    break;
                p = e;
            }
        }
        // e!=null 说明存在旧值的key与要插入的key"相等"
        // 和jdk7中一样,保持数据的key不变,将值覆盖为新插入的值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;//修改次数+1
    // 如果 HashMap 由于新插入这个值导致 size 已经超过了阈值，需要进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;

```
#### resize
  在jdk7代码的解说中,我并没有详细介绍resize的代码,在这里我进行解说一下,二者的实现方法大致相同,理解其中一个就可以了

```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) { // 对应数组扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 将数组大小扩大一倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 将阈值(threshold)扩大一倍
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) 
        newCap = oldThr;
    else {// 对应第一次 put 的时候对数据进行初始化
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
 
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
 
    // 用新的数组大小初始化新的数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab; // 如果是初始化数组，到这里就结束了，返回 newTab 即可
 
    if (oldTab != null) {
        // 开始遍历原数组，进行数据迁移。
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果该数组位置上只有单个元素，迁移这个元素就可以了
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果是红黑树，具体我们就不展开了(因为我不太会)
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { 
                    // 这块是处理链表的情况，
                    // 需要将此链表拆成两个链表，放到新的数组中，并且保留原来的先后顺序
                    // loHead、loTail 对应一条链表，hiHead、hiTail 对应另一条链表，代码还是比较简单的
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        // 第一条链表
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        // 第二条链表的新的位置是 j + oldCap
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;

```
#### get
  对于get方法,此处不做详细介绍,主要要读者们记住的就是,在get操作的时候,会判断是否是采用红黑树存储键值对数据,我们根据数组元素中，第一个节点数据类型是 Node 还是 TreeNode 来判断该位置下是链表还是红黑树的。两者采用的读取方式不同。源码附上:
```

final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```


## 关于HashMap你必须要知道的那些事

### HashMap是线程安全的吗
答:HashMap不是线程安全的,HashMap在设计的时候就是为了单线程而服务的,如果你想在多线程中使用HashMap的话,可以考虑使用ConcurrentHashMap或者HashTable.

### HashMap在多线程下面可能出现的问题
之前已经说了HashMap并非是线程安全的,如果你硬要在多线程的情况下使用HashMap,可能出现的问题就是CPU占用率达到百分之百,这个问题我将会专门写一个博客.欢迎关注.

### 如果两个键值对键的hashCode相同的话,那么在数据的put和get时会怎样去做
当出现hashCode相同的情况时,那么hash值也一定相同,这两个键值对所处的数组位置也就是相同的,put时调用equals方法比较是否有键和新插入的键一致,如果有的话,就覆盖原先键值对的值,没有的话就正常将数据插入.在get时同理,仅仅是hash值相同还不够,必须要比较键是否是一样的.

下面两个问题留给读者自己思考
### HashMap在jdk7和jdk8的差异

### 为什么重写equals方法就必须要重写hashCode方法

这篇博客就到这里结束啦,如果你有任何的疑问或者给我的建议的话,欢迎在底下进行留言.这篇博客似乎写的过于长了,第一次写博客,能力有限,好好加油吧.









