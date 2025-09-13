
Java 8 nunace

https://chatgpt.com/share/68489c7a-0a08-8007-9862-e2c71d56322a --> See this to grook around.....

  
Collectors.groupBy vs Collectors.toMap() —> take function, function and need the merger if there is repetition in the keys

boxed() vs mapToObj()

Know about the regular expression. 

skip()

distinct()

anyMatch()

box()

noneMatch()

Collectrors.groupingBy() —> Most useful while grouping in the map. By default it will collect as a list 

—> Additionally there is Collectors.count() which will keep the frequency of the elements or we can collect as a list and checks its size which is the similar approach..

  

Collectors.partitionBy() —> it will take predicate and do the collection based on it..

Collectors.joining() -> It concats the character or string to the string and u can pass the delimiter to it  for the expected output as you want..

  
Keep in mind  
  
Boxing —> while performing the arithmetic operations it is better to convert the wrapper class to the primitive type so that the memory is preserved the operation 

is faster too….

  

Primitive operations 

  

Middle operations..  
  
limit()

skip()

distinct()

sorted()


Terminal Operation …

sum()

average()

count()

reduce()

min()

max()

  
If we have to add some delimiters or format in the streams we can use the Collectors.joining()  
  
Collectors.joining()                     // joins without any delimiter  

Collectors.joining(delimiter)             // joins with a delimiter  

Collectors.joining(delimiter, prefix, suffix) // joins with delimiter, prefix, suffix  
  
String result = names.stream()

                     .collect(Collectors.joining(" , " , "[",  "]"));

System.out.println(result); // [Alice, Bob, Charlie]  
  
# For sorting the map(Specially in the reverse order) there is two approach 

 .sorted(Map.Entry.<Integer, Long>comparingByValue().reversed())

   .sorted(Comparator.comparing((Map.Entry<Integer, Long> x) -> x.getValue()).reversed())  

.sorted(comparingByValue(Comparator.reverseOrder()));


Concepts ….  

Stream once consumed can not be reused so we use the supplier which is the factory to create the Stream  
  
  
Supplier provides the get method to access the element which it stores. ……  
  
Some concepts while working in programming  
  
  
Need to know things ..  
  
1. Substring in string , startsWith() —> Need to know the  few methods of the string..   
2 . While filtering —> filter(x-> someList.contains(x)) —> this is solid way to find the  common elements in two arrays…

3 For finding the repeated and not repeated character consider the  index.of(‘character’) != char.lastIndexof() —> Always check the index and the lastIndexOf  

