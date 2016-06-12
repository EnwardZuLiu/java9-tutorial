# A Guide to Java 9

This tutorial guides you through all the new features Java 9 has and it explain it with code snippets. 

## Table of Contents

* [New Streams operations](#New Streams operations)

## New Streams operations

Streams added in Java 8 now had new operators. These are `takeWhile`, `dropWhile`, `ofNullable` and `iterate`.

`takeWhile` and `dropWhile` are well known operations in other programming languages like Haskell. `takeWhile` Returns a stream consisting of the longest prefix of elements from the original stream that match a given predicate.

```java
  List<Integer> list = Arrays.asList(0, 1, 2, -1, -2);
  list.stream().takeWhile(x -> x <= 1).forEach(System.out::println);
  // 0 1
```

As you can see, the `forEach` operator only applied for the first and second element of the stream, because the third element is greater than 1 and not match the `takeWhile` predicate (`x <= 1`), also, because of that, numbers like -1 are ignored since `takeWhile` returns only the longest prefix. This change in unordered streams, in this case `takeWhile` returns a subset of elements that match the given predicate.

```java
  Set<Integer> set = new HashSet<Integer>() {
  	{
  		add(0);
  		add(1);
  		add(2);
  		add(-1);
  		add(-2);
  	}
  };
  set.stream().takeWhile(x -> x <= 1).forEach(System.out::println);
  // 0 -1 1 -2
```

`dropWhile` works similary to `takeWhile`. For ordered streams, it returns a stream consisting of the remaining elements of this stream after dropping the longest prefix of elements that match the given predicate, and for unordered streams, a stream consisting of the remaining elements of this stream after dropping a subset of elements that match the given predicate.

```java
  List<Integer> list = Arrays.asList(0, 1, 2, -1, -2);
  list.stream().dropWhile(x -> x <= 1).forEach(System.out::println);
  // 2 -1 -2
  
  Set<Integer> set = new HashSet<Integer>() {
  	{
  		add(0);
  		add(1);
  		add(2);
  		add(-1);
  		add(-2);
  	}
  };
  set.stream().dropWhile(x -> x <= 1).forEach(System.out::println);
  // 2
```

`ofNullable` returns a stream containing a single non-null element, if the element is null then returns an empty stream.

```



