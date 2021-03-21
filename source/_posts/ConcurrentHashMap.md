---
title: ConcurrentHashMap
date: 2021-03-21 17:56:11
tags: ['HashMap', 'JDK']
categories: Java
---

## ConcurrentHashMap

----------
本文讨论的是1.8版本的代码,如果存在错误，欢迎指出。

### 问题

首先在多线程环境下，会出现`读-写`、`读-读`和`写-写`操作冲突。

其次对于`Map`中的常用API：

写:
```Java
{
    V put(K key, V value);

    V remove(Object key);
}
```

读：
```Java
{
    int size();

    boolean isEmpty();

    boolean containsKey(Object key);

    V get(Object key);

    Set<K> keySet();

    Set<Map.Entry<K, V>> entrySet();

    Collection<V> values();
}
```

这些API中大致可以了解到`ConcurrentHashMap`需要维护三类数据的同步
1. size
2. Entry
3. EntrySet

接下来我们一个一个讨论。

## size

可以先看下`ConcurrentHashMap`的关于数量的成员变量：
```Java
{
    //基础计数器 volatile
    private transient volatile long baseCount;
    //lockObject volatile
    private transient volatile int cellsBusy;
    //出现冲突后的计数器 volatile
    private transient volatile CounterCell[] counterCells;

    @sun.misc.Contended 
    static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }
}
```
计数总体上是依靠`CAS`完成数据的更新。
1. 首先更新`baseCount`
2. 如果更新出现冲突，则初始化`countCells[]`数组，将冲突的可能性降低
3. 获取Map中元素数量时，返回`baseCount`和`countCells[]`的总和；

### size增加的情况
通过`put()`增加元素
```Java
{
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        //**********
        //先忽略其他代码
        //下面的方法会将总数量+1， binCount和扩容有关，可以先忽略
        addCount(1L, binCount);
        return null;
    }

    private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            //1.counterCells已经初始化
            //2.更新baseCount失败（发送冲突）
            //上面任何一种情况发生，都会进入到这里 
            CounterCell a; long v; int m;
            boolean uncontended = true;
            // ThreadLocalRandom.getProbe() 获取线程的随机数，用于分散锁冲突
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                // 发生下面的情况
                //1.counterCells未初始化
                //2.counterCells[index] 没有初始化
                //3.counterCells[index] 更新失败
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            //扩容逻辑，先暂时忽略
        }
    }

    private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
        if ((h = ThreadLocalRandom.getProbe()) == 0) {
            //当前线程的随机数还没有初始化
            //进行初始化
            ThreadLocalRandom.localInit();      // force initialization
            h = ThreadLocalRandom.getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            CounterCell[] as; CounterCell a; int n; long v;
            if ((as = counterCells) != null && (n = as.length) > 0) {
                //counterCells[] 已经初始化
                if ((a = as[(n - 1) & h]) == null) {
                    // counterCells[index]没有初始化
                    if (cellsBusy == 0) {            // Try to attach new Cell
                        CounterCell r = new CounterCell(x); // Optimistic create
                        if (cellsBusy == 0 && U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                            // 只有拿到了cellsBusy的锁(CAS)才能进入这里
                            boolean created = false;
                            try {               // Recheck under lock
                                CounterCell[] rs; int m, j;
                                // 针对counterCells[index]没有初始化，创建新的CounterCell
                                if ((rs = counterCells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                //创建成功，是否锁
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            //竞争cellsBusy锁失败，重试
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                //CAS失败，第一次肯定会跳过
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                //直接尝试在counterCells[index]增加数量
                else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                    break;
                //更新counterCells[index]失败，说明又出现了冲突
                else if (counterCells != as || n >= NCPU)、
                    //如果counterCells扩容过 或者 counterCells数量大于CPU数量，则认为不需要扩容了
                    collide = false;            // At max size or stale
                else if (!collide)
                     //到了这一步说明需要继续扩容counterCells
                    collide = true;
                else if (cellsBusy == 0 &&
                         U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                    //对counterCells进行扩容，需要拿到CELLSBUSY的锁
                    try {
                        if (counterCells == as) {// Expand table unless stale
                            CounterCell[] rs = new CounterCell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            counterCells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                //重新设置线程的随机数
                h = ThreadLocalRandom.advanceProbe(h);
            }
            else if (cellsBusy == 0 && counterCells == as &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                //counterCells[] 没有初始化，进行初始化
                boolean init = false;
                try {                           // Initialize table
                    if (counterCells == as) {
                        CounterCell[] rs = new CounterCell[2];
                        rs[h & 1] = new CounterCell(x);
                        counterCells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            //如果都失败了，则回退到更新 BASECOUNT
            else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                break;                          // Fall back on using base
        }
    }
}
```
上面的代码还是比较简单的，
- 首先通过`CAS`更新`baseCount`的值,如果
- - 更新成功，代码执行完成
- - 更新失败，认为发生了冲突。然后会获取`cellsBusy`锁，初始化`countCells[]`数组
- 下次再次增加元素时，由于`countCells[]`数组已经初始化，一般不会更新`baseCount`的值了
- - 选择`countCells[]`一个槽位（线程随机数）进行增加操作（这样减少了冲突的概率）
- - - 如果当前槽位没有初始化，则会获取`cellsBusy`锁，进行初始化
- - - `cas`当前槽位的值进行增加操作
- - - - 更新成功，退出代码
- - - - 更新失败，说明出现了冲突，进行扩容操作（`countCells[]`数组长度最多为大于CPU核数的最近一个2^n的一个数）

在这里`cellsBusy`的作用是用于初始化和创建CounterCell的锁

### size减少的情况
通过`remove()`移除元素
```Java
{
    public V remove(Object key) {
        return replaceNode(key, null, null);
    }

    final V replaceNode(Object key, V value, Object cv) {
        //******
        //忽略其他代码
        //移除元素的逻辑和👆上面是一致的
        addCount(-1L, -1);
        //******
    }
}
```
### size计算
```Java
{
    public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }

    final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
}
```

这里的逻辑也很简单：
1. 元素数量 = `baseCount` + `counterCells[]`中的每一个计数
2. 如果`counterCells[]`没有初始化，那么`baseCount`是通过`CAS`增加的，就是现在状态下的元素数量；
3. 这里有一个问题是代码中没有对于`counterCells[]`的访问加锁，计算数量期间可能又会有元素被添加，所以这种情况下返回的总数是不准确的，只是一个估算值。

//TODO