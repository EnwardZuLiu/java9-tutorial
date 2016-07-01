# A Guide to Java 9

This tutorial guides you through most of the new features Java 9 has and explains them with code snippets. You should read this as a introduction to some of the new features but not as a in-depth guide.

## Table of Contents

* [Modules](#modules)
* [JShell](#jshell)
* [Reactive Streams](#reactive-streams)
* [New Collections APIs](#new-collections-apis)
  * [Factory methods](#factory-methods)
  * [Arrays new methods](#arrays-new-methods)
* [New Streams operations](#new-streams-operations)
* [Additions to Optional](#additions-to-optional)
* [New HTTP API with HTTP/2 support](#new-http-api-with-http2-support)
* [Web Sockets API](#web-sockets-api)
* [New Process API](#new-process-api)
* [Miscellaneous](#miscellaneous)
* [Other things you may be interested about](#other-things-you-may-be-interested-about)


## Modules

Java 9 is primarily all about modules. By definition, modules are self describing collections of code and data. But in order to introduce modules let’s just say that modules are components which have packages and these have classes and interfaces (actually modules may have some other things, like configuration files, native code, resources, etc). What we need to know to start with modules is that they have a name, so let’s write a module, the module declaration is, by convention, inside a file named `module-info.java` and looks something like this:

```java
module org.foo.baz {
}
```

This is a module definition, notice the `module` keyword to define a module named `org.foo.bar`. This is the simplest module that we can make, it can be used for anyone else and only requires on module, the `java.base` module. The `java.base` module is a module inside the JDK and is like the `Object` class of modules, you will always used in every module, in there you will find things like `java.lang` that’s why it’s always used by any module. We can compile module-info.java with the normal `javac` command and the result will be a module descriptor named module-info.class which is used at runtime to resolve the dependencies. Let’s modify this module in order to export something so we can use it, assume we have three classes: First, Second and Third, each of these are in a different package with the same name but without the uppercase. Now if `Third` is only intended to work internally to help classes `First` and `Second` but is never meant to be used outside that context we shouldn’t export the package `org.foo.baz.third`, the module would be something like this:

```java
module org.foo.baz {
    exports org.foo.baz.first;
    exports org.foo.baz.second;
}
```

You can see a new keyword `exports` which is use to declare which packages are accessible from outside the module. In order to use those packages in a different module we need to declare that we need them, exports doesn’t make it public to everyone but only for the ones that require that module, so a new module that uses these packages would be like:

```java
module org.foo.bar {
    exports org.foo.bar.fourth;
    requires org.foo.baz;
}
```

This new module `org.foo.bar` requires `org.foo.baz` so you can make use of the classes `First` and `Second` in that module, also it requires `java.base` because every module require `java.base`. Notice the new keyword `requires` declaring the modules required. So far we can make a bunch of packages with a bunch of classes and we can encapsulate all within a module that gives a name to all that logic and defines somehow a new concept of accessibility. You need to take in mind that the work  `public` no longer means accessible, class `Third` could have all his members public but we can’t use them outside that module, even if we export package `org.foo.baz.third` we can’t use them without require that module, so that is an important changes to know in Java 9.

Before we continue is very important to know that the entire JDK has been modularized, so we need to require modules from the JDK to access the packages we need, besides `java.base` if we do an import of for example, some packages related to xml we won’t have permissions to use them, we need to require `java.xml` in our module definition. If you want to know what modules you will need to use from the JDK you can see the modules definition of each module or wait until your compilation fails because you don’t have access to some packages. So far what you will never need is the package inside the `java.base` module (`lang`, `util`, `math`, `io`, `nio`, `net`, `security`, `text`, `time`, `javax.crypto`, `javax.security` and `javax.net`).

Now with that clear, let’s continue with our examples. Another new concept is the usage of a module by transitivity, if we write a new module that requires `org.foo.bar` we will do it this way:

```java
module org.foo.zoo {
    requires org.foo.bar;
}
``` 
But what is we can make use of class First? We require a module that in its requirement list has the module with that class and package exported, can we use it?, the answer is no. The exports only works for those module that explicitly require the module, there is no transitivity of the exports. However we can make this possibly we a “new” keyword, let’s make only package `org.foo.baz.first` transitive, for that we need to change module `org.foo.baz`:

```java
module org.foo.baz {
    exports public org.foo.baz.first;
    exports org.foo.baz.second;
}
```

As you can see in the export we put our “new” `public` keyword and now we can use the First class from `org.foo.zoo` module.

Another concept that is worth mention is the selective export,  if we are making a complex system and we want some module to be able to use some packages for other module but those packages are not intended to be uses for anyone else we can make it possible:

```java
module org.foo.baz {
    exports public org.foo.baz.first;
    exports org.foo.baz.second to org.foo.bar;
}
```

Now, using the `to` keyword we define that package` org.foo.baz.second` will be exported but only for module `org.foo.bar` (it still needs to require to use it). This is useful for when we have an internal API but we want to use it for another module as well. One last concept that we will see is Services. Services are a way of deal with consumers and providers modules. Let’s assume that `org.foo.bar.fourth` has class Fourth which uses an API in `org.foo.bar.fourth.provider.AbstractProvider` that doesn’t have an implementation (you know some interface or abstract class) because the implementation will be done in another module so you can use different modules in a pluggable way in order to change the implementation. What we need to do is export all the necessary packages and make use of a new keyword:

```java
module org.foo.bar {
    exports org.foo.bar.fourth;
    exports org.foo.bar.fourth.provider;
    requires org.foo.baz;
    uses org.foo.bar.fourth.provider.AbstractProvider;
}
```

`AbstractProvider` is our service consumer and because of that `org.foo.bar.fourth.provider` needs to be exported as well. `uses` is a new keyword used in the modules declaration and makes reference to our service consumer in our module. What is left is the service provider with an implementation. That would be our `org.foo.zoo` module with some changes:

```java
module org.foo.zoo {
    requires org.foo.bar;
    provides org.foo.bar.fourth.provider.AbstractProvider with org.foo.zoo.ProviderImpl;
}
```

And that’s it, in `ProviderImpl` you most likely will use a `extends` or `implements` to `AbstractProvider`.

The module system along with many of the concepts that surround it it’s a huge change in the JDK and the way that Java programs are designed, but it goes beyond that this are some of the things you should be aware of:

* New tools have been added and some modified: So far you want to look to `jdeps`, `jmod`, `jlink`, `jpkg`, `java`, `javac` and `jtreg`.
* New file formats to pack the modules: `Modular JAR` and `JMOD`.
* Automatic modules: the solution to use libraries designed without modules in your module designed system.
* Profiles: You can create your custom snapshots with only the modules you need and nothing more. You can also use predefined modules designed to create snapshots, these are `compact1`, `compact2` and `compact3`.
* Restructure of the JDK and JRE: not only you can use modules defined in the JDK but also the structure has change, so say goodbye to `rt.jar` and `tools.jar` among others.
* Internal APIs have been encapsulated, they will be private in a future: The module system allows the JDK to hide its internal APIs, so you don’t want to use them anymore since some of them will be encapsulated into modules and not exported (not all at once, also there is a migration plan with replacements). Some of the most common are those that have `sun.*` and `*.internal.*`

One of the best resources about modules out there to the date are the JavaOne videos, you should go watch them, they cover most of the items listed above:

[JavaOne - Prepare for JDK 9] (https://www.youtube.com/watch?v=nBAUaOoBdGU)

[JavaOne - Introduction to Modular Development] (https://www.youtube.com/watch?v=a99RmjgG5Eo)

[JavaOne - Advanced Modular Development] (https://www.youtube.com/watch?v=YPQ2V-hQb8w)

[JavaOne - Project Jigsaw Under the Hood] (https://www.youtube.com/watch?v=xswtIp730Ho)

[JavaOne - Ask the Architects] (https://www.youtube.com/watch?v=jAL72EhLTXo)

[JavaOne - Project Jigsaw Hack Session] (https://www.youtube.com/watch?v=r2DeuDCCywM)

## JShell

JShell is a utility tool added in Java 9 to use Java in a REPL (read-eval-print-loop). JShell allow you to run Java commands without wrapping them in classes and methods (if you know languages like Python you already use REPL in those). JShell is not a new language or a new way to program in Java, it’s just a utility tool that comes very handy for those people trying to learn Java o for experienced ones trying to learn some new API o new languages features like the ones in this guide.
You can run JShell in the terminal/console since it is in the `/bin` folder and start writing code immediately, here are some examples (you can use tab for autocomplete code):

```java
jshell> 1+2
$1 ==> 3
jshell> int a = 3
a ==> 3
jshell> a+$1
$3 ==> 6
jshell> System.out.println("I have got: " + $3)
I have got: 6
jshell> 78
$5 ==> 78
jshell> $5 + $3
$5 ==> 84
```

So there a few things you should kwon, first of all, you don’t need semicolons you can avoid those if you don’t like them. Every expression would be stored in a temporary variable and its type will be inferred, as you can see the first thing I did was an addition between two numbers and the result was stored in a variable even if I didn’t specified that. Temporary variables start with the dollar sign followed by a number. The only thing you should be aware of is the implicit type conversion because it can be confused sometimes; you need to know how Java does the conversions:

```java
jshell> "5" + 5
$6 ==> "55"
jshell> "5" + 5.5
$7 ==> "55.5"
jshell> "5" + new Date();
$8 ==> "5Sun Jun 26 19:18:07 ART 2016"
```

This is important because the temporary variables have type (kind of obvious) so you really want to know what the type is in order to work with them:

```java
jshell> new Date()
$10 ==> Sun Jun 26 19:23:48 ART 2016
jshell> $10.getMonth()
$11 ==> 5
jshell> new URL("http://www.google.com").openConnection().getHeaderFields()
$12 ==> {Transfer-Encoding=[chunked], null=[HTTP/1.1 200 OK], Server=[gws], P3P=
[CP="This is not a P3P policy! See https://www.google.com/support/accounts/answe
r/151657?hl=en for more info."], Date=[Sun, 26 Jun 2016 22:29:26 GMT], Accept-Ra
nges=[none], X-Frame-Options=[SAMEORIGIN], Cache-Control=[private, max-age=0], V
ary=[Accept-Encoding], Set-Cookie=[NID=80=AeE02ugSxND4hjh7s0CGGrJJswNNHtgrrO-coh
SUYFMObveYVkxrnNX4jmG4H1wtgFBcS5BHIrXskt_jQiZI-5l9xl3YaV7aJQ1vLRq69ULBoMSk988e5v
K29a9Ejhrb; expires=Mon, 26-Dec-2016 22:29:26 GMT; path=/; domain=.google.com.ar
; HttpOnly], Expires=[-1], X-XSS-Protection=[1; mode=block], Content-Type=[text/
html; charset=ISO-8859-1]}
```

In the last example the result is a `Map<String, List<String>>`, so yeah, probably the types are something you should be really aware of, hopefully as far as I know the final version will inform you the type of the variables. One final thought about this last example is that you don’t have to write all the logic for exceptions, you just make the code and if something crashes it would be informed.

You can see more information of JShell by typing `/help` inside it or in the `/repl/samples` folder.

This tool is not only a standalone application that you can run in the terminal but it’s also an API; most likely you will never use it as an API but be ready to use JShell inside other tools such as IDEs.

## Reactive Streams

Java 9 comes with a reactive streams API that corresponds to the [reactive streams specification](http://www.reactive-streams.org/). There are several third-party APIs that implements reactive streams in Java but in Java 9 they want to add support inside the JDK to reactive streams and empower the streams in Java 8. It's not in the scope of this guide to teach the concepts of reactive streams since there are several resources out there, you can start [with this one](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754). All the interfaces of this API are inside the `Flow` class so let's see what this class is about.

There are four interfaces: `Publisher<T>`, `Subscriber<T>`, `Subscription` and `Processor<T, R>`. It's important to know that all this interfaces are designed in an asynchronous way so they are a bit different for the one that we are familiar in Java. So here is how we start implementing those interfaces:

```java
import java.util.concurrent.Flow.*;

public class Main {
    public static void main(String[] args) throws Exception {
        DummyPublisher<String> publisher = new DummyPublisher<String>();
        DummySubscriber<String> subscriber = new DummySubscriber<String>();
        publisher.subscribe(subscriber);
    }
}

public class DummyPublisher<T> implements Publisher<T> {
    //See how subscribe() doesn't return a Subscription
    public void subscribe(Subscriber<? super T> subscriber) {}
}

public class DummySubscriber<T> implements Subscriber<T> {

    //This is the callback method
    public void onSubscribe(Subscription subscription) {}
    public void onComplete() {}
    public void onError(Throwable throwable) {}
    public void onNext(T item) {}
}

public class DummySubscription implements Subscription {
    public void request(long n) {}
    public void cancel() {}
}
```

It's very important to notice the asynchronous design in here, you will see for example that every method return void, when normally methods like subscribe() will return a Subscription. Here you need to use callbacks to do a “return”, let’s see a basic implementation of a subscription:

```java
import java.util.concurrent.Flow.*;

public class Main {
    public static void main(String[] args) throws Exception {
        DummyPublisher<String> publisher = new DummyPublisher<String>();
        DummySubscriber<String> subscriber = new DummySubscriber<String>();
		
        publisher.subscribe(subscriber);
        publisher.subscribe(subscriber); //Not allowed
    }
}

public class DummyPublisher<T> implements Publisher<T> {

    //We will allow only one supplier
    private boolean subscribed;
	
    public DummyPublisher() {
        subscribed = false;
    }
	
    //See how subscribe() doesn't return a Subscription
    public void subscribe(Subscriber<? super T> subscriber) {
        if( subscribed == false) {
            subscribed = true;
            // This is the equivalent to return the subscription, we call a callback
            subscriber.onSubscribe(new DummySubscription());
        } else {
            System.out.println("This subscriber it's already subscribed");
            subscriber.onError(new IllegalStateException("We don't accept more subscriber, sorry"));
        }
    }
}

public class DummySubscriber<T> implements Subscriber<T> {
	
    private Subscription subscription;
	
	//This is the callback method
	public void onSubscribe(Subscription subscription) {
		this.subscription = subscription;
		System.out.println("Successfully subscribed");
	}
	
	public void onComplete() {}
	
	public void onError(Throwable throwable) {
        System.out.println("Can subscribe publisher says: " + throwable.getMessage());
	}
	
	public void onNext(T item) {}

}

public class DummySubscription implements Subscription {
    public void request(long n) {}
	public void cancel() {}
}
```

That code shows how you can subscribe one subscriber and also how you can't deny one subscriber to subscribe that specific publisher. Now that we can subscribe to one publisher let’s see how we can publish information and also cancel the subscription so we stop listening to the publisher.


TODO


## New Collections APIs

### Factory methods

Java 9 comes with A LOT of static methods defined in the interfaces of the structures like `List`, `Set` and `Map`. These static methods offers a easy way to populate the structures, lets see some examples compared to the Java 8 way.

```java
//Java 8
List<String> list = Arrays.asList("a", "b", "c");
//Java 9
List<String> listJava9 = List.of("a", "b" , "c");
	
//Java 8
Set<String> setJava8 = new HashSet<String>(Arrays.asList("a", "b", "c"));
//Java 9
Set<String> setJava9 = Set.of("a", "b", "c");

//Java 8
Map<Integer, String> mapJava8 = new HashMap<Integer, String>();
mapJava8.put(1, "a");
mapJava8.put(2, "b");
mapJava8.put(3, "c");

//Java 8 (sorter)
Map<Integer, String> mapJava8Sorter = new HashMap<Integer, String>() {
    {
        put(1, "a");
        put(2, "b");
        put(3, "c");
    }
};
  
//Java 9
Map<Integer, String> mapJava9 = Map.of(1, "a", 2, "b", 3, "c");
	
//Java 9 (If you want to put more than 10 elements)
Map<Integer, String> mapJava9MoreThan10 = Map.ofEntries(
    Map.entry(1, "a"), 
    Map.entry(2, "b"), 
    Map.entry(3, "c")
);
```

So the method `of` seems a good alternative the populate structures more handy. I said that this new APIs had a lot a new methods but only show you `of` (and `ofEntries`), the thing is, it exist one method of for each number or arguments you pass to it, and up to ten elements the varargs are used, but in the case of `Map`, the varargs are not a solution, so that's why we need to use the method `ofEntries`. Some thing you should know is thar the structures generated by `of` are immutable and don't support `null` values, so they don't quite replace the put method, [more info](https://www.youtube.com/watch?v=OJrIMv4dAek).

### Arrays new methods

The utility class `Arrays` now comes with four new methods: `equals`, `compare`, `compareUnsigned` and `mismatch`. While those names sounds familiar the parameters are different, here's what they do.

```java
int[] a = {1,2,-7,6,8};
int[] b = {1,9,7,1,2};

System.out.println( Arrays.equals(a, 0, 1, b, 3, 4) );
// true because the first two elements of a are equals to the last two of b
  
System.out.println( Arrays.compare(a, 0, 1, b, 3, 4) );
// 0 because the specified ranges are equals
  
System.out.println( Arrays.compare(a, b) );
// -1 because 1 is lesser than 6
  
System.out.println( Arrays.compareUnsigned(a, 2, 2, b, 2, 2) );
// 0 because unsigned -7 is equals to 7
  
System.out.println( Arrays.mismatch(a, b) );
// 1 because the index of the first mismatch (2 and 9) you can specify ranges
```

## New Streams operations

Streams added in Java 8 now had new operators. These are `takeWhile`, `dropWhile`, `ofNullable` and `iterate`.

`takeWhile` and `dropWhile` are well known operations in other programming languages like Haskell. `takeWhile` Returns a stream consisting of the longest prefix of elements from the original stream that match a given predicate.

```java
List<Integer> list = List.of(0, 1, 2, -1, -2);
list.stream().takeWhile(x -> x <= 1).forEach(System.out::println);
// 0 1
```

As you can see, the `forEach` operator only applied for the first and second element of the stream, because the third element is greater than 1 and not match the `takeWhile` predicate (`x <= 1`), also, because of that, numbers like -1 are ignored since `takeWhile` returns only the longest prefix. This change in unordered streams, in this case `takeWhile` returns a subset of elements that match the given predicate.

```java
Set<Integer> set = Set.of(0, 1, 2, -1, -2);
set.stream().takeWhile(x -> x <= 1).forEach(System.out::println);
// 0 -1 1 -2
```

`dropWhile` works similary to `takeWhile`. For ordered streams, it returns a stream consisting of the remaining elements of this stream after dropping the longest prefix of elements that match the given predicate, and for unordered streams, a stream consisting of the remaining elements of this stream after dropping a subset of elements that match the given predicate.

```java
List<Integer> list = List.of(0, 1, 2, -1, -2);
list.stream().dropWhile(x -> x <= 1).forEach(System.out::println);
// 2 -1 -2
  
Set<Integer> set = Set.of(0, 1, 2, -1, -2);
set.stream().dropWhile(x -> x <= 1).forEach(System.out::println);
// 2
```

`ofNullable` returns a stream containing a single non-null element, if the element is null then returns an empty stream.

```java
List<Integer> indexes = List.of(0, 1, 2, -1, -2);

Map<Integer, List<String>> map = Map.of(
    1, List.of("bar", "foo"), 
    2, List.of("foo", "baz"), 
    3, List.of("baz", "bar")
);

indexes.stream().flatMap(index -> Stream.ofNullable(map.get(index))).forEach(System.out::println);

//[bar, foo]
//[foo, baz]
```

Notice how `map.get(index)` with index != 1 or 2 creates a empty stream that doesn't affect the execution of our program.

The `iterate` operator already exist in Java 8, but in Java 9 we can pass a Predicate to limit the execution of the `iterate` operator in a similar fashion of `takeWhile` and `dropWhile` works.

```java
//Java 8
Stream.iterate(1, n -> n + 1).limit(10).forEach(System.out::print); // 1 2 3 4 5 6 7 8 9 10

//Java 9
Stream.iterate(1, n -> n <= 10, n -> n + 1).forEach(System.out::print); // 1 2 3 4 5 6 7 8 9 10
```

A final note about this new methods is that you probably don't want to mix `takeWhile` and `dropWhile` with parallel streams, at least if you are trying to work with the longest prefix, because depending on how the instructions are executed you will end up with different results.

## Additions to Optional

We have some new methods for `Optional` there are no big changes in here but some of them are very useful. First of all, the method `ifPresentOrElse()` was added, as you may expect it's the same that `ifPresent()` but with an else in its second argument:

```java
someOptional.ifPresentOrElse(
	someValue -> System.out.println("Value:  " + someValue + " exist"), 
    () -> System.out.println("Value doesn't exist")
);
```

Another new method that is very handy it's the new `or()` method, now we can choose a value for some `Optional` if it doesn't have anything in it.

```java
Optional<Integer> optional = Optional.empty();
optional.or(() -> Optional.of(2)).ifPresent(s -> System.out.println(s)); //2
```

As you can see the `or()` method returns a new `Optional` since the first one was empty, this can be very useful to chain Optionals when you trying to return one in a method:

```java
public Optional<String> getFromCache(String key) {
	return cache.getValueFor(key)
			.or(() -> getFromDatabase(key));
}

public Optional<String> getFromDatabase(String key) {
	return dababase.getValueFor(key);
}
```

The last thing that is worth mention it is the possibility of transforming `Optional` to `Stream`.

```java
Optional<Integer> optional = Optional.of(1);
optional.stream().forEach(System.out::println); //1
```

This can be useful when dealing with collections of objects that have some method returning an `Optional`:

```java
class myObject {
	public Optional<Integer> getOptional() {
		if(Math.random() > 0.5) {
			return Optional.of(1);
		}
		return Optional.empty();
	}
}

List<myObject> list = List.of( new myObject(), new myObject(), new myObject(), new myObject() );

Stream<Integer> resultStream = list.stream()
		.map(mo -> mo.getOptional()) //Here we have a Stream<Optional<Integer>>
		.filter(Optional::isPresent) //We discard the empty ones
		.map(Optional::get); //We return the Integers

resultStream.forEach(System.out::println); //Will print some "1"
```

That is all the new stuff in the `Optional` class.

## New HTTP API with HTTP/2 support

Java 9 comes with a new HTTP client that supports HTTP/2 and Async request. It still supports HTTP 1.1 and the API is almost protocol agnostic. So before you continue if you have no clue what HTTP/2 does you can look it up or see [this video] (https://www.youtube.com/watch?v=QpLtBftqM04) which also talks a bit about Java. Let's start with the basics, do gets and posts.

```java
import static java.net.http.HttpResponse.asString;
import static java.net.http.HttpResponse.asFile;
import static java.net.http.HttpResponse.asByteArray;
import static java.net.http.HttpRequest.fromString;

//I included the main to show you some of the Exceptions you should be aware of 
public static void main(String[] args) throws URISyntaxException, IOException, InterruptedException, ExecutionException {

    //GET
    HttpResponse response = HttpRequest
        .create(new URI("https://github.com/")) // This returns a HttpRequest.Builder the class that is used to create HttpRequest
        .headers("Foo", "foo", "Bar", "bar") //This also returns a HttpRequest.Builder. "headers()" uses varargs
        .GET() // This is where we get out HttpRequest
        .response(); // This give us a HttpResponse

    int statusCode = response.statusCode();
    
    // You can get the response as a String
    String responseBody = response.body(asString());
    System.out.println(responseBody);
    
    // Or in a file
    Path path = Paths.get("/path/to/file");
    Path responseBodyFile = response.body(asFile(path));
    System.out.println(responseBodyFile);
    
    // Or as a byte[]
    byte[] responseBodyBytes = response.body(asByteArray());
    System.out.println(responseBodyBytes.length);
    
    // POST
    HttpResponse response = HttpRequest
        .create(new URI("http://www.google.com"))
        .body(fromString("param1=foo,param2=bar"))
        .POST()
        .response();
		   
    int statusCode = response.statusCode();
    if(statusCode == 200) {
        System.out.println(response.body(asString()));
    } else {
        System.out.println("You got a " + statusCode + " code!!");
    }
}
```

The arguments we pass to `response.body()` are called body processors and there are some others that you can use, for example if you don't want the body you can use the `ignoreBody()` body processor. The body processors are the ones who retrieve the response body and they must be called to ensure that the connection can be re-used.

So far so good, but when you use `response()` you're blocking your program because it is a synchronous call. To make the calls asynchronous we need to use `responseAsync()` and store the result in a `CompletableFuture<HttpResponse>`. Let’s do it:

```java
HttpRequest request = HttpRequest
    .create(new URI("https://github.com/"))
    .body(noBody())
    .GET();

CompletableFuture<HttpResponse> cfResponse = request.responseAsync();

while(!cfResponse.isDone()) {
    System.out.println("This code is executing while waiting for the response");
    Thread.sleep(100);
}

HttpResponse response = cfResponse.get();
System.out.println(response.body(asString()));
```

This example doesn't really shows the beauty of Async request with the CompletableFuture API, let's request asynchronous a list of web pages and count the amount of characters (This example is based on the one given by the javadocs of `HttpRequest`):

```java
//First we declare our list of web pages
List<URI> sites = List.of( new URI("https://github.com/"), new URI("https://www.reddit.com/") );
 
List<CompletableFuture<File>> cfs = sites
    .parallelStream()
    .map(site -> {
        return HttpRequest
            .create(site)
            .GET()
            .responseAsync() // We do Async request to each of them 
            .thenCompose(response -> {
                Path path = Paths.get("/path/to/file", site.getHost() + ".txt"); // i.e of file name github.com.txt
                if (response.statusCode() == 200) {
                    return response.bodyAsync(asFile(path)); // We store the results in as file
                } else {
                    return CompletableFuture.completedFuture(path);
                }
            })
            .thenApply((Path path) -> {
                return path.toFile(); // We convert from Path to File
            });
    })
    .collect(Collectors.toList());

We wait until everything is done
CompletableFuture.allOf(cfs.toArray(new CompletableFuture<?>[0])).join();

//For each result we show the length of the file
cfs.stream()
    .map( cf -> {
        try {
            return cf.get().length();
        } catch(Exception e) { 
            return "Unexpected error";
        }
    })
    .forEach(System.out::println);
```

That's for simple requests to public sites, but what if we need to do a more complex request? probably with some cookies or SSL. In that case you will need to build a `HttpClient` and then create the `HttpRequest` from there. `HttpClient` has a lot of methods for configure you client, so let's quickly see some of them:

```java
HttpResponse response = HttpClient.create() // Creates a HttpClient.Builder from where you can build immutable HttpClient   
    .cookieManager(cookieManager) //CookieManager from java.net (all these methods returns a HttpClient.Builder)
    .sslContext(sslContext) // SSLContext from javax.net.ssl
    .sslParameters(sslParameters) // SSLParameters from javax.net.ssl
    .authenticator(authenticator) // Authenticator from java.net
    .proxy(ProxySelector.of(new InetSocketAddress("proxy", 80))) // ProxySelector from java.net has a factory method now
    .followRedirects(HttpClient.Redirect.SAME_PROTOCOL) // Redirect can be ALWAYS, NEVER, SAME_PROTOCOL or SECURE
    .build() // returns a HttpClient no longer a HttpClient.Builder
    .request(URI.create("https://www.github.com/")) // returns a HttpRequest.Builder
    .GET() // returns a HttpRequest
    .response(); // finally! our HttpResponse
```

Many of the parameters you use are in Java for a while, the main idea of that code was to show you how we can chain methods one after another to configure our client and make the request. If you need to do some of this configuration the best you can do is look into the javadocs for specific information about all the methods.

The last topic we will see in the HTTP Client is the support for multiple responses. Multiple responses is a common thing that can happen in HTTP/2 when a server makes a push, in this cases the client tells the server that he is ready for a push and needs to handle the multiple responses for a single request. What we need to do is to get the response with a new method called `multiResponseAsync()`, this method doesn’t accept a BodyProcessor as a parameter because now we have multiple responses, what we need to use is a MultiProcessor.

```java
Path path = Paths.get("path/to/file");

CompletableFuture<Map<URI,Path>> cf = HttpRequest
    .create(new URI("https://www.google.com/"))
    .version(HttpClient.Version.HTTP_2)
    .GET()
    .multiResponseAsync( HttpResponse.multiFile(path) );

while(!cf.isDone()) {
    System.out.println("This code is executing while waiting for the response");
    Thread.sleep(100);
}

Map<URI,Path> results = cf.join();
```

The result will be a Map<URI,Path> beacuse the server returs different resources located in diferent URI and our MultiProcessor store each one of them if a different file.

Final note: remember that Java 9 only adds a client with support for HTTP/2 the server side of the history will eventually comes with Servlet 4.0 for the Java EE platform.

## Web Sockets API

TODO

## New Process API

The Process API has been renewed in order to make the code that use it less platform dependant. A new class has been added that can be used to handle the processes: list processes, get PID of a process, destroy, get parents, get childrens and get information about a process.

```java
// Create a ProcessHandler for the JVM Process
ProcessHandle currentProcess = ProcessHandle.current();

// Shows JVM PID
System.out.println( currentProcess.getPid() );
// Shows JVM Parent PID
currentProcess.parent().ifPresent(parent -> System.out.println(parent));
// Shows all the children processes of the JVM process
currentProcess.children().forEach( processHandler -> System.out.println(processHandler.getPid()));
// Shows all the descendants processes of the JVM process (children and children's children)
currentProcess.descendants().forEach( processHandler -> System.out.println(processHandler.getPid()));

// Gets information about the process
ProcessHandle.Info processInfo = currentProcess.info();
System.out.println( processInfo ); 
/* Will print "[user: Optional[Usuario\Usuario], 
 * cmd: C:\Program Files (x86)\Java\jdk-9\bin\java.exe, 
 * startTime: Optional[2016-06-14T23:36:52.836Z], 
 * totalTime: Optional[PT0.2028013S]]"
 */

// User of the process
processInfo.user().ifPresent(user -> System.out.println(user));
// Executable pathname of the process
processInfo.command().ifPresent(command -> System.out.println(command));
// Total cputime accumulated of the process
processInfo.totalCpuDuration().ifPresent(duration -> System.out.println(duration));
// Start time of the process
processInfo.startInstant().ifPresent(startInstant -> System.out.println(startInstant));
// String array of the arguments of the process
processInfo.arguments().ifPresent(arguments -> Arrays.stream(arguments).forEach(System.out::println));
// All processes visible to the current process
ProcessHandle.allProcesses().forEach(System.out::println);
// Gets the process with PID = NUMBER (a long value) and detroy it.
ProcessHandle.of( NUMBER ).ifPresent(processHandle -> processHandle.destroy());

// Here is how to wait for a process to end and then show a message on exit

//Get ProcessHandler for process with PID = 3816
Optional<ProcessHandle> anotherProcess = ProcessHandle.of( 3816 ); 

if( anotherProcess.isPresent() ) {
    // ProcessHandler.onExit() returns a CompetableFuture<ProcessHandler> and we register a message when complete
    CompletableFuture<ProcessHandle> competable = anotherProcess.get().onExit().whenComplete( 
        (p,t) -> System.out.println("process " + p.getPid() + " finished.") 
    );
    // Waits (blocks thread) for the process to end and then shows the message 
    competable.get(); 
}
```

That's pretty much it, you have almost the same methods in the `Process` class if you want to do something like this and also handle input and output of the process. Anf if you create a `Process` with `ProcessBuilder.start()` or `Runtime.exec()` you can get its handler with `toHandle()`.

## Miscellaneous

### Improvement to try-with-resources

One small but nice to know change that have been introduced in Java 9 is in the try-with-resources statement. In the old try-with-resources all the resources should be defined in the try statement, something like this:

```java
//Java 8 example
final BufferedReader br1 = new BufferedReader(new InputStreamReader(System.in));
BufferedReader br2 = new BufferedReader(new InputStreamReader(System.in)); //effectively final

try (BufferedReader r1 = br1;
    BufferedReader r2 = br2; ) {
    String line = r1.readLine();
    System.out.println(line);
    line = r2.readLine();
    System.out.println(line);
} catch (IOException e) {
    e.printStackTrace();
}
```

Now, if you have a resource as a final or effectively final variable, you can use it directly in the try-with-resources statement, just like this:

```java
//Java 9 example
final BufferedReader br1 = new BufferedReader(new InputStreamReader(System.in));
BufferedReader br2 = new BufferedReader(new InputStreamReader(System.in)); //effectively final

try (br1; br2) {
    String line = br1.readLine();
    System.out.println(line);
    line = br2.readLine();
    System.out.println(line);
} catch (IOException e) {
    e.printStackTrace();
}
```

See how we avoid creating new variable in the try statement and the code it's sorter.

Note: if you don't know, a effectively final variable is [a variable or parameter whose value is never changed after it is initialized is effectively final] (http://docs.oracle.com/javase/tutorial/java/javaOO/localclasses.html)

### Diamond operator for anonymous classes

This it is only possible if the argument type of the inferred type is denotable. Now you can declare anonymous classes with empty diamond operators just like this:

```java
Comparator<Double> comparator = new Comparator<>() {
	@Override
	public int compare(Double d1, Double d2) {
		return Double.compare(d1, d2);
	}
};
```

Notice how we do not specify the type inside the diamond when creating the new object.

### Private methods for interfaces

In addition to static and default methods in Java 8, now we can add private methods to interfaces in order to use them when we want to shere code between non abstract methods.

```java
interface MyInterface {
	//Static method in Java 8
    static void sayHi() {
		talking();
		System.out.println("Hi");
	}
	//Default method in Java 8
	default void sayGoodBye() {
		talking();
		def();
		System.out.println("Good bye");
	}
	//Private method in Java 9, it is used by multiple methods in the interface.
	//Cannot be used outside the interface
    private static void talking() {
        System.out.println("I am going to talk");
    }
    //We can use them without static keyword
    private void def() {
        System.out.println("Default method");
    }
}

class MyObject implements MyInterface {
	public void sayHi() {
		MyInterface.super.sayGoodBye();
	}
}



MyObject myObject = new MyObject();
MyInterface.sayHi();
myObject.sayGoodBye();
```

### Underscore identifier is gone

Just a small change that can break your code when you migrate to Java 9 it is the underscore as a keyword, that means that you cannot use it any longer as a identifier:

```java
String _ = new String(); //won't compile in Java 9
```

This it is an isolated change and we will see the "why" in future releases of the JDK, as far as I know they plan to do something like this:

```java
Map<String, _> = new HashMap<String, _>();
```

But we can't know exactly what this is or if it is going to be implemented.

### @SafeVarargs on private instance methods

In Java 7 the annotation `@SafeVarargs` was introduced in the JDK to denoted that one method that uses varargs it is going to use them properly and won't generate heap  pollution. We can see this annotation being used in methods like for example `Arrays.asList()`.

```java
@SafeVarargs
@SuppressWarnings("varargs")
public static <T> List<T> asList(T... a) {
    return new ArrayList<>(a);
}
```

This annotation can only be used in methods that cannot be overridden (static, final, private, constructors) however private methods were omitted in Java 7 and they are adding that with Java 9.

### Streamified APIs

In Java 9 as we see with Optionals or the HTTP Client the APIs are supporting Streams more an more, you will find that several classes now have new methods that breturn some kind of Stream, this is part of a modernization of the APIs and we can find small changes all over the Java libraries. Some examples are:

```java
"Some String".chars()
	.filter(Character::isLowerCase)
	.mapToObj(c -> new String(Character.toChars(c)))
	.forEach(System.out::print); //ometring

LocalDate.now()
	.datesUntil(LocalDate.of(2017, 10, 20), Period.ofDays(1))  
	.forEach(System.out::println); //All the date from now to that day
```

## Other things you may be interested about

### JavaDoc.Next

[JavaDoc.Next](http://openjdk.java.net/projects/javadoc-next/) it is a project to improve the JavaDocs, it include:
* HTML5 Output
* Updated Doclet API
* New search box to search among the JavaDocs

### String Enhancements

The String performace has been improved. [more info](https://www.youtube.com/watch?v=_evzaAkd594)
