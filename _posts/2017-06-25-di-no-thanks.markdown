---
layout: post
title:  "Dependency injection framework? - no thanks!"
date:   2017-06-24 10:18:00
categories: dependency injection
---

Dependency injection is old and popular technique to glue your application components together and keep them decoupled at the same time.
This is that simple, so why we need sophisticated DI frameworks which sometimes are not trivial to use?
Lets review pros and cons of few popular frameworks from java world and decide whether we need them and can we do better without.

First one will be pioneer among DI framework - spring xml. Probably most hated framework because of its verbosity.

### Spring Xml DI

```xml
<xml>
    <bean id="a_id" class="A"/>
    <bean class="B">
        <property name="a" ref="a_id"/>
    </bean>>
</xml>
```
-   Pros
    -   no magic, injection based on instances rather than classes
    -   does not pollute classes with annotations like
        @Inject,@Name etc.
-   Cons
    - verbose, xml is heavy
    - non homogeneous - injection logic is in XML, application is in java.
    - you desperately need IDE to refactor and manipulate xml contexts, but it is still too easy to embarrass IDE by quite trivial case.
    - any non trivial initialization become bulky and cumbersome expressed in XML (a.k.a. XML programming)
    - xml contexts tend to grow in size as they hard to split and combine.

```java
@Component
class A{
}
@Component
class B{
    @Autowired
    A a;
}
```

### Spring Annotation based DI
-   Pros
    -   homogeneity , i.e. all DI logic expressed in java
-   Cons
    -   a lot of magic happens during runtime causing non trivial pazzles for trivial applications. Weak control over application initialization.
    -   DI based on classes rather than instances. Robot legs problem, you need to use qualifiers to distinct instances.
    -   Pollute classes with annotations related to DI

### Google Guice
-   Pros
    -   homogeneity, i.e. DI logic expressed in java.
    -   keeps DI logic in one place - guice module, does not use classpath resolution.
-   Cons
    -   DI based on classes rather than instances , same robot legs problem as for spring AOP D, need qualifiers to distinct instances.
    -   Pollute classes with annotations related to DI

### Alternative approach

Lets ask ourselves can we do better? I think yes.

There are few problems to be addressed to beat above frameworks:

1. gluing logic reside in plain java code not in any xml/json/whatever ( homogeneity  )
2. no runtime magic, proxy generation etc.

Lets demonstrate that approach on simple example:

we have class Server that process string:

```java
public interface Server{
    String process(String str);
}
```

but it is lazy bastard and prefer to delegate it to function provided in constructor:

```java
public class ServerImpl implements Server{
    public ServerImpl(Function<String,String> processor){
        this.processor = processor;
    }
    public String process(String inStr) {
        return processor.apply(inStr);
    }
}
```

inject !

```java
public class DiModule {
    public Function<String, String> getProcessor() {
        return (s) -> "processed(" + s + ")";
    }

    public Server getServer(){
        return new ServerImpl(getProcessor());
    }
}
```
Run it!

```java
class Main{
    public static void main(String[] args){
        System.out.println(new DiModule().getServer().process("in_str"));
        //will print : "processed(in_str)"
    }
}
```

mock with overriding and test!

```java
class TestDi{
    public void testDi(){
        DiModule testModule = new DiModule() {
            @Override
            public Function<String, String> getProcessor() {
                return (s)->"processed_by_mock(" + s + ")";
            }
        };
        Assert.assertEquals("processed_by_mock(in_str)", mockedModule.getServer().process("in_str"));
    }
}
```
We miss singletons here for dealing with heavy classes and services, it is very easy to fix:

```java
public class DiModule {

    private SingletonsContainer singletons = new SingletonsContainer();

    public Function<String, String> getProcessor() {
        return (s) -> s + "processed";
    }

    public Server getServer(){
        return singletons.get("server", ()->{
            return new ServerImpl(getProcessor());
        });
    }
}
```

SingletonsContainer is pretty easy to implement so I will put only interface here:

```java
public interface SingletonsContainer extends AutoCloseable{
    <T> T get(String name, Supplier<T> provider);
    void close() throws Exception {}
}
```

I use it at work for more than 2 years and feel relief comparing to times when I had to deal with spring and even with guice.
