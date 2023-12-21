# Java Serilaization
*Adapted from: [https://www.baeldung.com/java-serialization](https://www.baeldung.com/java-serialization)*

## Introduction
Seriaization is the conversion of the state of an object into a byte stream; deserialization does the opposite. Stated differently, serialization is the conversion of Java object into a static stream (sequence) of bytes, which we can save to a database or transfer over a network.

## Serialization and Deserialization
The serialization process is instance-dependent; for example, we can serialize objects on one platform and deserialize them on another. Classes that are elligible for serialization need to implement the special marker interface, Serializable.

Both ObjectInputStream and ObjectOutputStream are high level classes that extend Java.io.InputStream and java.io.OutputStream respectively. ObjectOutputStream can write primitve types and graphs of objects to an OutputStream as a stream of bytes. We can read these streams using ObjectInputStream.

The most important method in ObjectInputStream is:

```java
public final void writeObject(Object o) throws IOException;
```

This method takes a serializable object and converts it into a sequence (stream) of bytes.

Similarly, the most important method in ObjectOutputStream is:

```java
public final Object readObject() throws IOException, ClassNotFoundException;
```

This method can read a stream of bytes and convert it back into a Java object. It can then be cast back to the original object.

### Example
Let's ilustrate serialization with a Person class. Note that static fields belong to a class (as opposed to an object) and are not serialized. Also, note that we can use the keyword `transient` to ignore class fields during serialization.

```java
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    static String country = "ITALY";
    private int age;
    private String name;
    transient int height;

    // Boilerplate getters and setters
}

```

The test belwo shows the value of saving an object of type Person to a local file and then reading the value back in:

```java
@Test
public void whenSerializingAndDeserializing_ThenObjectIsTheSame() throws IOException, ClassNotFoundException    {
    Person person = new Person();
    person.setAge(20);
    person.setName("Joe");

    FileOutputStream fileOutputStream = new FileOutputStream("yourfile.txt");
    ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
    objectOutputStream.writeObject(person);
    objectOutputStream.flush();
    objectOutputStream.close();

    FileInputStream fileInputStream = new FileInputStream("yourfile.txt");
    ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);
    Person p2 = (Person) objectInputStream.readObject();
    objectInputStream.close();

    assertTrue(p2.getAge() == person.getAge());
    assertTrue(p2.getName() == person.getName());

}
```

We used ObjectOutputStream for saving the state of this object to a file using `FileOutputStream`. The file "yourfile.txt" is created in the project directory. This file is then loaded using `FileInputStream`. `ObjectInputStream` picks this stream up and converts it into a new object called p2.

Finally, we'll test the state of the loaded object to ensure it matches the state of the original object.

*Note that we have to explicitly cast the state of the loadefd object to a `Person` type*

## Java Serialization Caveats

### Inheritance and Composition
When a class implements the java.io.Serializable interface, all of its sub-classes are serializable as well. Conversely, when an object has a reference to another object, these objects must implement the `Serializable` interface seperately, or else a `NotSerializable` exception will be thrown.

### Serial Version UID
The JVM associates a version (long) number with each serializable class. We use it to verify that the saved and loaded objects have the same attributes, and thus are compatible on serialization.

Most IDEs can generate this number automatically, and it's based on the class name, attributes and associated access modifiers. Any changes result in a different number and can cause an InvalidClassException.

If a serializable class doesn't declare a serialVersionUID, the JVM will generate one automatically at run-time. However, it's highly recommended that each class declares its serialVersionUID,as the generated one is compiler-dependent and thus may result in unexpected InvalidClassExceptions.

### Custom Serialization in Java
Java specifies a default way to serialize objects, but Java classes can override this default behaviour. Custom serialization can be particularly useful when trying to serialize an object that has some unserializable attributes. We can do this by providing two methods inside the class we want to serialize.

```java
private void writeObject(ObjectOutputStream out) throws IOException;
```

and

```java
private void readObject(ObjectInoutStream in) throws IOException, ClassNotFoundException;
```

With these methods, we can serialize the unserializable attributes into other forms that we can serialize:

```java

@Getter
@Setter
public class Employee implements Serializable   {
    private static final long serialVersionUID = 1L;
    private transient Address address;
    private Person person;

private void writeObject(ObjectOutputStream out) throws IOException {
    out.defaultWriteObject;
    out.writeObject(address.getHouseNumber);
}

private void readObject(ObjectInputStream ois) throws ClassNotFoundException, IOexception {
    ois.defaultReadObject();
    Integer houseNumber = (Integer) ois.readObject();
    Address a = new Address();
    a.setHouseNumber(houseNumber);
    this.setAddress(a);
}
}
```
 
```java
@Getter
@Setter
public class Address    {
    private int houseNumber;
}
```

We can run the following unit test to test this custom serialzation.

```java
@Test
public void whenCustomSerializingAndDeserializing_ThenObjectIsTheSame() throws IOException, ClassNotFoundException  {
    Person p = new Person();
    p.setAge(20);
    p.setName("Joe");

    Address a = new Address();
    a.setHouseNumber(1);

    Employee e = new Employee();
    e.setPerson(p);
    e.setAddress(a);

    FileOutputStream fileOutputStream = new FileOutputStream("yourfile2.txt");

    ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
    objectOutputStream .writeObject(e);
    objectOutputStream.flush();
    objectOutputStream.close();

    FileInputStream fileIputStream = new FileInputStream("yourfile2.txt");
    ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);
    
    Employee e2 = (Employee) objectInputStream.readObject();
    objectInputStream.close();

    assertTrue(e2.getPerson.getAge() == e.getPerson().getAge());
    assertTrue(e2.getAddress.getHouseNumber() == e.getAddress().getHouseNumber());
}

```

In this code we can see how to save some unserializable attributes by serializing `Address` with custom serialization. Note that we must mark the unserializable attributes as transient to avoid the `NotSerializableException`
