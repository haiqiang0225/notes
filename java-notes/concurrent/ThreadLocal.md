[toc]

# ThreadLocal

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。

## ThreadLocal介绍

**ThreadLocal**是```java.lang```下提供的工具类，```ThreadLocal```提供了线程级别的变量访问域，也就是说```ThreadLocal```提供了线程的局部变量，这个局部变量实现了线程间的数据隔离。并且```ThreadLocal```提供的局部变量是与线程绑定的，只要我们不手动将它清楚，那么在线程存活的生命周期里，这个变量是一直存在的，不管线程在执行什么方法，执行哪个类的方法，只要我们有```ThreadLocal```实例的引用，我们就可以使用这个线程级别的变量。

所以，```ThreadLocal```是用于**线程间的变量隔离**，而显式锁、```synchronized```关键字则是用于**线程间变量共享的同步**问题。

```ThreadLocal```的实现原理其实非常非常的简单：```Thread```类中有一个```ThreadLocal.ThreadLocalMap```的实例```threadLocals```，本质上就是和```HashMap```差不多的东西，是个保存K-V结构数据的工具类。当我们调用```threadLocal.set(T)```*<u>实例方法</u>*来保存变量Val时，实际上是把我们的变量Val存到了```Thread```内部的```threadLocals```这个Map中了，其中Key是我们的```ThreadLocal```实例，当我们执行```threadLocal.get()```*<u>实例方法</u>*时，实际上是以```threadLocal```为Key把Val从这个Map中取出来了；执行```threadLocal.remove()```方法，会把```Thread```内部的Map的Key的引用（是个弱引用，```WeakReference```的子类）设置为```null```，这样我们再取的时候就获取不到值了。

通过上面的说明不难看出，```ThreadLocal```本身是不保存我们的局部变量的，只是个工具人，真正存放变量的数据结构其实存放在```Thread```类中，这也是为什么```ThreadLocal```的变量能实现线程隔离的原因：**我们大家只是把你当成Key，只是一把“钥匙”，宝箱在我自己（线程）手里，钥匙大家都有，但是宝箱确是我自己独享的。当我需要宝箱里的宝贝（局部变量）时，只是把钥匙拿来，打开我自己的宝箱拿出属于我自己的宝物，当我不需要时，就把钥匙扔了，顺带把箱子里的东西也扔了（Key和Val都置为null）。但是钥匙不是我自己（Thread类）保管的，我只有个牌子（弱引用），这个牌子能证明我可以使用这个钥匙，有的时候，保管钥匙的人把钥匙丢了（ThreadLocal实例被GC了），我的牌子也就失效了，更可恶的是，我 的 宝 贝，它还在箱子里锁着，我这辈子都没法拿出它来了**。

