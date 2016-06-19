# A Guide to Java 9

This tutorial guides you through all the new features Java 9 has and it explain it with code snippets. 

## Table of Contents

* [Modules](#modules)
* [Reactive Streams](#reactive-streams)
* [New Collections APIs](#new-collections-apis)
  * [Factory methods](#factory-methods)
  * [Arrays new methods](#arrays-new-methods)
  * [New Streams operations](#new-streams-operations)
* [New Http Client](#new-http-client)
* [New Process API](#new-process-api)
* [Unified Logger](#unified-logger)
* [Changes in the private/public API](#changes-in-the-privatepublic-api)


## Modules

Java 9 is primarily all about modules. By definition, modules are self describing collections of code and data. But in order to introduce modules let’s just say that modules are components which have packages which have classes/interfaces (modules may have some other things, like configuration files, native code, resources, etc). What we need to know to start with modules is that they have a name, a list of the modules which it depends on and a list of the packages that are intended for external (to that module) use.  So let’s write a module, the module declaration is, by convention, inside a file named `module-info.java` and looks something like this:

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

And that’s it, in `ProviderImpl` you most likely will use a `extends` or `implements` to `AbstractProvider`

TODO

One of the best resources about modules out there to the date are the JavaOne videos, you should go watch them:

[JavaOne - Prepare for JDK 9] (https://www.youtube.com/watch?v=nBAUaOoBdGU)

[JavaOne - Introduction to Modular Development] (https://www.youtube.com/watch?v=a99RmjgG5Eo)

[JavaOne - Advanced Modular Development] (https://www.youtube.com/watch?v=YPQ2V-hQb8w)

[JavaOne - Project Jigsaw Under the Hood] (https://www.youtube.com/watch?v=xswtIp730Ho)

[JavaOne - Ask the Architects] (https://www.youtube.com/watch?v=jAL72EhLTXo)

[JavaOne - Project Jigsaw Hack Session] (https://www.youtube.com/watch?v=r2DeuDCCywM)




## Reactive Streams

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

### New Streams operations

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
  
  //I hope you already read about the "of" method
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

## New Http Client

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
     cmd: C:\Program Files (x86)\Java\jdk-9\bin\java.exe, 
     startTime: Optional[2016-06-14T23:36:52.836Z], 
     totalTime: Optional[PT0.2028013S]]"
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

  //This isn't necessary but you can see the new ifPresentOrElse method in the Optional class ;)
  anotherProcess.ifPresentOrElse(
    process -> System.out.println("Process  " + process.getPid() + " exist"),
    () -> System.out.println("Such a process doesn't exist")
  );

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