Also need to know the basic about regex..  
  
  
Look this video for the common use case  
[https://www.youtube.com/watch?v=jky2GNyODFs&list=PL63BDXJjNfTElajNCfg_2u_pbe1Xi7uTy&index=48&ab_channel=code_period](https://www.youtube.com/watch?v=jky2GNyODFs&list=PL63BDXJjNfTElajNCfg_2u_pbe1Xi7uTy&index=48&ab_channel=code_period)

If. we have to add the collections like set inside the map or to some mapping while collecting we can use the Collectors.mapping lijke

```java
Collectors.groupingBy(  
        Task::getEmployeeId,  
        Collectors.mapping(Task::getStatus, Collectors.toSet()))
```


Full-Code 

```java
Set<TaskStatus> allStatus = EnumSet.allOf(TaskStatus.class);  

return tasks.stream()  
        .collect(Collectors.groupingBy(  
                Task::getEmployeeId,  
                Collectors.mapping(Task::getStatus, Collectors.toSet())))  
        .entrySet()  
        .stream()  
        .filter(entry -> entry.getValue().containsAll(allStatus))  
        .map(Map.Entry::getKey)  
        .collect(Collectors.toList());
```

Important second arguments that can be passed after the Collectors.groupingBy()

Collectors.toMap()
Collectors.summingToInt()
Collectors.counting()


---
## ✅ 1. `Collectors.toMap()` – Use When You Want a Simple Map

> Use it when you want to convert a **stream of elements into a Map** with **one unique value per key**.


```java
Collectors.toMap(
    keyMapper,       // how to get the key
    valueMapper,     // how to get the value
    mergeFunction,   // what to do if key already exists (take old one, take new one or merge both)
    mapSupplier      // which kind of Map to create
)
```

---
```java
tasks.stream().collect(Collectors.toMap(
    Task::getId,
    Function.identity(),
    (a, b) -> b,
    LinkedHashMap::new //For mantaining the order or TreeMap for sorting
));

```

mergeFunction --> Means what to do to the value if the same records occur either merge it , keep old one or keep new one

## 🎯 What `(a, b) -> b` means:

> “If two values have the same key, keep the **second one** (i.e., the new one).”

- `a`: existing value already in the map
    
- `b`: new value that also wants to go into the map with the same key
    

👉 `(a, b) -> b` means **replace the old value with the new one**.

---

### ✅ Example:

Suppose you have these two tasks with the **same ID** `"T1"`:

```java
new Task("T1", "E1", "HR", date1, 10, TaskStatus.COMPLETED)
new Task("T1", "E2", "HR", date2, 20, TaskStatus.FAILED)
```

Now this collector:

```java
tasks.stream().collect(Collectors.toMap(
    Task::getId,              // key = "T1"
    Function.identity(),      // value = Task object
    (a, b) -> b               // if "T1" already exists, keep b (the newer one)
));
```

🔁 When the second "T1" comes:

- Old value `a`: Task("T1", "E1"...)
    
- New value `b`: Task("T1", "E2"...)
    

💡 Since you said `(a, b) -> b`, it replaces the old task with the new one.

---

## 🛑 What if you don’t provide a merge function?

You’ll get an **`IllegalStateException`** if there are duplicate keys.

```java
tasks.stream().collect(Collectors.toMap(
    Task::getId,
    Function.identity()
));
```

👆 This will fail if there are multiple tasks with the same ID.

---

## ✅ Other examples of merge functions

### 1. Keep the first value (ignore duplicate):

```java
(a, b) -> a
```

### 2. Combine durations (if values are integers or longs):

```java
(a, b) -> a + b //Similar to map.merge(key, value, Integer::sum)
```
---

## 🧠 Analogy:

Imagine filling a spreadsheet where the **Task ID is the row key**.

- If a second task tries to write into the same row (duplicate key), you must decide:
    
    - Overwrite? → `(a, b) -> b`
        
    - Ignore the new one? → `(a, b) -> a`
        
    - Merge both values? → `(a, b) -> mergeThem(a, b)`
        

---

## ✅ 2. `Collectors.groupingBy()` – Use When You Want to Group Elements

> Use it when you want to **group stream elements by a classifier function** (i.e., group them into buckets).

### 🔹 Syntax:

```java
Collectors.groupingBy(classifier) // returns Map<K, List<T>>
```

Optionally:

```java
Collectors.groupingBy(classifier, downstream collector)
Collectors.groupingBy(classifier, map supplier, downstream collector)
```

### 🔹 Example Use Cases:

#### A. Group tasks by department → `Map<String, List<Task>>`

```java
tasks.stream().collect(Collectors.groupingBy(Task::getDepartment));
```

#### B. Group tasks by status and count → `Map<Status, Long>`

```java
tasks.stream().collect(Collectors.groupingBy(Task::getStatus, Collectors.counting()));
```

#### C. Group tasks by department, and then by date

```java
tasks.stream().collect(Collectors.groupingBy(
    Task::getDepartment,
    Collectors.groupingBy(Task::getDate)
));
```

#### D. Grouping with summing → Total duration per employee

```java
tasks.stream().collect(Collectors.groupingBy(
    Task::getEmployeeId,
    Collectors.summingInt(Task::getDuration)
));
```

---

## 🧠 Summary Table

| Collector              | Use Case                                                | Returns                        |
| ---------------------- | ------------------------------------------------------- | ------------------------------ |
| `toMap(k, v)`          | Unique key-value mapping                                | `Map<K, V>`                    |
| `toMap(k, v, mergeFn)` | Merge duplicate keys                                    | `Map<K, V>`                    |
| `toMap(k, v, m, s)`    | Custom map implementation (e.g., TreeMap)               | Custom `Map<K, V>`             |
| `groupingBy(f)`        | Group by a classifier, collect to list                  | `Map<K, List<T>>`              |
| `groupingBy(f, c)`     | Group by classifier, apply collector (sum, count, etc.) | `Map<K, R>` (R is result type) |
| `groupingBy(f, m, c)`  | Group with custom map and collector                     | Custom `Map<K, R>`             |

---

## 👨‍🏫 Layman Analogy

- **`toMap`** is like making a phonebook: one person → one number.
    
- **`groupingBy`** is like organizing a classroom by subject: multiple students → per subject.
    

---

## 🔄 Common Patterns

### 🔁 Combine `groupingBy` + `mapping`

```java
tasks.stream().collect(Collectors.groupingBy(
    Task::getDepartment,
    Collectors.mapping(Task::getTaskId, Collectors.toSet())
));
```

### 🔁 Group and find max

```java
tasks.stream().collect(Collectors.groupingBy(
    Task::getDepartment,
    Collectors.collectingAndThen(
        Collectors.maxBy(Comparator.comparingInt(Task::getDuration)),
        Optional::get
    )
));
```

Little more on the regex or while reading the paragraphs ..

Every time if you want to split with the whitespace including the space, multiple space , newline and tab
instead of using the .split(" ") use the

	`inputString.split("\\s+")`

.

---

### 🧠 Quick Comparison Table:

| Feature                 | `" "`                   | `"\\s+"`                             |
| ----------------------- | ----------------------- | ------------------------------------ |
| Splits on               | Exactly one space       | Any whitespace (space, tab, newline) |
| Handles multiple spaces | ❌ Creates empty strings | ✅ Ignores them gracefully            |
| Trims tabs/newlines     | ❌ No                    | ✅ Yes                                |
| Ideal for real text     | ❌ No                    | ✅ Yes                                |


### ✅ How to Remove Punctuation as Well?

You want **clean words** without punctuation like commas, periods, exclamation marks, etc.

#### ✅ Option 1: Clean before splitting

```java
String cleaned = paragraph.replaceAll("[^a-zA-Z0-9\\s]", "");  // removes punctuation
String[] words = cleaned.split("\\s+");
```

- `[^\w\s]` or `[^a-zA-Z0-9\\s]` means "anything that's not a word character or whitespace". (String s1 = s.replaceAll("[^\\w\\s]", "");)
    
- This removes `.,!?:;"'()[]{}` etc.
    

#### ✅ Option 2: Clean after splitting

If you still want to split first, then sanitize each word:

```java
String[] words = paragraph.split("\\s+"); //splitting on the basis of whitespace
List<String> cleanWords = Arrays.stream(words)
    .map(word -> word.replaceAll("[^a-zA-Z0-9]", "")) // remove punctuation per word
    .filter(word -> !word.isEmpty()) // skip empty strings
    .collect(Collectors.toList());
```

---

### ✅ Example Code:

```java
String paragraph = "Hello, world! Java is great. Let's code.";
List<String> words = Arrays.stream(paragraph.split("\\s+"))
    .map(w -> w.replaceAll("[^a-zA-Z0-9]", ""))
    .filter(w -> !w.isEmpty())
    .collect(Collectors.toList());

System.out.println(words);
```

🧾 Output:

```
[Hello, world, Java, is, great, Lets, code]
```

---

### 🧠 Analogy:

Think of punctuation as "noise" between words. `split("\\s+")` only removes **gaps between words**, not **the noise stuck to the words**. So you need an extra cleanup step to brush that noise away.

---

Let me know if you want to preserve things like apostrophes (`don't`, `it's`) or handle multilingual content — we can fine-tune the regex!


Note:

While doing the flatMap it moves and loses the content if we want the context then we have to stay 
within the scope with the bracket like below  and can use the `AbstractMap.SimpleEntry<>(key, value)` to store the info in key value pair... :

```java
 departments.stream()  
        .flatMap(department -> department.getEmployees().stream())  
	        .flatMap(employee -> employee.getProjects().stream()  //Note --> To get the details within the flatmap we                                                                     should be within the flatmap scope and map and                                                                        collect to the abstractMap .map(Project::getName) 
                .distinct()  
                .map(projectName -> new AbstractMap.SimpleEntry<>(projectName, employee.getId())))  
        .distinct()  
        .collect(Collectors.groupingBy(  
                Map.Entry::getKey,  
                Collectors.counting()  
        ));**
```



---Summary

Collectors.groupBy(x-> x.getName, TreeMap::new, Collectors.toList())

Collectors.toMap(key, value, (a,b) -> b,  LinkedHashMap::new)

Collectors.grouping(key, value)


