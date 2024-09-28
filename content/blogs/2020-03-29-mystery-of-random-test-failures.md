---
categories:
- junit
date: "2020-03-29T00:00:00Z"
description: ""
tags: [junit, test]
title: Mystery of Random Test Failures
---
On `mvn test` the tests were failing randomly. There was one thing common among all the test that failed,
they were all interacting with a mocked spring bean. The exception stacktrace (shown below) was not very helpful. On re-running the failed test in IDE it always passed.  

```java
org.mockito.exceptions.misusing.UnfinishedVerificationException: 
Missing method call for verify(mock) here:
-> at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)

Example of correct verification:
    verify(mock).doSomething()
```

If there is one issue that's hard to debug, it's an issue that can't be replicated. The intermittent failure meant it had to do with how maven forks the test. So, if I was able to run the tests without forking, I assumed the failure would be consistent. And, that's what exactly happened, when I reran the entire test suite in IJ with fork disabled, the test always failed consistently in one class.

There was nothing special about the test class, it was `@RunWith(SpringRunner.class)` that created a `@MockBean` instance of a a spring service.

I tried removing `@MockBean` on the instance and tried using regular mocks without `SpringRunner`, the test still failed. That's when I searched online and it lead me to [this](https://stackoverflow.com/a/22550055) stack overflow question.

Per the answer in StackOverflow, the error could have happened in the previous mock interaction with that Service but fails in subsequent interaction.

I checked the test that passed prior to the failed one, it looked like this.

```java
@SpringBootTest(classes = SomeTest.TestConfig.class)
public class SomeTest {

    @Autowired
    private ServiceUnderTest serviceUnderTest;

    @Test
    public void someTest() {

        // some service invocation
        // should invoke the underlying getFileById only once since it is cached
        verify(serviceUnderTest, times(1)).someMethod(any());
    }

    @Configuration
    public static class TestConfig {      
        @Bean
        ServiceUnderTest serviceUnderTest() {
            return mock(ServiceUnderTest.class);
        }
    }


}
```

Per the StackOverflow answer, I added the following after block to the test and as expected this test started failing with the same error.
```
    @After
    public void tearDown() {
        validateMockitoUsage();
    }
```

The fix was simple after identifying the test that caused the issue, instead of creating a bean in the `Config` class, I used the `@MockBean` to create a bean that was not CGLib enhanced.



```java
@SpringBootTest(classes = SomeTest.TestConfig.class)
public class SomeTest {

    @MockBean
    private ServiceUnderTest serviceUnderTest;

    @Test
    public void someTest() {

        // some service invocation
        // should invoke the underlying getFileById only once since it is cached
        verify(serviceUnderTest, times(1)).someMethod(any());
    }

    @Configuration
    public static class TestConfig {      
       // some other config
    }
}
```
