# 1、都知道缓存淘汰，说下LRU怎么构建与原理

    关键点：1、一个最大容量、两种数据结构

​				2、保证都是O（1）的时间复杂度

​				3、上一次访问的元素在最后一个

数据结构：哈希表与双向链表或api可用LinkedHashMap实现.

实现：一个Node类（存储key与val），一个DoubleList类，一个LRUCache类。

```java
public class Node {
    public int key,val;
    public Node next,prev;
    public Node(int k,int v){
        this.key=k;
        this.val=v;
    }
}
```

```java
/**
 * 存储数据
 * 1.正常添加节点与删除
 * 2.重复的key
 * 3.容量满了
 *
 * 返回双向链表的大小
 */
public class DoubleList {

    //双向链表的头节点
   private Node head;
   //双向链表的尾节点
   private Node tail;
   //双向链表的容量
   private int size;

   public DoubleList(){
       //初始化双向链表的数据
       head = new Node(0,0);
       tail = new Node(0,0);
       head.next = tail;
       tail.prev = head;
       //此时形成head->tail的双向链表
       size=0;
   }
    //在链表尾部添加节点x，时间复杂度O(1)
    //此方法结束加入一个节点，形成head—>x1->x2->tail的双向链表
    //并且每次添加新的元素是在双向链表的表尾的,先来的就越接近head
    public void addFirst(Node x){
        x.prev = tail.prev;
        x.next = tail;
        tail.prev.next = x;
        tail.prev = x;
        size++;
    }

    //删除链表中的x节点（x一定存在）
    //由于是双向链表且给的是目标Node节点，时间复杂度为O(1)
    //注意，此双向链表包括两个初始化的头节点与尾节点，删除也只是
    //删除head与tail之间的节点
    public void remove(Node x){
        x.prev.next = x.next;
        x.next.prev = x.prev;
        size--;
    }

    //删除链表中第一个节点，并返回该节点，时间复杂度为O(1)
    //双向链表的第一个节点是head.next
    public Node removeLast(){
        if(head.next == tail){
            return null;
        }
        Node first = head.next;
        remove(first);
        return first;
    }

    //返回链表长度，时间复杂度O(1)
    public int size(){
        return  size;
    }
}
```

```java
public class LRUCache {
    public static void main(String[] args) {
        LRUCache cache = new LRUCache(2);
        cache.put(1, 1);
        cache.put(2, 2);
        System.out.println(cache.get(1));
        cache.put(3, 3);
        //容量是2，插入3后，原先使用过一次1，2这个元素就被删除了
        System.out.println(cache.get(2));
    }

    private int cap;
    DoubleList cache;
    HashMap<Integer,Node> map;

    public LRUCache(int capacity){
        this.cap = capacity;
        map = new HashMap<>();
        cache = new DoubleList();
    }

    public int get(int key){
        //如果key不存在
        if(!map.containsKey(key)){
            return -1;
        }
        //获取最近用的key
        makeRecently(key);
        return map.get(key).val;
    }


    public void put(int key,int val){
        if(map.containsKey(key)){
            //删除旧的数据
            deleteKey(key);
            //新插入的数据为最近使用的数据
            addRecently(key,val);
            return;
        }
        if(cap == cache.size()){
            //删除最久没有使用的元素
            removeLeastRecently();
        }
        //添加最近使用的元素
        addRecently(key,val);
    }


    /*将某个key提升为最近使用的*/
    private void makeRecently(int key){
        Node x = map.get(key);
        //从链表中删除这个节点
        cache.remove(x);
        //重新插入到队尾
        cache.addLast(x);
    }

    /*添加最近使用的元素*/
    private void addRecently(int key,int val){
        Node x = new Node(key,val);
        //链表尾部就是最近使用的元素
        cache.addLast(x);
        //并在map中放入链表的映射
        map.put(key,x);
    }

    /*删除某一个key*/
    private void deleteKey(int key){
        Node x = map.get(key);
        //从链表中删除
        cache.remove(x);
        //从map中删除
        map.remove(key);
    }

    /*删除最久未使用的元素*/
    private void removeLeastRecently(){
        //链表头部的第一个元素就是最久未使用的
        Node deletedNode = cache.removeFirst();
        //同时删除map中的key
        int deletedKey = deletedNode.key;
        map.remove(deletedKey);
    }
}
```

------

**API的LinkedHashMap实现**

```java
public class LRUCache {
    public static void main(String[] args) {
        LRUCache cache = new LRUCache(2);
        cache.put(1, 1);
        cache.put(2, 2);
        System.out.println(cache.get(1));
        cache.put(3, 3);
        //容量是2，插入3后，原先使用过一次1，2这个元素就被删除了
        System.out.println(cache.get(2));
    }

//    private int cap;
//    DoubleList cache;
//    HashMap<Integer,Node> map;
    private int capacity;
    LinkedHashMap<Integer, Integer> cache = new LinkedHashMap<>();

    public LRUCache(int capacity) {
        this.capacity = capacity;
    }

    public int get(int key) {
        if (!cache.containsKey(key)) {
            return -1;
        }
        //将key变为最近使用
        makeRecently(key);
        return cache.get(key);
    }

    public void put(int key, int val) {
        if (cache.containsKey(key)) {
            //修改key的值
            cache.put(key, val);
            //将key变为最近使用
            makeRecently(key);
            return;
        }

        if (cache.size() >= this.capacity) {
            //链表头部就是最久未使用的key
            int oldeskey = cache.keySet().iterator().next();
            cache.remove(oldeskey);
        }
        //将新的key添加到链表尾部
        cache.put(key, val);
    }


    private void makeRecently(int key) {
        int val = cache.get(key);
        //删除key，重新插入到队尾
        cache.remove(key);
        cache.put(key, val);
    }
}
```

# 
