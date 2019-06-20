## LRU(LinkedHashMap实现)队列使用不当
 - 问题场景
 - 问题发现
 - 问题解决
 - 问题回顾
 #### 问题场景
   使用LinkedHashMap实现LRU队列，保证队列长度为1024。由于LinkedHashMap不是线程安全，采用了ReentrantReadWriteLock对put()/get()方法进行了加锁操作。
   LruCache.java
  
```black
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Created by yahto on 2019-06-20 15:19
 *
 * @author yahto
 */
public class LruCache<K, V> extends LinkedHashMap<K, V> {

	private final int maxCacheSize;

    public LRUCache(int initialCapacity) {
        super((int) Math.ceil(initialCapacity / 0.75) + 1,
                0.75f, true);
        maxCacheSize = initialCapacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxCacheSize;
    }
}
```
LruOpsUtil.java

```black
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * Created by yahto on 2019-06-20 15:27
 *
 * @author yahto
 */
@Slf4j
public class LruOpsUtil<K, V> {
    private LruCache<K, V> cache = new LruCache<>(1024);

    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public V put(K key, V value) {
        V val = null;
        try {
            //get write lock
            lock.writeLock().lock();
            val = cache.put(key, value);
        } catch (Exception e) {
            log.error("put val error", e);
        } finally {
            lock.writeLock().unlock();
        }
        return val;
    }

    public V get(K key) {
        V val = null;
        try {
            //get read lock
            lock.readLock().lock();
            val = cache.get(key);
        } catch (Exception e) {
            log.error("put val error", e);
        } finally {
            lock.readLock().unlock();
        }
        return val;
    }

}
```
 项目中因为token加解密算法耗时，在token解密之前先去LRU队列里面查找是否有解密过后的token对象。  
   LruCache<String,Token> cache;  
   Key为Token加密后的字符串,Token为解密后的实例(平均大小为0.23M)
 #### 问题发现
   在性能环境中，发现内存不断增长，JVM 24G 内存，在两天内就被占满，导致虚拟机大部分时间都在做 GC 操作，导致服务器响应客户端的线程特别慢，并且 GC 效率不高，每次都只能GC出0.1%左右的空间。
   
   通过查看堆栈快照，发现堆栈中绝大部分对象都是Token，数量级高达10^6，占了JVM内存的97%。但是设置的LRU队列为1024长度，理应不该出现数据量这么大。
   
   于是跟踪put()方法，由于用了写锁，并发出2000个线程，当LRU长度为1024，下一个入队进来的token对象会将对头的token剔除。保证了队列长度只为1024个。
fuck me！！！！！岂不是理论上没有bug了？？？？？？？ 但是他妈的又出现了10^6数量级的token对象。我已经准备好删库跑路了(rm -rf /*)
  
  午饭过后，阳光明媚，一支玉溪过肺。太阳和头发在对我笑。
  
  反正没事(要被动跑路了？)，就来看看get()方法吧。fuck me again ！ ！ 一看不打紧，有问题呀。
  
LinkedHashMap里的get()方法
```black
public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder) //accessOrder设置成true时,为最近最少使用（LRU）算法实现,accessOrder设置成false时,为先进入先过期
            afterNodeAccess(e);
        return e.value;
    }
```

我他妈就是需要最近最少使用啊,肯定设置为true了啊。

  ok 那就看看afterNodeAccess(Node e)这个东西吧。
  

```black
void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

  还行，就是把e节点"移到队列最后面"。问题不大，问题不大。just fuck me one more time！put()方法是个读锁，允许多个线程访问的，多个线程对双向链表操作，搞出大事情了。原因是链表的"移到队列最后面"都是要动的token引用，就乱序了，一个队列不晓得被整成了什么样子，感觉出现环都有可能。还好没把CPU跑死。

 #### 问题解决
 解决是不可能解决了
