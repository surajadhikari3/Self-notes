Refer to this site too --> https://www.digitalocean.com/community/tutorials/gangs-of-four-gof-design-patterns

Creational
Structural
Behavioural


Prototype pattern is the creational pattern for creating the objects by cloning the object, known as prototype

can be Used  in the graphics based applications like creating the game character using the same shape

Concepts : Can be done by shallow copy and deep copy of the objects.(By default it is shallow copy..)

Shallow copy : copies the high level object but for the nested objects only the reference is copied so the changes in copy reflect to original object and vise-versa.

Deep copy: copies entire object along with the nested object pointing completly to the another memory reference with proper isolation 

Implements the Cloneable interface which is marker interface..

```
class Address {
    String city;
    Address(String city) {
        this.city = city;
    }
}

class Person implements Cloneable {
    String name;
    Address address;

    Person(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    public Person clone() throws CloneNotSupportedException {
        return (Person) super.clone();  // Shallow copy
    }
}

```

```
class Address {
    String city;
    Address(String city) {
        this.city = city;
    }

    public Address clone() {
        return new Address(this.city);  // Deep copy of nested object
    }
}

class Person implements Cloneable {
    String name;
    Address address;

    Person(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    public Person clone() {
        return new Person(this.name, this.address.clone());  // Deep copy
    }
}

```

## Summary Table:

| Feature        | Shallow Copy                                             | Deep Copy                            |
| -------------- | -------------------------------------------------------- | ------------------------------------ |
| Copies object? | Yes                                                      | Yes                                  |
| Nested objects | References copied                                        | Nested objects are also cloned       |
| Speed          | Faster                                                   | Slower                               |
| Memory         | Lower                                                    | Higher                               |
| Use case       | When objects are immutable or shared references are fine | When complete independence is needed |

Builder Pattern --> Study from the java-problems code and code once

Factory Pattern -->The factory pattern takes out the responsibility of instantiating a object from the class to a Factory class. The interface is defined and the classes which implement the interface are returned with its instance based on the value you passed.

Do coding once check the java-problems factory design...

Structural Design pattern

Adapter --> Provides an interface betn two unrelated entities so thay they can work together..(For running or supporting the legacy code/ Old APIs, ) --> Ensures the compatability without the code changes........

simple analogy is using the adapter conver the usb to type c..

 Facade -->  it simplifies the complex system by exposing a unified interface..

```
// Complex subsystems
class CPU { void start() { System.out.println("CPU started"); } }
class Memory { void load() { System.out.println("Memory loaded"); } }
class Disk { void read() { System.out.println("Disk read"); } }

// Facade
class ComputerFacade {
    private CPU cpu = new CPU();
    private Memory memory = new Memory();
    private Disk disk = new Disk();

    public void startComputer() {
        cpu.start();
        memory.load();
        disk.read();
    }
}

// Usage
public class FacadeExample {
    public static void main(String[] args) {
        ComputerFacade computer = new ComputerFacade();
        computer.startComputer();
    }
}

```



Proxy
Decorator --> Add new behaviors to an object dynamically without modifying its code.

```
interface Coffee {
    String getDescription();
    double getCost();
}

class BasicCoffee implements Coffee {
    public String getDescription() { return "Basic Coffee"; }
    public double getCost() { return 5.0; }
}

class MilkDecorator implements Coffee {
    private Coffee coffee;

    public MilkDecorator(Coffee coffee) { this.coffee = coffee; }

    public String getDescription() { return coffee.getDescription() + ", Milk"; }
    public double getCost() { return coffee.getCost() + 1.5; }
}

// Usage
public class DecoratorExample {
    public static void main(String[] args) {
        Coffee coffee = new MilkDecorator(new BasicCoffee());
        System.out.println(coffee.getDescription() + " $" + coffee.getCost());
    }
}

```

Comparison

