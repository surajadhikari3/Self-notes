
![[Pasted image 20250501104414.png]]

Configuration is feed to the session factory which is thread safe which creates the session which is not thread safe that makes the communication with the database. (Knowing the architecture is important)

- **SessionFactory**: A factory for `Session` objects, created once per database.
- **Session**: Represents a connection with a database. Provides methods for CRUD operations.
- **Transaction**: Manages a unit of work.
- **Query and Criteria**: Used to retrieve data. (HQL, Criteria Api)

| **Topic**                     | **Interview Question**                                              | **Short Answer**                                                                                            | **Real-World Example**                                                                                                                             |
| ----------------------------- | ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ORM Basics**                | What is ORM and why is it used in Hibernate?                        | ORM maps Java classes to DB tables to avoid boilerplate JDBC code.                                          | Mapping a `User` class to a `users` table.                                                                                                         |
| **Hibernate Architecture**    | Can you explain Hibernate architecture?                             | Consists of Configuration, SessionFactory, Session, and Transaction.                                        | Factory setup: Config → SessionFactory → Session → Transaction.                                                                                    |
| **Session vs SessionFactory** | What is the difference between Session and SessionFactory?          | SessionFactory is thread-safe and creates Sessions; Session is not thread-safe and used for DB interaction. | Like a Restaurant (SessionFactory) serving different Customers (Sessions).                                                                         |
| **Hibernate Annotations**     | How do you map a class to a table using annotations?                | Use `@Entity`, `@Table`, `@Id`, `@Column`.                                                                  | `@Entity <br>class User { <br>@Id <br>Long id; <br>@Column <br>String name; }`                                                                     |
| **Spring Data JPA**           | How does Spring Data JPA simplify Hibernate?                        | It abstracts DAO logic using `JpaRepository`.                                                               | `UserRepository extends JpaRepository<User, Long>` gives you full CRUD.                                                                            |
| **Transaction Management**    | How does Spring handle transactions with Hibernate?                 | Uses `@Transactional` and AOP proxy for commit/rollback management.                                         | Like a bank transaction: rollback if save fails.                                                                                                   |
| **Lazy vs Eager Fetching**    | What's the difference between lazy and eager fetching in Hibernate? | Lazy loads data when needed; eager loads all related data upfront.                                          | Lazy: loads Orders when accessed. Eager: loads Orders immediately with Customer.                                                                   |
| **N+1 Problem**               | What is N+1 select problem in Hibernate?                            | 1 query for parent and N queries for children = performance issue.                                          | 1 Customer + 10 Orders = 11 queries instead of 1 join. (Solution : Use join fetch  from the hql or use the @EntityGraph from the spring data jpa ) |
| **Caching**                   | How does Hibernate cache data?                                      | Uses first-level (Session) and second-level (e.g. Ehcache) caching.                                         | Once data is fetched, it's reused in the session or globally via 2nd level cache.                                                                  |
| **HQL vs SQL**                | What is HQL and how is it different from SQL?                       | HQL is object-oriented (uses entity names), SQL is DB-specific.                                             | HQL: `from User`, SQL: `SELECT * FROM users`.                                                                                                      |
|                               |                                                                     |                                                                                                             |                                                                                                                                                    |
|                               |                                                                     |                                                                                                             |                                                                                                                                                    |


## **What Is the N+1 Problem?**

When you fetch a list of parent entities, and **for each parent**, Hibernate executes an **extra SQL query** to fetch its child entities, you end up with:

> ✅ 1 query to get all parents  
> ❌ N additional queries to get each parent’s children

🔁 **Total = N + 1 queries**

## How to Solve the N+1 Problem

### ✅ Use `JOIN FETCH` in HQL/JPQL


```
List<Department> depts = entityManager.createQuery(     
"SELECT d FROM Department d JOIN FETCH d.employees", Department.class)     
.getResultList();

```
✅ Only 1 SQL query → fetch departments and employees in one go.

