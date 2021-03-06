---
layout: global
title: Spark Java Programming Guide
---

The Spark Java API implements all of the Spark features available in the Spark Scala API. 

This Spark Java Programming Guide shows you how to use Java to implement the Spark features described in the [Spark Scala Programming Guide](scala-programming-guide.html).

To learn the basics of Spark, we recommend that Java developers first read through the [Spark Scala Programming Guide](scala-programming-guide.html) before reading this manual. The [Spark Scala Programming Guide](scala-programming-guide.html) is easy to follow even if you don't know Scala. 

The Spark Java API is defined in the
[`org.apache.spark.api.java`](api/java/index.html?org/apache/spark/api/java/package-summary.html) package, and includes a [`JavaSparkContext`](api/java/index.html?org/apache/spark/api/java/JavaSparkContext.html) for
initializing Spark and [`JavaRDD`](api/java/index.html?org/apache/spark/api/java/JavaRDD.html) classes. 

These classes support the same methods as their Scala counterparts but take Java functions and return Java data and collection types. The main differences have to do with passing functions to RDD operations (*e.g.*, `map`) and handling RDDs of different types, as discussed next.

# Key Differences in the Java API

There are a few key differences between the Java and Scala APIs:

* Java does not support anonymous or first-class functions, so functions are passed using anonymous classes that implement the
  [`org.apache.spark.api.java.function.Function`](api/java/index.html?org/apache/spark/api/java/function/Function.html),
  [`Function2`](api/java/index.html?org/apache/spark/api/java/function/Function2.html), etc. interfaces.

* To maintain type safety, the Java API defines specialized Function and RDD classes for key-value pairs and doubles. For example, 
  [`JavaPairRDD`](api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html)
  stores key-value pairs.

* Some methods are defined on the basis of the passed function's return type.
For example `mapToPair()` returns
[`JavaPairRDD`](api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html),
and `mapToDouble()` returns
[`JavaDoubleRDD`](api/java/index.html?org/apache/spark/api/java/JavaDoubleRDD.html).

* RDD methods like `collect()` and `countByKey()` return Java collection types, such as `java.util.List` and `java.util.Map`.

* Key-value pairs, which are simply written as `(key, value)` in Scala, are represented
by the `scala.Tuple2` class and need to be created using `new Tuple2<K, V>(key, value)`.

## RDD Classes

Spark defines additional operations on RDDs of key-value pairs and doubles, such as `reduceByKey`, `join`, and `stdev`.