![//TO-DO ThreadLocal引用](https://img-blog.csdnimg.cn/20210320154712243.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE5MDA3MzM1,size_16,color_FFFFFF,t_70)


上面这个小比喻中箱子就是Map中的```Entry```（它是一个```WeakReference```的子类，```reference```属性指向Key），当```ThreadLocal```实例只有```Entry```（弱引用）指向它时，下次JVM发生GC的时候就会把```ThreadLocal```实例回收，但是还有```Entry```->```Val```这么一条强引用，但是我们没法通过正常手段来获得它了，这个时候就发生了内存泄漏。除此之外我们还发现，多个线程会同时使用同一个```ThreadLocal```实例作为```ThreadLocalMap```的key。

## ThreadLocal使用

```ThreadLocal```可以用来保存数据库连接，或者用来传递参数。

```java
public class ConnectionManager {

    private static final ThreadLocal<Connection> context = new ThreadLocal<>();

    public static Connection getConnection() {
        Connection connection = null;
        if ((connection = context.get()) != null && connection.isValid()) {
            return connection;
        }
        connection = Connection.connect();
        context.set(connection);
        return connection;
    }

    public static void closeConnection() {
        Connection connection = null;
        if ((connection = context.get()) != null) {
            connection.close();
            context.remove();
        }
    }

    static class Connection {
        private static int COUNTER;
        private final String name;

        public Connection() {
            name = "connection-" + nextCount();
        }

        public Connection(String name) {
            this.name = name;
        }

        public static Connection connect() {
            return new Connection();
        }

        public boolean isValid() {
            return ThreadLocalRandom.current().nextBoolean();
        }

        public void close() {
            System.out.println(name + ": successfully closed!");
        }

        @Override
        public String toString() {
            return super.toString();
        }

        public synchronized static int nextCount() {
            return COUNTER++;
        }
    }

}
```

## ThreadLocal源码

我们使用```ThreadLocal```时主要使用```get()、set(T)、remove()```这三个方法，先从```get()```的源码看起。

### get()方法源码

```java
public T get() {
    Thread t = Thread.currentThread();
    // getMap ->  return t.threadLocals 返回Thread内部的ThreadLocalMap实例引用
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 获取对应的Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        // 如果存在就返回对应的value
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // map为null 或者 虽然map不为null但是没有对应的Entry
    return setInitialValue();
}

private T setInitialValue() {
    // 获取默认的初值，initialValue()默认为返回null，
    // 如果想获取其它初始值，需要重写initialValue()
    T value = initialValue();
    Thread t = Thread.currentThread();
    // 获取map
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // key是当前ThreadLocal实例
        map.set(this, value);
    else
        // map还未创建的话，就创建一个map
        createMap(t, value);
    // 返回初始值
    return value;
}
// 简单的创建一个ThreadLocalMap实例返回即可
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

可以看到get方法还是比较简单的：

1. 先尝试从map中获取，有的话直接返回；如果map为空或者map中没有这个值，就执行```initialValue()```方法获得一个初始值
2. 获得初始值后，如果map不为空就把```this```作为key，初始值作为value存入map，否则的话，建立一个map然后存入，最后返回value

### set(T)方法源码

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    // 获取map
    ThreadLocalMap map = getMap(t);
    // map不为空就存入，为空的话创建一个map然后存入
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

### remove()方法

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    // 如果map不为null，就执行map的remove方法，把this作为key
    if (m != null)
        m.remove(this);
}
```

## ThreadLocalMap

**ThreadLocalMap**是```ThreadLocal```的内部类，从名字就能看出来它是一个K-V的hash map结构。它的唯一作用就是保存线程局部变量。和其它Map实现一样，Entry用来保存键值对。```ThreadLocalMap```使用<u>开放地址法</u>解决哈希冲突，在不冲突的情况下，key存放在```key.threadLocalHashCode & (table.length - 1)```处

### 内部类Entry以及内部属性和部分方法

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;
    // 对k是弱引用，对value是强引用
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
// 初始大小
private static final int INITIAL_CAPACITY = 16;

// 哈希桶
private Entry[] table;

// 哈希桶中有效实体的数量
private int size = 0;

// 哈希桶扩容的阈值，默认是0，达到最大值的2/3会扩容（除了第一次扩容）
private int threshold; // Default to 0

// 设置阈值，默认是哈希桶长度的2/3
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}

// 计算下一个index，防止越界
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

// 计算上一个index，防止越界
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

#### Entry为什么要继承WeakReference以及ThreadLocal内存泄漏

考虑一种情况：外部所有对```ThreadLocal```实例的强引用如果全部都成为了null，可能是手动设置的，也可能是持有强引用的对象死亡被GC了，但是不论哪种情况，这个```ThreadLocal```实例在外部都没法访问了，如果Entry对key的引用为强引用的话，那么这个key在GC时是不会被回收的。然鹅这个key却没法访问了，这就出现了内存泄漏，所以用弱引用，当```ThreadLocal```实例仅在Entry中被引用时，下一次GC就会回收这个实例。**但是**，这只是解决了一部分问题，因为Entry中value的强引用，会导致我们的value不能被回收。具体而言有这么一条引用：Thread -> threadLocals -> Entry -> value，最后的value指向了我们的局部变量，并且不难看出。Entry -> k 这个引用对上面的引用链是没有影响的，换句话说就是```ThreadLocal```实例被回收与否与我们的局部变量能否被回收是没有关系的。所以我们每次使用完```ThreadLocal```后都应该手动执行```remove```方法来清除局部变量的引用。

