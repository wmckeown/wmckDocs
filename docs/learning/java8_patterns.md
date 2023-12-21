# Java 8 Features

*Adapted From [https://winterbe.com/posts/2014/03/16/java-8-tutorial/](https://winterbe.com/posts/2014/03/16/java-8-tutorial/)*

This tutorial will guide you through all the new Java 8 language features. Backed by short and sweet code samples, you'l learn about:

* default interface methods
* lambda expressions
* method references
* repeatable annotations

At the end of the article, you'll be more familiar with the most recent API changes like:

* streams
* functional interfaces
* map extensions
* Date API

## Default Methods for Interfaces
Java 8 enables us to add non-abstract method implementations to interfaces by using the `default` keyword. This feature is also known as Extensiom Methods. Here is our first example:

```java
interface Formula   {

    double calculate(int a);

    default double sqrt(int a)  {
        return Math.sqrt(a);
    }
}

```

Besdies the abstract method `calculate`, the interface `Formula` also defines the default method `sqrt`. Concrete classes only have to implemegt the abstract method `calculate`. The default method `sqrt` can be used out of the box.

```java
Formula formula = new Formula() {
    @Override
    public double calculate(int a)  {
        return sqrt(a * 100);
    }
};

formula.calculate(100); // 100.0
formula.sqrt(16);       // 4.0
```

The formula is implemented as an anonymous object. The code is quite verbose: 6 lines of code for sycg a simple calculation of `sqrt(a * 100)`. As we'll see in the next section, there's a much nicer way of implementing single-method objects in Java 8.

## Lambda Expressions
Let's start with a simple example of how to sort a list of strings in prior versions of Java:

```java
List<String> names = Arrays.asList("peter", "anna", "mike", "xenia");

Collections.sort(names, new Comparator<String>()    {
    @Override
    public int compare(String a, String b)  {
        return b.compareTo(a);
    }
});

```

The static utility method `Collections.sort` accepts a list and a comparator in order to sort the elements of the given list. You often find yourself creating anonymous comparators and passing them to the sort method.

Instead of creating anonymous objects all day long, Java 8 comes with a much shorter syntax, **lambda expressions**:

```java
Collections.sort(names, (String a, String b) -> {
    return b.compareTo(a);
});
```

As you can see, the code is much shorter and easier to read. But it gets even shorter:

```java
Collections.sort(names, (String a, String b) -> b.compareTo(a));

```

For one line method bodies, you can skip both the braces `{}` and the `return` keyword. And it gets even shorter.

```java
Collections.sort(names, (a,b) -> b.compareTo(a));
```

The java compiler is aware of the parameter types so you can skip them as well. Let'a dive deeper into how lambda expressions can be used in the wild.

## Functional Interfaces
How do lambda expressions fit into Java's type system? Each lambda corresponds to a given type, specified by an interface. A so-called *functional interface* must contain exactly one abstract method declaration. Each lambda expression of that type will be matched to this abstract method. Since default methods are no abstract, you are free to add default methods to your functional interface.

We can use arbitrary interfaces as lambda expressions as long as the interface only contains one abstract method. To ensure that your interface meets the requirement, you should add the `@FunctionalInterface` annotation. The compiler is aware of this annotation and throws a compiler error as soon as you try to add a second abstract method declaration to the interface.

Example:

```java
@FunctionalInterface
interface Converter<F, T>  {
    T convert(F from)'
}

//...

Converter<String, Integer> converter = (from) -> Integer.valueOf(from);
Integer converted = converter.convert("123");
System.out.println(converted); // 123

```

Keep in mind that the code is also valid if the `@FunctionalInterface` annotation would be ommitted.

## Method and Constrcutor References
The above sample code can be further simplified by utilising static method references:

```java
Converter<String, Integer> converter = Integer::valueOf;
Integer converted = converter.convert("123");
System.out.println(converted); // 123

```

Java 8 enables you to pass references of methods or constructors via the `::` keyword. The above example shows how to reference a static method. But we can also reference object methods:

```java
class Something {
    String startsWith(String s) {
        return String.valueOf(s.charAt(0));
    }
}

//...

Something something = new Something();

Converter<String, String> converter = something::startsWith;

String converted = converter.convert("Java");

System.out.println(converted); // "J"

```

Let's see how the `::` keyword works for constructors. First we define an example bean with different constructors.

```java
class Person  {
    String firstname, lastname

    Person() {}

    Person(String firstName, String lastName)   {
        this.firstName = firstName;
        this.lastName = lastName;
    }
}

```

Next we specify a person factory interface to be used for creating new instances of `Person`

```java
interface PersonFactory<P extends Person>   {
    P create(String firstName, String lastName);
}

```

Instead of implementing the factory manually, we glue everything together via constructor references:

```java
PersonFactory<Person> personFactory = Person::new;
Person person = personFactory.create("Peter", "Parker");

```

We create a reference to the `Person` constuctor via `Person::new`. The Java compiler automatically chooses the right constructor by matching the signature of `PersonFactory.create`.

## Lambda Scopes
Accessing outer scope variables from lambda expressions is very similar to anonymous objects. You can access final variables from the 



