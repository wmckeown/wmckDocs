# `@ContextConfiguration`

`@ContextConfiguration` dines class0level metadata that is used to determine how to laod and configure an `ApplicationContext` for integration tests. Specifically, `@ContextConfiguration` declatres the application context resource `locations` or the component `classes` used to load the context.

Resource locations are typically XML configuration files or Groovy scripts located in the classpath, while component classes are typically `@Configuration` classes. However, resource locations csn also refer to files and scripts in the file system, and component classes can be `@Component` classes, `@Service` classes, and so on. 

The following example shows a `@ContextConfiguration` annotation tat refers to an XML file:

```java
@ContextConfiguration("/test-config.xml")
class XmlApplicationContextTests    {
    // class body...
}
```

The next example, shows a `@ContextConfiguration` annotation that refers to a class:

```java
@ContextConfiguration(classes = TestConfig.class)
class ConfigClassApplicationContextTests    {
    // class body...
}
```

As an alternative, or in addition to declaring resource locations or component classes, you can use `@ContextConfiguration` to declare `ApplicationContextInitializer` classes. The following example shows such a case:

```java
@ContextConfiguration(initializers = CustomContextInitializer.class)
class ContextInitializerTests    {
    // class body...
}
```

You can optionally use `@ContextConfiguration` to declare the `ContextLoader` strategy as well. Note: However, that you typically do not need to explicitly configure the loader, since the default loader supports `initializers` and either resource `locations` or component `classes`.

The following example uses both a location and a loader:

```java
@ContextConfiguration(locations = "/test-context.xml", loader = CustomContextLoader.class)
class CustomLoaderXmlApplicationContextTests    {
    // class body...
}
```

!!! Note
        `@ContextConfiguration` provides support for inheriting resource locations or configuration classes as well as context initializers that are declared by superclasses or enclosing classes.