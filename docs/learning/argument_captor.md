# Mockito ArgumentCaptor

## Using ArgumentCaptor
ArgumentCaptor allows us to capture an argument passed to a method to inspect it. This is especially useful when we can't access the argument outside of the method that we would lke to test.

For example, consider an `EmailService` class with a send method that we would like to test:

```java
public class EmailService   {

    private DeliveryPlatform platform;

    public EmailService(DeliveryPlatform platform)  {
        this.platform = platform
    }

    public void send(String to, String subject, String body, boolean html)  {
        Format format = Format.TEXT_ONLY;
        if (html)   {
            format = format.HTML;
        }
        Email email = new Email(to, subject, body);
        email.setFormat(format);
        platform.deliver(email);
    }

    ...
}
```

In `EmailService.send()`, notice how `platform.deliver()` takes a new Email as an argument. As part of out test, we'd like to check that the format field of the new `Email` is set to `Format.HTML`. To do this, we need to capture and inspect the argument that is passed to `platform.deliver()`.

Let's see how ArgumentCaptor helps out with this:

### Setting up the unit test
First, let's create out unit test class

```java
@ExtendWith(MockitoExtension.class)
class EmailServiceUnitTest  {

    @Mock
    DeliveryPlatform platform;

    @InjectMocks
    EmailService emailService;

    ...
}

```

We're using the `@Mock` annotation to mock `DeliveryPlatform`, which is automatically injected into our `EmailService` with the `@InjectMocks` annotation.

### Add an ArgumentCaptor Field
Second, let's add a new ArgumentCaptor field of type Email to store our captured argument

```java
@Captor
ArgumentCaptor<Email> emailCaptor;
```

### Capture the Argument
Third, let's use `verify()` with the `ArgumentCaptor` to capture the `Email`:

```java
verify(platform).deliver(emailCaptor.capture());
```

We can then get the captured value and store it as a new Email object:

```java
Email emailCaptorValue = emailCaptor.getValue();
```

### Inspect the Captured Value
Finally, let's see the whole test with an assert to inspect the captured Email object:

```java
@Test
void whenDoesSupportHtml_expectHTMLEmailFormat()    {
    String to = "info@baledung.com";
    String subject = "Using ArgumentCaptor";
    String body = "Hey, Let's use ArgumentCaptor";

    emailService.send(to, subject, body, to);

    verify(platform).deliver(emailCaptor.capture());
    Email value  = emailCaptor.getValue();
    assertThat(value.getFormat()).isEqualTo(Format.HTML);
} 
```

## Avoiding Stubbing
Although we can use an ArgumentCaptor with stubbing we should generally avoid doing so. To clarify, in Mockito, this generally means avoiding using an ArgumentCaptor with Mockito.when. With stubbing, we should use an ArgumentMatcher instead.

Let's look at a few reasons as to why we should avoid stubbing.

### Decreased Test Readability
First consider a simple test:

```java
Credentials credentials = new Credentials("baeldung", "correct_password", "correct_key");
when(platform.authenticate(eq(credentials))).thenReturn(AuthenticationStatus.AUTHENTICATED);

assertTrue(emailService.authenticatedSuccessfully(credentials));
```

Here we use `eq(credentials)` to specify when the mock should return an object.

Next, consider the same test using an `ArgumentCaptor` instead:

```java
Credentials credentials = new Credentials("baeldung", "correct_password", "correct_key");
when(platform.authenticate(credentialsCaptor.capture())).thenReturn(AuthenticationStatus.AUTHENTICATED);

assertTrue(emailService.authenticatedSuccessfully(credentials));
assertEquals(credentials, credentialsCaptor.getValue());
```

In contrast to the first test, notice how we have to perform an extra assert on the last line to do the same as `eq(credentials)`

Finally, notice how it isn't immediately clear what credentialsCaptor.capture() refers to. **This is because we have to create the captor outside of the line we use it on, reducing readability.**

### Reduced Defect Localisation
Another reason is that if emailService.authenticatedSuccessfully doesn't call platform.authenticate, we'll get an exception:

```bash
org.mockito.exceptions.base.MockitoException: 
No argument value was captured!
```

This is because our stubbed method hasn't captured an argument. However, the real issue is not in our test itself but in the actual method we are testing.

In other words, **it misdirects us to an exception in the test, whereas the actual defect is in the method we are testing.**

## Conclusion
In this short article, we looked at a general use case of using `ArgumentCaptor`. We also looked at the reasons for avoiding using `ArgumentCaptor` with stubbing.