---

### ✅ OR Use `@EntityGraph` (Spring Data JPA)



```
@EntityGraph(attributePaths = "employees") 
List<Department> findAll();```

This tells Spring Data to load `employees` with each `Department` in one go.

```

Hibernate Schema Mamagement --> ( ddl-auto configuration)

Different modes............

validate --> just validate the schema
update --> do not drop the table but add the missing table
create --> drops all the table and create the new every time suitable for testing 
create-drop --> Similar to. the create 
none --> If you do not want the hibernate to manage the schema.......(Default value)

## **What is Cascading in Hibernate?**

Imagine you delete a **Customer**, and you want all their **Orders** deleted too — this is where **cascade** helps. You define this rule once, and Hibernate applies it automatically to all relationships.

---

## 🧩 **Types of Cascade in Hibernate**

Hibernate defines several **cascade types**, available through `CascadeType` enum.

| **Cascade Type** | **What It Does**                                                         | **Use Case Example**                            |
| ---------------- | ------------------------------------------------------------------------ | ----------------------------------------------- |
| `PERSIST`        | Saves child entities when parent is saved                                | Saving a `Customer` also saves their `Address`  |
| `MERGE`          | Updates child entities when parent is merged                             | Updating a `Customer` also updates `Orders`     |
| `REMOVE`         | Deletes child entities when parent is deleted                            | Deleting a `User` also deletes their `Posts`    |
| `REFRESH`        | Reloads child entities when parent is refreshed                          | Refreshing `Invoice` also refreshes its `Items` |
| `DETACH`         | Detaches child entities when parent is detached from persistence context | Detaching `Cart` also detaches its `CartItems`  |
| `ALL`            | Applies all of the above cascade operations                              | Typically used for tightly coupled entities     |

--> Detach is removing from the persistence context(From session ) but not the db while remove deletes from the db itself...

merge --> when the entity is detached from the persistence context and now want to persist the change to the db then the cascade type merge is used...

Joins 

The concept is the who have the foreign_key that is the owning side...
Who have the @JoinColumn in case of one-to-one and many-to-one relationship they are the owning side which actually holds the foreign key
And who have the the mappedBy they are the non-owning side

Note --> It is the best practise always to make the owning side to the many side in case of many-to-one relation. It helps to maintain the relation more deterministic....


### **One-to-One**

| Description | One entity is linked to only one of another.              |
| ----------- | --------------------------------------------------------- |
| Example     | A `User` has one `Profile`.                               |
| Annotation  | `@OneToOne`                                               |
| Owning Side | Usually the side that has the foreign key (`@JoinColumn`) |



@Entity 
public class User {   

@OneToOne(cascade = CascadeType.ALL)     
@JoinColumn(name = "profile_id") // Foreign key column     
private Profile profile; 
}

🧠 _Think of owning side as the one controlling the relationship table._


### **One-to-Many / Many-to-One**

| Description | One parent has many children; child belongs to one parent.              |
| ----------- | ----------------------------------------------------------------------- |
| Example     | One `Department` has many `Employees`.                                  |
| Annotation  | `@OneToMany` on parent, `@ManyToOne` on child                           |
| Owning Side | The `ManyToOne` side (the child), because it has the actual foreign key |
🧠 _Think of owning side as the one controlling the relationship table._

### **Many-to-Many**

|Description|Both sides can have multiple of the other.|
|---|---|
|Example|A `Student` can enroll in many `Courses`, and a `Course` can have many `Students`.|
|Annotation|`@ManyToMany`|
|Owning Side|The side with `@JoinTable` (defines the join table explicitly)|

```
@Entity 
public class Student { 

@ManyToMany 
@JoinTable(   
name = "student_course",     //name of the join table 
joinColumns = @JoinColumn(name = "student_id"),     //FK to Student
inverseJoinColumns = @JoinColumn(name = "course_id")) //FK to course
private List<Course> courses; 
}
```
## ✅ **Summary Table of Relationship Types**

| Relationship Type | Annotations Used             | Owning Side                  | Common Mapping            |
| ----------------- | ---------------------------- | ---------------------------- | ------------------------- |
| One-to-One        | `@OneToOne`, `@JoinColumn`   | The side with `@JoinColumn`  | `User` → `Profile`        |
| One-to-Many       | `@OneToMany(mappedBy="...")` | The Many side (`@ManyToOne`) | `Department` → `Employee` |
| Many-to-One       | `@ManyToOne`, `@JoinColumn`  | **This is the owning side**  | `Employee` → `Department` |
| Many-to-Many      | `@ManyToMany`, `@JoinTable`  | The side with `@JoinTable`   | `Student` ↔ `Course`      |

Special case Many to many with the join table having it's extra  attributes in the relation itself.... (Check the sir's hibernate note for the many to many read 3.1 and 4.1..)

In this case we have to make the composite key with @Embedable annotation and that annotation we use in the class which maintains the relationship.. 

1. First make the composite key
```
@Embeddable
class CourseRatingKey implements Serializable {