In the Scala API, these methods are automatically added using Scala's
[implicit conversions](http://www.scala-lang.org/node/130) mechanism.

In the Java API, the extra methods are defined in the
[`JavaPairRDD`](api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html)
and [`JavaDoubleRDD`](api/java/index.html?org/apache/spark/api/java/JavaDoubleRDD.html)
classes.  RDD methods like `map` are overloaded by specialized `PairFunction`
and `DoubleFunction` classes, allowing them to return RDDs of the appropriate types.  

Each common method like `filter` and `sample` is implemented by its own specialized RDD class, so filtering a `PairRDD` returns a new `PairRDD`, etc. This achieves the "same-result-type" principle used by the [Scala collections
framework](http://docs.scala-lang.org/overviews/core/architecture-of-scala-collections.html).

## Function Interfaces

The following table lists the function interfaces used by the Java API, located in the
[`org.apache.spark.api.java.function`](api/java/index.html?org/apache/spark/api/java/function/package-summary.html)
package. Each interface has a single abstract method, `call()`.

<table class="table">
<tr><th>Class</th><th>Function Type</th></tr>

<tr><td>Function&lt;T, R&gt;</td><td>T =&gt; R </td></tr>
<tr><td>DoubleFunction&lt;T&gt;</td><td>T =&gt; Double </td></tr>
<tr><td>PairFunction&lt;T, K, V&gt;</td><td>T =&gt; Tuple2&lt;K, V&gt; </td></tr>

<tr><td>FlatMapFunction&lt;T, R&gt;</td><td>T =&gt; Iterable&lt;R&gt; </td></tr>
<tr><td>DoubleFlatMapFunction&lt;T&gt;</td><td>T =&gt; Iterable&lt;Double&gt; </td></tr>
<tr><td>PairFlatMapFunction&lt;T, K, V&gt;</td><td>T =&gt; Iterable&lt;Tuple2&lt;K, V&gt;&gt; </td></tr>

<tr><td>Function2&lt;T1, T2, R&gt;</td><td>T1, T2 =&gt; R (function of two arguments)</td></tr>
</table>

## Storage Levels

RDD [storage level](scala-programming-guide.html#rdd-persistence) constants like `MEMORY_AND_DISK` are
declared in the [org.apache.spark.api.java.StorageLevels](api/java/index.html?org/apache/spark/api/java/StorageLevels.html) class. To define your own storage level, use StorageLevels.create(...). 

# Other Features

The Java API supports other Spark features, including
[accumulators](scala-programming-guide.html#accumulators),
[broadcast variables](scala-programming-guide.html#broadcast-variables), and
[caching](scala-programming-guide.html#rdd-persistence) (persistence).

# Upgrading From Pre-1.0 Versions of Spark

In Spark version 1.0 the Java API was refactored to better support Java 8 lambda expressions. Users upgrading from older Spark versions should note the following changes:

* All `org.apache.spark.api.java.function.*` classes have been changed from abstract classes to interfaces. This means that concrete implementations of these `Function` classes must use `implements` instead of `extends`.

* Certain transformation functions now have multiple versions depending on their return types. In Spark core, the map functions (map, flatMap, mapPartitons) have type-specific versions, *e.g.*, 
on the return type. In Spark core, the map functions (`map`, `flatMap`, and
`mapPartitons`) have type-specific versions, e.g. 
[`mapToPair`](api/java/org/apache/spark/api/java/JavaRDDLike.html#mapToPair(org.apache.spark.api.java.function.PairFunction))
and [`mapToDouble`](api/java/org/apache/spark/api/java/JavaRDDLike.html#mapToDouble(org.apache.spark.api.java.function.DoubleFunction)).
Spark Streaming also uses the same approach, e.g.,  [`transformToPair`](api/java/org/apache/spark/streaming/api/java/JavaDStreamLike.html#transformToPair(org.apache.spark.api.java.function.Function)).

# Example

As an example, we'll implement word count using the Java API.

{% highlight java %}
import org.apache.spark.api.java.*;
import org.apache.spark.api.java.function.*;

JavaSparkContext ctx = new JavaSparkContext(...);
JavaRDD<String> lines = ctx.textFile("hdfs://...");
JavaRDD<String> words = lines.flatMap(
  new FlatMapFunction<String, String>() {
    public Iterable<String> call(String s) {
      return Arrays.asList(s.split(" "));
    }
  }
);
{% endhighlight %}

The word count program starts by creating a `JavaSparkContext`, which accepts
the same parameters as its Scala counterpart.  `JavaSparkContext` supports the
same data loading methods as Scala's `SparkContext`. Here, `textFile` loads lines 
from text files stored in HDFS.

To split the lines into words, we use `flatMap` to split each line on whitespace.  `flatMap` is passed a `FlatMapFunction` that accepts a string and
returns a `java.lang.Iterable` of strings.

Here, the `FlatMapFunction` was created inline. Another option is to subclass `FlatMapFunction` and pass an instance to `flatMap`:

{% highlight java %}
class Split extends FlatMapFunction<String, String> {
  public Iterable<String> call(String s) {
    return Arrays.asList(s.split(" "));
  }
);
JavaRDD<String> words = lines.flatMap(new Split());
{% endhighlight %}

Java 8+ users can also write the `FlatMapFunction` in a more concise way using 
a lambda expression:

{% highlight java %}
JavaRDD<String> words = lines.flatMap(s -> Arrays.asList(s.split(" ")));
{% endhighlight %}

This lambda syntax can be applied to all anonymous classes in Java 8.

Continuing with the word count example, we map each word to a `(word, 1)` pair:

{% highlight java %}
import scala.Tuple2;
JavaPairRDD<String, Integer> ones = words.mapToPair(
  new PairFunction<String, String, Integer>() {
    public Tuple2<String, Integer> call(String s) {
      return new Tuple2(s, 1);
    }
  }
);
{% endhighlight %}

Note that `mapToPair` was passed a `PairFunction<String, String, Integer>` and
returned a `JavaPairRDD<String, Integer>`.

To finish the word count program, we'll use `reduceByKey` to count the
occurrences of each word:

{% highlight java %}
JavaPairRDD<String, Integer> counts = ones.reduceByKey(
  new Function2<Integer, Integer, Integer>() {
    public Integer call(Integer i1, Integer i2) {
      return i1 + i2;
    }
  }
);
{% endhighlight %}

Here, `reduceByKey` is passed a `Function2`, which implements a function with
two arguments.  The resulting `JavaPairRDD` contains `(word, count)` pairs.

In this example, we explicitly showed each intermediate RDD.  It is also
possible to chain the RDD transformations, so the word count example could also
be written as:

{% highlight java %}
JavaPairRDD<String, Integer> counts = lines.flatMapToPair(
    ...
  ).map(
    ...
  ).reduceByKey(
    ...
  );
{% endhighlight %}

There is no performance difference between these approaches, the choice is
just a matter of style.

# API Docs

[API documentation](api/java/index.html) for Spark in Java is available in Javadoc format.

# Where to Go from Here

Spark includes several sample programs using the Java API in
[`examples/src/main/java`](https://github.com/apache/spark/tree/master/examples/src/main/java/org/apache/spark/examples), including JavaWordCount.java. You can run them by passing the class name to the
`bin/run-example` script included in Spark; for example:

    $ ./bin/run-example org.apache.spark.examples.JavaWordCount local README.md

README.md is a file located in the top-level Spark directory.

Note: Each example program prints usage help when run
without any arguments.

For help on optimizing your programs, the Spark [Configuration](configuration.html) and
[Tuning](tuning.html) Guides provide information on best practices. They are especially important for
making sure that your data is stored in memory in an efficient format.