| Feature        | **Facade**                       | **Decorator**                       | **Adapter**                                 |
| -------------- | -------------------------------- | ----------------------------------- | ------------------------------------------- |
| Purpose        | Simplifies complex subsystems    | Adds new functionality dynamically  | Converts one interface into another         |
| Use-case       | Hide complexity                  | Extend object behavior              | Compatibility with legacy/incompatible APIs |
| Inheritance?   | Usually no                       | Can be interface-based              | Can involve composition or inheritance      |
| Real-World Use | Spring’s `JdbcTemplate` for JDBC | Java I/O Streams (`BufferedReader`) | `Arrays.asList()` adapter for array to List |

Behavioural pattern 

Template method --> Spring uses the template method




-------------------



Great one 👍 — interviewers love this because it checks if you **really** understand object references in Java. Let’s go step by step:

---

## 1. **Shallow Copy**

- Copies the **top-level object** but **not the nested objects** it refers to.
    
- Only the **references** of child objects are copied, not their actual data.
    
- So, the original and copy share the **same nested objects**.
    
- If you change a nested object in the copy, it also changes in the original (and vice versa).
    

Example:

```java
class Address {
    String city;
    Address(String city) { this.city = city; }
}

class Person implements Cloneable {
    String name;
    Address address;
    Person(String name, Address address) {
        this.name = name;
        this.address = address;
    }
    public Object clone() throws CloneNotSupportedException {
        return super.clone(); // shallow copy
    }
}

public class Test {
    public static void main(String[] args) throws CloneNotSupportedException {
        Person p1 = new Person("Alice", new Address("Toronto"));
        Person p2 = (Person) p1.clone();

        p2.address.city = "Ottawa";   // changes both p1 and p2
        System.out.println(p1.address.city); // Ottawa
    }
}
```

---

## 2. **Deep Copy**

- Copies the **top-level object AND all nested objects** (recursively).
    
- The copy is **completely independent** of the original.
    
- Changes in one object’s nested fields won’t affect the other.
    

Example:

```java
class Address implements Cloneable {
    String city;
    Address(String city) { this.city = city; }
    public Object clone() throws CloneNotSupportedException {
        return new Address(this.city); // deep copy of Address
    }
}

class Person implements Cloneable {
    String name;
    Address address;
    Person(String name, Address address) {
        this.name = name;
        this.address = address;
    }
    public Object clone() throws CloneNotSupportedException {
        return new Person(this.name, (Address) this.address.clone()); // deep copy
    }
}

public class Test {
    public static void main(String[] args) throws CloneNotSupportedException {
        Person p1 = new Person("Alice", new Address("Toronto"));
        Person p2 = (Person) p1.clone();

        p2.address.city = "Ottawa";   
        System.out.println(p1.address.city); // Toronto (independent copy)
    }
}
```

---

## 3. **Summary Table**

|Feature|Shallow Copy|Deep Copy|
|---|---|---|
|Top-level fields|Copied|Copied|
|Nested objects|Only references copied (shared)|Entire objects recursively copied|
|Independence|❌ Not independent (changes reflect)|✅ Independent (changes isolated)|
|Performance|Faster (less work to do)|Slower (more memory & processing needed)|

---

## 4. **Layman Analogy**

Imagine you photocopy a **blueprint of a house**:

- **Shallow copy** → You copy the paper, but both copies point to the _same house_. If the house is repainted, both blueprints now show a different reality.
    
- **Deep copy** → You not only copy the blueprint but also **build a new house** exactly like the original. Now each house is independent — repainting one doesn’t affect the other.
    

---

Summary (September 12)

--> Creational  Pattern -> Builder pattern (Fixed Telescopic constructor issue), Singleton (simple, Block level synchronize with double null check and volatile keyword, BillPugh Singleton --> static inner class -> for lazy loading and static final variable only instantiate the variable once without synchronized keyword........................... ) 

--> Structural ( Facade --> For making the complex implementation exposing in simpler interface often done by frameworks, eg JDBCTemplate, Decorator pattern --> Adding functionality on the base classes instead of writing from scratch; eg using the BlackCoffee to make the MilkCoffee with sugar,  Adapter pattern is for adaption ensuring the compatibility while working with legacy based, external libraries with out making the code change....  )


Behavioural


--> Template method (Heavily used in spring , JMS template,  JDBC template, Kafka Template,  RestTemplate, HibernateTemplate,  WebClient , MongoTemplate, )




