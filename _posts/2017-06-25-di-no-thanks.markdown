---
layout: post
title:  "Dependency injection framework? - no thanks!"
date:   2017-06-24 10:18:00
categories: dependency injection
---

Dependency injection is an old popular technique to glue your application components together.
Sometime struggling with DI framework I asked myself if it is possible to make this easier and 
eventually I found solution that satisfy me.

Lets review pros and cons of few popular frameworks in java world and think if we can do better without.

First one is a pioneer among DI frameworks : spring xml. This is probably most hated framework because of its verbosity inherited from xml.

### Spring Xml DI

-   Pros
    -   no magic, injection based on object instances rather than classes which eliminates [robot leg problems](https://github.com/google/guice/wiki/FrequentlyAskedQuestions)
    -   does not pollute classes with annotations like
        @Inject,@Name etc.
-   Cons
    - verbose, xml is heavy
    - non homogeneous , injection logic is in XML, application is in java. Any non trivial configuration become bulky and cumbersome in XML (a.k.a. XML programming)
    - you need IDE to refactor and manipulate xml contexts and, still, IDE easy get confused by quite trivial cases.
    - xml contexts tend to grow in size as they hard to split and combine.

### Spring Annotation based DI

-   Pros
    -   homogeneity - DI logic expressed in java
-   Cons
    -   a lot of magic happens during runtime which cause non trivial puzzles for trivial applications. Weak control over application initialization order.
    -   DI based on classes rather than instances so [Robot leg problems](https://github.com/google/guice/wiki/FrequentlyAskedQuestions), you need to use qualifiers to distinct instances.
    -   Pollute classes with annotations related to DI

### Google Guice
-   Pros
    -   homogeneity - DI logic expressed in java.
    -   keeps DI logic in one place - guice module. 
    -   Does not have magic problems with classpath resolution comparing to spring.
-   Cons
    -   DI based on classes rather than instances so [Robot leg problems](https://github.com/google/guice/wiki/FrequentlyAskedQuestions), you need to use qualifiers to distinct instances.
    -   Pollute classes with annotations related to DI

### Alternative approach

There are few problems we need to address to beat above frameworks:

1. Having gluing logic in plain java code not in any xml/json/whatever ( homogeneity  )
2. No runtime magic, proxy generation etc.

Lets demonstrate that approach on simple example:

Imagine that we have class Server that does something with string:

```
public interface Server{
    String process(String str);
}
```

it is lazy bastard and prefer to delegate processing to service provided in constructor:


{% highlight java %}
public class ServerImpl implements Server{
    public ServerImpl(Function<String,String> processor){
        this.processor = processor;
    }
    public String process(String inStr) {
        return processor.apply(inStr);
    }
}

{% endhighlight %}

```java
```

lets inject this service!

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
and run application:

```java
class Main{
    public static void main(String[] args){
        System.out.println(new DiModule().getServer().process("in_str"));
        //will print : "processed(in_str)"
    }
}
```

if we want to mock provided service we just override *getProcessor()* method in module:

<!--
```java
public void testDi(){
    DiModule testModule = new DiModule() {
        @Override
        public Function<String, String> getProcessor() {
            return (s)->"processed_by_mock(" + s + ")";
        }
    };
    Assert.assertEquals("processed_by_mock(in_str)",
        mockedModule.getServer().process("in_str"));
}
```
-->

As you may notice we are missing singletons here , that could be useful when dealing with heavy classes and services, but this is very easy to fix:
Introduce SingletonsContainer which will hold all singletons in one place:

```java
public class DiModule implements AutoCloseable{

    private SingletonsContainer singletons = new SingletonsContainer();

    public Function<String, String> getProcessor() {
        return (s) -> s + "processed";
    }

    public Server getServer(){
        return singletons.get("server", ()->{
            return new ServerImpl(getProcessor());
        });
    }

    public void close(){
        singletons.close();
    }
}
```

SingletonsContainer is pretty easy to implement so I will just put its interface here:

```java
public interface SingletonsContainer extends AutoCloseable{
    <T> T get(String name, Supplier<T> provider);
    void close() throws Exception {}
}
```

I use this approach at work for more than 2 years and only downside of it that your collegues tries to convince that spring is good you just need to cook it properly. 
Hopefully this will be useful for lost souls in spring mess.