    @Column(name = "student_id")
    Long studentId;

    @Column(name = "course_id")
    Long courseId;

    // standard constructors, getters, and setters
    // hashcode and equals implementation
}

```

2. Use this composite key to the class as the @EmbeddedId that is creating the join table. Here this table has the Many To One relation with both of them.

```

@Entity
class CourseRating {

    @EmbeddedId
    CourseRatingKey id;
    

    @ManyToOne
    @MapsId("studentId")
    @JoinColumn(name = "student_id")
    Student student;

    @ManyToOne
    @MapsId("courseId")
    @JoinColumn(name = "course_id")
    Course course;

    int rating;

}

```


This code is very similar to a regular entity implementation. However, we have some key differences:

- We used ***@EmbeddedId* to mark the primary key**, which is an instance of the _CourseRatingKey_ class.
- **We marked the *student* and *course* fields with *@MapsId*.**

_@MapsId_ means that we tie those fields to a part of the key, and they’re the foreign keys of a many-to-one relationship.

After this, we can configure the inverse references in the _Student_ and _Course_ entities as before:

```
class Student {

    // ...

    @OneToMany(mappedBy = "student")
    Set<CourseRating> ratings;

    // ...
}

class Course {
    // ...

    @OneToMany(mappedBy = "course")
    Set<CourseRating> ratings;

    // ...
}
```


the other way if we do not have the composite key but have extra grade field then we can define like below :

```
@Entity
public class Enrollment {
    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne
    @JoinColumn(name = "course_id")
    private Course course;

    private String grade;

    // getters and setters
}
```


## 🧠 Summary

|Relationship Style|Extra Field Support|Best For|
|---|---|---|
|`@ManyToMany` only|❌ No|Simple links (tags, followers)|
|Join Entity (e.g., `Enrollment`)|✅ Yes|Relationships with extra data like grades, timestamps, roles|
### **Caching**

- **First-Level Cache** is enabled by default and works within a session.
- **Second-Level Cache** is optional and shared across sessions(which is session factory level).
- **Enable Second-Level Cache**:
    
    ```java
    @Entity
    @Cacheable
    @org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    public class Student {
        // Fields, constructors, getters, and setters
    }
    ```
## 📌 Summary

|Feature|Status|Notes|
|---|---|---|
|First-level cache|✅ Default|Session-local|
|Second-level cache|⚙️ Needs setup|SessionFactory-scoped|
|Query cache|⚙️ Optional|Cache result sets|
|Cache provider|❗Required|Ehcache, Redis, etc.|
|Annotations needed|✅ `@Cacheable`|Use with `@Cache` and strategy|


- **Batch Fetching**:
    
    ```java
    @BatchSize(size = 10)
    private List<Course> courses;
    ```

