
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