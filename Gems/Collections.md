
#### **What is the Java Collections Framework (JCF)?**

Java Collections Framework is a set of **interfaces and classes** that support operations on **data structures like lists, sets, queues and maps**. It provides algorithms and utilities for sorting, searching, and more.

Difference between ArrayList and LinkedList...

| Feature            | ArrayList                   | LinkedList                  |
| ------------------ | --------------------------- | --------------------------- |
| Backed By          | Dynamic Array               | Doubly Linked List          |
| Insertion/Deletion | Slower O(n) due to shifting | Faster at ends 0(1)         |
| Access (get)       | Fast 0(1)                   | Slow O(n)                   |
| Memory             | Less memory overhead        | More (due to node pointers) |
Use the Arraylist for the heavy read operation
Use LinkedList for frequent inserts/removals....


Main interfaces in Collection Framework?

List --> ArrayList, LinkedList
Set --> HashSet, TreeSet, LinkedHashSet
Queue --> PriorityQueue, Deque

Map(is also a Interface) is not a subtype of Collection in jaca


![[Pasted image 20250512162816.png]]#### **Difference between HashSet and TreeSet?**

| Feature     | HashSet                  | TreeSet                        |
| ----------- | ------------------------ | ------------------------------ |
| Order       | No order                 | Sorted (natural or comparator) |
| Performance | Faster (uses hash table) | Slower (uses Red-Black Tree)   |
| Nulls       | Allows one null          | No null elements allowed       |

HashMap Vs HashTable 

HastTable implements Map<K, V> interface 

|Feature|HashMap|Hashtable|
|---|---|---|
|Thread Safety|Not synchronized|Synchronized (Legacy)|
|Null Keys/Val|Allows one null key|No null key/value|
|Performance|Faster|Slower|
-> Concept if it is not thread safe and normal it is faster and allowed the null 
    but if it is thread safe or ordered then it is slow and does not allowed null.
# How does HashMap work internally?

- Uses an **array of buckets** where each bucket is a **linked list (or tree in Java 8+)**.
    
- Stores **key-value pairs** using the key's **hashCode()** to find the bucket.
    
- If keys collide (same hash), uses chaining (linked list or tree).
    
- `equals()` is used to resolve key equality in a bucket.

**Java 8 Optimization**: If one bucket becomes too long, the linked list turns into a **balanced tree (Red-Black Tree)** for faster access.

Why do we override equals() and hashCode() ?

-> To ensure that keys work correctly in hash-based collections like (HashMap and HashSet).
-> If equals() is overriden, but hashCode() is not, two equal objects might end up in different   buckets.

📌 Rule:

- Equal objects must have **same hashCode**.
    
- Unequal objects can have same or different hashCode.


#### 8. **What is ConcurrentHashMap? How is it different from HashMap?**

|Feature|HashMap|ConcurrentHashMap|
|---|---|---|
|Thread-safe|❌ No|✅ Yes|
|Synchronization|❌ None|✅ Uses segments/locks (Java 7) or CAS (Java 8)|
|Null keys/values|✅ Allowed|❌ Not allowed|

🎯 Use `ConcurrentHashMap` in **multi-threaded** environments for high concurrency and thread safety.


#### **How does CopyOnWriteArrayList work?**

**Answer**:  

It's a **thread-safe variant** of `ArrayList`. On every write (add, remove), it creates a **new copy** of the underlying array.  

Ideal for **read-heavy** scenarios like caching.