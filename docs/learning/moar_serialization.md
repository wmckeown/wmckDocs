# Different Serialization Approaches in Java

*Adapted from [https://www.baeldung.com/java-serialization-approaches](https://www.baeldung.com/java-serialization-approaches)*

## Overview
Serialization is the process of converting an object into a stream of bytes. That object can then be saved to a database or transferred over a network. The opposite operation, extracting an object from a series of bytes, is deserialization. Their main purpose is to save the state of an object so that we can recreate it when needed.

This page covers different serialization approaches for Java objects:

* Java Native Serialization APIs
* Libraries that support JSON and YAML Formats for Serialization
* Cross-language protocols

## Sample Entity Class
Let's start by introducing a simple entity that we're going to use throughout this tutorial.

```java
@Getter
@Setter
public class User   {
    private int id;
    private String name;
}

```

In the next sections, we'll go through the most widely used serialization protocols. Through examples, we'll learn the basic usage of each of them:

## Java Native Serialization

Serialization in Java helps to achieve effective and prompt communication between multiple systems. Java specifies a default way to serialize objects. A java class can override this default serialization and define its own way of serializing objects.

The advantages of Java native serialization are:

* It's a simple, yet extensible mechanism
* It maintains the object type and safety properties in the serialized form
* Extensible to support marshaling and un-marshaling as needed for remote objects
* This is a native Java solution, so it doesn't require any external libraries

### The Default Mechanism
As per the Java Object Serialization Specification, we can use the `writeObject()` from the `ObjectOutputStream` class to serialize the object. We can then use `readObject()` which belongs to the ObjectInputStream class, to deserialize it.

We'll illustrate the basic process with the `User` class we created earlier.

First, our class needs to implement the Serializable interface.

```java
public class User implements Serializable   {
    // fields and methods
}
```

Next we need to add the `serialVersionUID` attribute:

```java
private static final long serialVersionUID = 1L'
```

Now let's create a `User` object:

```java
User user = new User();
user.setId(1);
user.setName("Mark");

```

We need to provide a file path to save our data:

```java
String filePath = "src/test/resources/protocols/user.txt";
```

Now it's time to serialize our `User` object to a file:

```java
FileOutputStream fileOutputStream = new FileOutputStream(filePath);
ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
objectInputStream.writeObject(user);

```

Here, we used ObjectOutputStream for saving the state of the `User` object to a `user.txt` file.

We can then read the `User` object from the same file and deserialize it.

```java
FileInputStream fileInputStream fileInputStream = new FileInputStream(filePath);
ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);
User deserializedUser = (User) objectInputStream.readObject();
```

Finally, we can test the state of the loaded object:

```java
assertEquals(1, deserializedUser.getId());
assertEquals("Mark", deserializedUser.getName())

```

This is the default way to serialize Java objects. In the next section, we'll see the custom way to do the same.

### Custom Serialization using the Externalizable Interface
Custom serialization can be particularly useful when trying to serialize an object that has some unserializable attributes. This can be done by implementing the Externalizable interface, which has two methods:

```java
public void writeExternal(ObjectOutput out) throws IOException;

public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException;

```

We can implement these two methods inside the class we want to serialize.

### Java Serialization Caveats
There are some caveats with native serialization in Java:

* **Only objects marked `Serializable` can be persisted**. The `Object` class does not implement serializable, and hence, not all the objects in Java can be persisted automatically
* When a class implements the `Serializable` interface, all of its subclasses are serializable as well. **However, when an object has reference to another object, these objects must implement the serializable interface separately, or else a `NotSerializableException` will be thrown**.
* **If we want to control the versioning, we need to provide a `serialVersionUID` attribute.** This attribute is used to verify that the saved and loaded objects are compatible. Therefore, we need to ensure that it is always the same, or else `InvalidClassException` will be thrown.
* Java serialization heavily uses I/O streams. We need to close a stream immediately after a read or write operation because, **if we forget to close the stream, we will end up with a resource leak.** To prevent such resource leaks, we can use the try-with-resources idiom.

## Gson Library

