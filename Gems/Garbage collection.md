
**Garbage collector  
  
  
**[**https://github.com/FullStackLabsCa/bootcamp2024-material/blob/main/lecture33-garbage-collection-notes.md**](https://github.com/FullStackLabsCa/bootcamp2024-material/blob/main/lecture33-garbage-collection-notes.md)

-Xms (minimum) and -Xmx (maximum) flag for the  
  
**java -Xms 512m -Xmx2048m MyApplication  
  
this allows to start the java program with 512 mb heap and allows it grow up to 2gb**  

  
objects become eligible for gc —> when it is no longer reachable via any strong references from the root (like static fields,  stack vaiables)..  

**Tricky question —> how does the garbage collection take place in the stack memory  
  
No garbage collection needed — memory is popped when a method returns, thread finished it s execution..  
  
Where it physically resides:**

➔ **Also inside the process’s address space, but managed separately by the JVM.**

➔ **It’s typically a small fixed region of RAM per thread.**

➔ **Each thread gets its own private stack.  
  
  
Xss -> stack size  
  
Xms -> Heap initial size and Xmx -> is the heap max size   
  
If they ask how can you increase  the more number of threads then  
  
First it is limited by the ulimit of the os —> increase the ulimit with ulimit -u your-number…  
  
Second one can decrease the per thread stack size with the -Xss this way overall thread creation will increase by lightweight threads…**

|                            |                                                             |
| -------------------------- | ----------------------------------------------------------- |
| **Heap**                   | **Stack**                                                   |
| Dynamic memory for objects | Memory for method calls,local variables and thread creation |
| Shared among all threads   | Private to each thread                                      |
| Needs Garbage Collection   | Frees up automatically                                      |
| Larger (MBs to GBs)        | Smaller (KBs to MBs)                                        |
| -Xms, -Xmx options         | -Xss option                                                 |

  
  
**Simple Mental Model:**

|             |                                  |                           |
| ----------- | -------------------------------- | ------------------------- |
| **Type**    | **Behavior**                     | **Example Use Case**      |
| **Strong**  | Never GC’ed while referenced     | Normal code               |
| **Soft**    | GC’ed only if memory is low      | Image cache, object cache |
| **Weak**    | GC’ed in next GC cycle           | WeakHashMap, listeners    |
| **Phantom** | After finalize, cleanup tracking | Resource cleanup          |
![[Pasted image 20250429160641.png]]

Default GC for now is the G1 GC ( region based and low pause time )


- **ZGC** and **Shenandoah** collectors introduced (low-latency GCs).
    
- Not the default, but you can enable them explicitly:
    
    - `-XX:+UseZGC`
        
    - `-XX:+UseShenandoahGC`


| Java Version | Default GC  |
| ------------ | ----------- |
| Java 8       | Parallel GC |
| Java 9–10    | Parallel GC |
| Java 11+     | G1 GC       |
| Java 17      | G1 GC       |
| Java 21      | G1 GC       |
Got it 👍 — let’s structure this in a **hierarchical way** so it’s super clear for interviews.

---

## **Heap Layout in G1 GC (High Level → Detailed)**

### 1. **Young Generation**

- Where new objects are allocated.
    
- Collected frequently with **Minor GCs** (short pauses).
    
- Subdivided into:
    
    - **Eden regions** → fresh allocations (short-lived).
        
    - **Survivor regions (S0, S1)** → objects that survive a collection in Eden get copied here.
        
        - Ping-pong between S0 and S1 on each cycle.
            

---

### 2. **Old Generation**

- Where **long-lived objects** eventually end up.
    
- Collected less frequently, often via **Mixed GCs** (collect some old + young regions).
    
- Subdivided into:
    
    - **Old regions** → promoted survivors.
        
    - **Humongous regions** → very large objects (≥ 50% of region size). Allocated directly here.
        

---

### 3. **Free Regions**

- Unused regions, available for allocation as Eden/Survivor/Old when needed.
    

---

## **Hierarchy View**

```
Heap
├── Young Generation
│   ├── Eden regions
│   └── Survivor regions (S0, S1)
│
├── Old Generation
│   ├── Old regions
│   └── Humongous regions
│
└── Free Regions
```

---

💡 **Interview phrasing:**  
👉 _“In G1 GC, the heap is divided into equal-sized regions. At a high level, these regions are grouped into Young (Eden + Survivors), Old (long-lived + Humongous), and Free regions. Unlike older collectors, the boundaries are not fixed — regions can change roles dynamically depending on GC activity.”_

---

Do you want me to now create a **colored diagram** matching this hierarchy (Young vs Old vs Free, with subdivisions) so you can use it visually in prep?