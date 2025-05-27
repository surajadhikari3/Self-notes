
Java 8 nunace

  
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

