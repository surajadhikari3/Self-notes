Refer to this site too --> https://www.digitalocean.com/community/tutorials/gangs-of-four-gof-design-patterns

Creational
Structural
Behavioural


Prototype pattern is the creational pattern for creating the objects by cloning the object, known as prototype

can be Used  in the graphics based applications like creating the game character using the same shape

Concepts : Can be done by shallow copy and deep copy of the objects.(By default it is shallow copy..)

Shallow copy : copies the primitive value and copies the reference of the object..   (If the reference is changed it will also change outside..)
Deep copy: copies entire object keeping its own reference..

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

Adapter --> Provides an interface betn two unrelated entities so thay they can work together..(For running or supporting the legacy code/ Old APIs)

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
