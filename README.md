# HashMap
HashMap 详解

![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/hashmap.png)

## 四种实现概述：
#### 1. HashMap：
###### 它根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。 HashMap最多只允许一条记录的键为null，允许多条记录的值为null。HashMap非线程安全，即任一时刻可以有多个线程同时写HashMap，可能会导致数据的不一致。如果需要满足线程安全，可以用 Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。

#### 2. Hashtable：
###### Hashtable是遗留类，很多映射的常用功能与HashMap类似，不同的是它承自Dictionary类，并且是线程安全的，任一时间只有一个线程能写Hashtable，并发性不如ConcurrentHashMap，因为ConcurrentHashMap引入了分段锁。Hashtable不建议在新代码中使用，不需要线程安全的场合可以用HashMap替换，需要线程安全的场合可以用ConcurrentHashMap替换。

#### 3. LinkedHashMap：
###### LinkedHashMap是HashMap的一个子类，保存了记录的插入顺序，在用Iterator遍历LinkedHashMap时，先得到的记录肯定是先插入的，也可以在构造时带参数，按照访问次序排序。

#### 4. TreeMap：
###### TreeMap实现SortedMap接口，能够把它保存的记录根据键排序，默认是按键值的升序排序，也可以指定排序的比较器，当用Iterator遍历TreeMap时，得到的记录是排过序的。如果使用排序的映射，建议使用TreeMap。在使用TreeMap时，key必须实现Comparable接口或者在构造TreeMap传入自定义的Comparator，否则会在运行时抛出java.lang.ClassCastException类型的异常。

## HashMap 详解
#### 1. 底层实现原理：数组+链表+红黑树（JDK1.8增加了红黑树部分）
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/hashmap_datas_tructure2.png)
#### 2. 为什么数组初始长度是 16，以后扩容都是 2 的幂次倍数？
###### 之所以选择 16，是为了服务于从 Key 映射到 index 的 Hash 算法。从 Key 映射到 HashMap 数组的对应位置，会用到一个 Hash 函数：
    index = Hash(key)
###### 如何实现一个尽量均匀分布的 Hash 函数呢？我们通过利用 Key 的 HashCode 值来做某种运算 --- 位运算 & (不能采用取模运算，虽然简单但是效率很低)
###### 如何进行位运算(&)呢？有如下公式(Length 是 HashMap 的长度)：
    index = HashCode(key) & (Length - 1)
###### 下面我们以值为“book”的Key来演示整个过程：
    1.计算book的hashcode，结果为十进制的3029737，二进制的101110001110101110 1001。
    2.假定HashMap长度是默认的16，计算Length-1的结果为十进制的15，二进制的1111。
    3.把以上两个结果做与运算，101110001110101110 1001 & 1111 = 1001，十进制是9，所以 index=9。
###### 可以说，Hash算法最终得到的index结果，完全取决于Key的Hashcode值的最后几位。
###### 那么为什么长度必须是 16 或者 2 的幂呢？ 假设 HashMap 长度是 10 会怎样？
    HashCode: 10 1110 0011 1010 1110 1011
    Length-1:                        1001
       index:                        1001
       
    HashCode: 10 1110 0011 1010 1110 1111
    Length-1:                        1001
       index:                        1001
       
    上面两个例子，虽然 HashCode 的倒数第二第三位从 0 变成了 1，但是运算的结果都是 1001。
    也就是说，当 HashMap 长度为 10 的时候，有些 index 结果的出现几率会更大，而有些 index 结果永远不会出现（比如 0111）！
    这样，显然不符合 Hash 算法均匀分布的原则。
    如果长度是 16 或者是 2 的幂， Length-1 的值的所有二进制位全为 1，这种情况下，index的结果等同于HashCode后几位的值。
    只要输入的HashCode本身分布均匀，Hash算法的结果就是均匀的。
#### 3. HashMap 扩容(Resize)
###### 1) 影响发生 Resize 的因素有两个：
        1. Capacity - HashMap 当前长度
        2. LoadFactor - 负载因子，默认值 0.75f
###### 2) 发生 Resize 条件：HashMap.Size >= Capacity * LoadFactor
###### 3) Resize 过程：
        1. 扩容：创建一个新的Entry空数组，长度是原数组的2倍。
        2. ReHash：遍历原 Entry 数组，把所有的 Entry 重新 Hash 到新数组。
        为什么要重新Hash呢？因为长度扩大以后，Hash 的规则也随之改变。
        回顾：index = HashCode(key) & (Length - 1)
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/HashMap%20Resize.png)
## Rehash 死循环 - 环形链表形成过程
        /**
         * Transfers all entries from current table to newTable.
         */
        void transfer(Entry[] newTable, boolean rehash) {
            int newCapacity = newTable.length;
            for (Entry<K,V> e : table) {
                while(null != e) {
                    Entry<K,V> next = e.next;
                    if (rehash) {
                        e.hash = null == e.key ? 0 : hash(e.key);
                    }
                    int i = indexFor(e.hash, newCapacity);
                    e.next = newTable[i];
                    newTable[i] = e;
                    e = next;
                }
            }
        }
#### 假设一个 HashMap 已经到了 Resize 的临界点。此时有两个线程A和B，在同一时刻对 HashMap 进行 Put 操作：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash1.png)
#### 此时达到Resize条件，两个线程各自进行Rezie的第一步，也就是扩容：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash2.png)
#### 这时候，两个线程都走到了 ReHash 的步骤。让我们回顾一下ReHash的代码：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash%20code1.png)
#### 假如此时线程B遍历到Entry3对象，刚执行完红框里的这行代码，线程就被挂起。对于线程B来说：

    #### e = Entry3
    #### next = Entry2

#### 这时候线程A畅通无阻地进行着Rehash，当ReHash完成后，结果如下（图中的e和next，代表线程B的两个引用）：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash3.png)
直到这一步，看起来没什么毛病。接下来线程B恢复，继续执行属于它自己的ReHash。线程B刚才的状态是：
    
    #### e = Entry3
    #### next = Entry2
    
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash%20code2.png)
#### 当执行到上面这一行时，显然 i = 3，因为刚才线程A对于Entry3的hash结果也是3。
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash%20code3.png)
#### 我们继续执行到这两行，Entry3放入了线程B的数组下标为3的位置，并且e指向了Entry2。此时e和next的指向如下：

    #### e = Entry2
    #### next = Entry2
    
#### 整体情况如图所示：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash4.png)
#### 接着是新一轮循环，又执行到红框内的代码行：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash%20code1.png)

    #### e = Entry2
    #### next = Entry3
    
#### 整体情况如图所示：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash4.png)
#### 接下来执行下面的三行，用头插法把Entry2插入到了线程B的数组的头结点：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash%20code5.png)
#### 整体情况如图所示：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash6.png)
#### 第三次循环开始，又执行到红框的代码：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash%20code1.png)

    #### e = Entry3
    #### next = Entry3.next = null
    
#### 最后一步，当我们执行下面这一行的时候，见证奇迹的时刻来临了：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash%20code5.png)

    #### newTable[i] = Entry2
    #### e = Entry3
    #### Entry2.next = Entry3
    #### Entry3.next = Entry2
    
#### 链表出现了环形！
#### 整体情况如图所示：
![示例图片](https://github.com/Muran-Hu/HashMap/blob/master/Rehash7.png)
#### 此时，问题还没有直接产生。当调用Get查找一个不存在的Key，而这个Key的Hash结果恰好等于3的时候，由于位置3带有环形链表，所以程序将会进入死循环！
