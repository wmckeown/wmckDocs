# Summary of Annotations

## Spring
### @Autowired
See seperate article on [@Autowired]()


## Lombok
### @Data
A shortcut annotation that combines the features of `@ToString`, `@EqualsAndHashCode`, `@Getter`, `@Setter` and `@RequiredArgsConstructor`. So @Data generates all boilerplate code associated with POJOs (Plain Old Java Objects) and beans. i.e. getters for all fields, setters for all non-final fields, and appropriate `toString`, `equals` and `hashcode` implementations that involve the fields of the class. It also includes a constructor that initialises all final fields as well as non-final fields with no initialiser that have been marked with `@NonNull` to ensure that the field is never null

## Junit5
### @TestInstance(Lifecycle)
Annotation that alows you to configure the lifecycle of JUnit5 tests. There are two modes:

| Mode                                | Description                                                                                   |
| ----------------------------------- | --------------------------------------------------------------------------------------------- |
| @TestInstance(Lifecycle.PER_METHOD) | Default mode. Creates an instanve of the test class for each test method annotated with @Test |
| @TestInstance(Lifecycle.PER_CLASS)  | Ask JUnit to create only one instance of the test class and re-use it between tests           |

If none of the variables or functions are static, We can use instance methods in the @BeforeAll when we use the PER_CLASS lifecycle. We should also note that any changes made to the state of the instance variables by one tests will now be visible to the others.

### @Testcontainers
JUnit5 integration with TestContainers is provided using the `@Testcontainers` annotation.

The extension finds all fields that are annotated with `@Container` and calls their container lifecycle methods (methods on the `Startable` interface). Containers declared as static fields will be shared between test methods. They will be started only once before any test method is executed and stopped after the last test method has executed. Containers declared as instance fields will be started and stopped for each test method.

!!! Note

    Only sequential test execution is officially supported. Using it with parallel executed tests may yield unintended side-effects

E.g.

```java
@TestConatiners
class MixedLifecycleTests   {
    // Will share container between methods
    @Container
    private static final MySQLContainer MY_SQL_CONTAINER = new MySQLContainer();

    // Will be started and stopped for each test method
    @Container
    private PostgreSQLContainer postgresqlContainer = new PostgreSQLContainer()
        .withDatabaseName("foo")
        .withUsername("foo")
        .withPassword("supersecret")

    @Test
    void test() {
        assertThat(MY_SQL_CONTAINER.isRunning()).isTrue();
        assertThat(postgreSQLContainer.isRunning()).isTrue();
    }
}






