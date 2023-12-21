# Understanding Spring @Autowired

## Overview
Staring with Spring 2.5, the framework introduced annotation-driven Dependency Injection. The main annotation of this feature is `@Autowired`

## Enabling @Autowired annotations
The Spring Framework enables automatic dependency injection. In other words, by declaring all the bean dependencies in a Spring configuration file, Spring container can autowire relationships between collaborating beans. This is called Spring Bean Autowiring

To use Java-based configuration in our applicaton, let's enable annotation-driven injection to load our Spring configuration:

```java
@Configuration
@ComponentScan("com.baeldung.autowire.sample")
public class AppConfig  {}
```

Spring Boot introduces the @SpringBootApplication annotation. This single annotation is equivalent to using `@Configuration`, `@EnableAutoConfiguration` and `@ComponentScan`

Let's use this in the main class of the application

```java
@SpringBootApplication
public class App    {
    public static void main(String[] args)  {
        SpringApplication.run(App.class, args);
    }
}
```

As a result, when we run this Spring Boot application, **it will automatically scan the components in the current package and its sub-packages**. Thus, it will register them in Spring's Application Context and allow us to inject beans using `@Autowired`.

## Using @Autowired
After enabling annotation injection, **we can use autowiring on properties, settings and constructors**

### Autowired on Properties
Let's see how we can annotate a property using `@Autowired`. This eliminates the need for getters and setters.

First let's define a `fooFormatter` bean:

```java
@Component("fooFormatter")
public class FooFormatter   {
    public String format()  {
        return "foo";
    }
}
```

Then we will inject the bean into the `FooService` bean using `@Autowired` on the field definition:

```java
@Component
public class FooService {
    @Autowired
    private FooFormatter fooFormatter;
}
```

As a result, Spring injects `fooFormatter` when `FooService` is created.

### @Autowired on Setters
Now let's try adding `@Autowired` on a setter method.
In this next example, the setter method is called with the instance of `FooFormatter` when `FooService` is created:

```java
public class FooService {
    private FooFormatter fooFormatter;

    @Autowired
    public void setFormatter(FooFormatter fooFormatter) {
        this.fooFormatter = fooFormatter;
    }
}
```

### @Autowired on Constructors
Finally, let's use `@Autowired` on a constructor.
We'll see that an instance of `FooFormatter` is injected by Spring as an argument to the `FooService` constructor:

```java
public class FooService {
    private FooFormatter fooFormatter;

    @Autowired
    public FooService(FooFormatter fooFormatter)    {
        this.fooFormatter = fooFormatter;
    }
}
```

### @Autowired and Optional Dependencies
When a bean is being constructed, the `@Autowired` dependencies should be available. Otherwise, **if Spring cannot resolve a bean for wiring, it will throw an exception**

Consequently, it prevents the Spring container from launching successfully with an exception of the form:

```bash
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: 
No qualifying bean of type [com.autowire.sample.FooDAO] found for dependency: 
expected at least 1 bean which qualifies as autowire candidate for this dependency. 
Dependency annotations: 
{@org.springframework.beans.factory.annotation.Autowired(required=true)}
```

To fix this, we need to declare a bean of the required type:

```java
public class FooService {

    @Autowired(required = false)
    private FooDAO dataAccessor;
}
```

## Autowire Disambiguation
By default, Spring resolves `@Autowired` entries by type. **If more than one bean of the same type is available in the container, the bean will throw a fatal exception**

To resolve the conflict, we need to tell Spring explicitly which bean we want to inject.

### Autowiring by @Qualifier
For instance, let's see how we can use the `@Qualifier` annotation to indicate the required bean.

First we'll define two beans of type `Formatter`

```java
@Component("fooFormatter")
public class FooFormatter implements Formatter  {
    public String format()  {
        return "foo";
    }
}
```

```java
@Component("barFormatter")
public class BarFormatter implements Formatter {
    public String format() {
        return "bar";
    }
}
```

Now let's try and inject a `Formatter` bean into the `FooService` class:

```java
public class FooService {

    @Autowired
    private Formatter formatter;
}
```

In our example, there are two concrete implenetations of `Formatter` available for the Spring container. As a result, **Spring will throw a NoUniqueBeanDefinitionExecption exception when constructing the FooService**

```bash 
Caused by: org.springframework.beans.factory.NoUniqueBeanDefinitionException: 
No qualifying bean of type [com.autowire.sample.Formatter] is defined: 
expected single matching bean but found 2: barFormatter,fooFormatter
```

**We can avoid this by narrowing the implementation using a @Qualifier annotation:**

```java
public class FooService{
    @Autowired
    @Qualifier("fooFormatter")
    private Formatter formatter;
}
```

When there are multiple beans of the same type, it's a good idea to **use `@Qualifier` to avoid ambiguity**.

Please note that the value of the `@Qualifier` annotation matches with the name declared in the `@Component` annotation of our `FooFormatter` implementation.

### Autowiring by Custom Qualifier
Spring also allows us to create our own custom @Qualifier annotation, to do so, we should provide the `@Qualifier` annotation with the definition:

```java
@Qualifier
@Target({
    ElementType.FIELD, ElementType.METHOD, ElementType.TYPE, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface FormatterType {
    String value();
}
```

Then we can use the `FormatterType` within various implementations to specify a custom value:

```java
@FormatterType("Foo")
@Component
public class FooFormatter implements Formatter {
    public String format() {
        return "foo";
    }
}
```

```java
@FormatterType("Bar")
@Component
public class BarFormatter implements Formatter {
    public String format() {
        return "bar";
    }
}
```

Finally, our custom `Qualifier` annotation is ready to use for autowiring:

```java
@Component
public class FooService {

    @Autowired
    @FormatterType("Foo")
    private Formatter formatter;
}
```

The value specified in the `@Target` meta-annotation restricts where to apply the qualifier, which in our example is fields, methods, types and parameters.

### Autowiring by Name
**Spring uses the bean's name as a default qualifier value**. It will inspect the container and look for a bean with the exact 





