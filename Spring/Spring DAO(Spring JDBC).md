
The **Spring DAO** (Data Access Object) module is a key part of the Spring Framework that simplifies interaction with databases and other persistence layers.

**What are the advantages of using Spring JDBC over plain JDBC?**  

it provide the jdbc template'

Spring JDBC reduces boilerplate code, provides better exception handling, simplifies connection management, and integrates easily with Spring’s transaction management._

```
@Repository
public class EmployeeDao {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;

    public int save(Employee e) {
        String sql = "INSERT INTO employee(name, salary) VALUES(?, ?)";
        return jdbcTemplate.update(sql, e.getName(), e.getSalary());
    }

    public List<Employee> findAll() {
        return jdbcTemplate.query("SELECT * FROM employee", 
            new BeanPropertyRowMapper<>(Employee.class));
    }
}

```


### **What is the role of RowMapper?**

**Answer:** `RowMapper<T>` is an interface used to map rows of a ResultSet to Java objects.

```
public class EmployeeRowMapper implements RowMapper<Employee> {
    public Employee mapRow(ResultSet rs, int rowNum) throws SQLException {
        Employee e = new Employee();
        e.setId(rs.getInt("id"));
        e.setName(rs.getString("name"));
        e.setSalary(rs.getDouble("salary"));
        return e;
    }
}

```

## Summary (Cheat Sheet Style)

| Area        | Key Concept                             | Tool/Annotation                            |
| ----------- | --------------------------------------- | ------------------------------------------ |
| Spring JDBC | Template-based SQL execution            | `JdbcTemplate`                             |
| Spring ORM  | ORM with mapping and session management | `@Entity`, `@Repository`, `@Transactional` |
| JPA         | Standardized ORM API                    | `JpaRepository`, `@Query`                  |
| Transaction | Declarative transaction management      | `@Transactional`                           |
| Lazy Load   | Delayed data fetching until accessed    | `fetch = FetchType.LAZY`                   |
| Row Mapping | Convert ResultSet to Object             | `RowMapper`                                |