![//TO-DO 内存泄漏的图片](https://img-blog.csdnimg.cn/20210320154737119.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE5MDA3MzM1,size_16,color_FFFFFF,t_70)


### remove()方法

```ThreadLocal```中的```remove()```方法只是简单的调用```ThreadLocalMap.remove()```方法，```ThreadLocalMap.remove()```方法源码如下：

```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    // 获得key不冲突情况下的存放位置i
    int i = key.threadLocalHashCode & (len-1);
    // 有可能冲突，利用开放地址法来找key对应的Entry e
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        // 如果找到了key
        if (e.get() == key) {
            // 清除e对key的引用
            e.clear();
            // 清除“陈旧”的Entry
            expungeStaleEntry(i);
            return;
        }
    }
}


```

```remove()```方法本身并不会清除value对局部变量的强引用，真正清除的方法在```expungeStaleEntry(i);```中。

### expungeStaleEntry(int staleSlot)：清除陈旧的Entry

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 删除传入的槽（slot）中的value强引用和这个Entry
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    // 哈希桶中有效实体数减一
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    // 从当前槽的nextIndex开始rehash，每次递进一个槽，直到遇到null的Entry
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        // 获取当前Entry中的key，即ThreadLocal实例
        ThreadLocal<?> k = e.get();
        // 这里说明Entry不为null，在这个前提下，如果k为null，说明
        // 发生了key被回收但是value没有被回收的情况，即发生了内存泄漏
        if (k == null) {
            // 把value强引用断掉，并把i处的Entry引用也断开，帮助GC
            e.value = null;
            tab[i] = null;
            // 有效计数减一
            size--;
        } else {
            // k 不为null就rehash，先计算h在不冲突的情况下存储的位置
            int h = k.threadLocalHashCode & (len - 1);
            // 说明这个Entry是冲突后才被存储到i处的
            if (h != i) {
                // 需要转移Entry，先把哈希桶中的当前位置引用清空
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                // rehash，把当前的Entry使用开放地址法转移到它应该存储的位置
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    // 返回的是从被remove的Entry之后的第一个为null的槽的位置
    return i;
}
```

清除陈旧Entry的代码不复杂，大致逻辑如下：

1. 首先清除指定Entry的强引用，之后从这个槽后的第一个位置开始rehash，rehash的目的是防止hash值相同的“链”断开，因为是开放地址法解决的冲突，不rehash的话，可能会导致虽然key在哈希桶中但是却访问不到的情况。
2. rehash每次移动Entry之前会判断它的key是否为null，是的话这就是一个“脏”的Entry，需要清除。
3. rehash的过程也很简单，先判断当前槽中的Entry是否是因为冲突了才存放的这里的，是的话就从根据它的hash值算出的存放位置开始往后找空的槽把它放下。

大致过程如下面图片：
![//TO-DO rehash1的图片](https://img-blog.csdnimg.cn/20210320154748981.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE5MDA3MzM1,size_16,color_FFFFFF,t_70)


![//TO-DO rehash2的图片](https://img-blog.csdnimg.cn/20210320154758728.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE5MDA3MzM1,size_16,color_FFFFFF,t_70)


这篇文章只是大致的解释了```ThreadLocal```为什么会导致内存泄漏以及内存泄漏后如何处理这一块也没有深入的分析，有兴趣的小伙伴可以研究一下```cleanSomeSlots```方法以及执行往```ThreadLocalMap```中添加元素或者删除元素时，```ThreadLocalMap```具体是如何清除“脏“的Entry的，这篇文章不再做分析。