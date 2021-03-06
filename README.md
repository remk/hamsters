# Hamsters

A mini Scala utility library. Compatible with functional programming beginners. For the JVM and Scala.js.

<img src="https://img4.hostingpics.net/pics/987051hamsters.jpg" alt="hamster logo" width="150px"/>

Currently, Hamsters supports :

 * Validation
 * Monad transformers 
 * Enum typeclass
 * Default values for options (orEmpty)
 * HLists
 * Union types
 * Retry util

[![Travis](https://travis-ci.org/scala-hamsters/hamsters.svg?branch=master)](https://travis-ci.org/scala-hamsters/hamsters)

## Install as dependency

With SBT :

```scala
libraryDependencies ++= Seq(
  "io.github.scala-hamsters" %% "hamsters" % "1.5.2"
)
```

With Maven :

```xml
<dependency>
  <groupId>io.github.scala-hamsters</groupId>
  <artifactId>hamsters_${scala.version}</artifactId>
  <version>1.5.2</version>
</dependency>
```

For Scala.js :

```scala
libraryDependencies ++= Seq(
  "io.github.scala-hamsters" %%% "hamstersjs" % "1.5.2"
)
```

## Usage

### Validation

Statements can be succesful or fail. Validation helps you to retrieve values or failures for a sequence of statements.

```
  Note : Validation relies on standard Either types.
  Left is used for failures, Right for correct results.
  You can also use KO as an alias for Left and OK as an alias for Right.
```

```scala
import io.github.hamsters.Validation
import Validation._

val e1 = Right(1)
val e2 = Left("error 1")
val e3 = Left("error 2")
val e4 = Right("4")

Validation.result(e1,e2, e3) // List[String] : Left(List("error 1", "error 2"))
Validation.result(e1, e4) // Right((1, "4"))

Validation.failures(e1,e2, e3) // List[String] : List("error 1", "error 2")
Validation.failures(e1, e4) // Nil
```

To automatically catch exceptions into a Left or KO object, you can use `fromCatchable`.
By default it will give you an error message, but you can specify what to do in error cases :

```scala
def compute(x: Int):Int = 2/x

fromCatchable(compute(1)) //Right(2)
fromCatchable(compute(0)) //Left("/ by zero")

fromCatchable(compute(0), (t: Throwable) => t.getClass.getSimpleName) //Left("ArithmeticException")
```

```
Note : You can also use Right/Left (or OK/KO) in a monadic way (for example in a for-comprehension) if you want to stop processing at the first encountered error, even with Scala 2.11.
To make it work with Scala 2.11, just import Validation.OKBiasedEither.
```

###  Monad transformers

Example : combine Future and Option types then make it work in a for comprehension.
More information on why it's useful [here](http://loicdescotte.github.io/posts/scala-compose-option-future/).

#### FutureOption

```scala
def foa: Future[Option[String]] = Future(Some("a"))
def fob(a: String): Future[Option[String]] = Future(Some(a+"b"))

val composedAB: Future[Option[String]] = for {
  a <- FutureOption(foa)
  ab <- FutureOption(fob(a))
} yield ab
```

#### FutureEither

```scala
import io.github.hamsters.Validation._
import io.github.hamsters.{FutureEither, FutureOption}
import io.github.hamsters.MonadTransformers._

def fea: Future[Either[String, Int]] = Future(Right(1))
def feb(a: Int): Future[Either[String, Int]] = Future(Right(a+2))

val composedAB: Future[Either[String, Int]] = for {
  a <- FutureEither(fea)
  ab <- FutureEither(feb(a))
} yield ab
```

Note : For Future[Try] composition, see `Future.fromTry` to get a single future success or failure.

### Enums

This typeclass allows to use `parse`, `name` and `list` methods on Algebraic Data Types. It can be very useful if you need to serialize and deserialize your types (in Json, in a database...).
You can [read here](https://underscore.io/blog/posts/2014/09/03/enumerations.html) why it's better to use sealed case objects than Scala Enumerations.

```scala
sealed trait Season
case object Winter extends Season
case object Spring extends Season
case object Summer extends Season
case object Fall extends Season

implicit val seasonEnumerable = new Enumerable[Season] {
  override def list: List[Season] = List(Winter, Spring, Summer, Fall)
}

Enumeration.name(Winter) // "winter"
Enumeration.parse[Season]("winter") // Some(Winter)
Enumeration.list[Season] // List(Winter, Spring, Summer, Fall)
```

It is also possible to use custom namings :

```scala

implicit val seasonEnumerable = new Enumerable[Season] {
  override def list = List(Winter, Spring, Summer, Fall)

  override def name(s: Season) = {
    s match {
      case Winter => "WINTER_SEASON"
      case other => super.name(other)
    }
  }
}

Enumeration.name(Winter) // "WINTER_SEASON"
Enumeration.parse[Season]("WINTER_SEASON") // Some(Winter)
```

## Default values for options (orEmpty)

Default values for options avoid repeating the same `getOrElse` code again and again : 

```scala
val noneString: Option[String] = None
noneString.orEmpty  // ""

val noneInt: Option[Int] = None
noneInt.orEmpty // 0 

val noneList: Option[List[String]] = None
noneList.orEmpty  // List[String]()
``` 

To support other types, you just have to declare a new implicit value : 
 
 ```scala
implicit def myCaseClassDefaultValue= new DefaultValue[MyCaseClass] { ... }
```

### HList

HLists can contain heterogeneous data types but are strongly typed. It's like tuples on steroids!
When you're manipulating data using tuples, it's common to add or subtrack some elements, but you have to make each element explicit to build a new tuple. HList simplifies this kind of task.

 * `::` is used to append elements at the beginning of an HList, and to construct new HLists
 * `+` is used add element at the end of a HList
 * `++` is used to concatenate 2 Hlists
 * HNil is the empty HList
 * other operations : filter, map, foldLeft, foreach

```scala
import io.github.hamsters.{HList, HCons, HNil}
import HList._

val hlist1: Double :: String :: HNil = 2.0 :: "hi" :: HNil
val hlist2 = 1 :: HNil

val sum = hlist1 + 1 // 2.0 :: "hi" :: 1 :: HNil
val sum2 = hlist1 ++ hlist2 // 2.0 :: "hi" :: 1 :: HNil

sum2.tail // hi :: (1 :: HNil)
sum2.head // 2.0 (Double)
sum2.tail.head // "hi" (String)

// Retrieve element by index and type
hlist1.get[String](1) // Some("hi")
// Or use apply to avoid Option
hlist1[String](1) // "hi"

(2.0 :: "hi" :: HNil).foldLeft("")(_+_) // "2.0hi"

(2.0 :: "hi" :: HNil).map(_.toString) // "2.0" :: "hi" :: HNil

(2.0 :: "hi" :: HNil).filter {
  case s: String if s.startsWith("h") => true
  case _ => false
} //"hi" :: HNil

```

### Union types

You can define functions or methods that are able to return several types, depending on the context.

```scala
import io.github.hamsters.{Union3, Union3Type}

//json element can contain a String, a Int or a Double
val jsonUnion = new Union3Type[String, Int, Double]
import jsonUnion._

def jsonElement(x: Int): Union3[String, Int, Double] = {
  if(x == 0) "0"
  else if (x % 2 == 0) 1
  else 2.0
}
```

Then you can use pattern matching, or ask for a specifc type and retrieve an option :

```scala
jsonElement(0).get[String] // Some("0")
jsonElement(1).getOrElse("not found") // get String value or "not found" if get[String] is undefined
```


### Retry

Retry allows to run a code block several times if it fails, with a customizable maximum number of tries. It accepts a function to log the error messages : 

```scala
val logErrorsFunction = (errorMessage: String) => println(errorMessage)

Retry(maxTries = 3, logErrorsFunction) {
  1+1
} // util.Success(2)
```


### Extensions

 * [Hlist extensions](https://github.com/scala-hamsters/hamsters-extensions/blob/master/hamsters-hlists-extensions/README.md)
 * [Twitter Utils extensions](https://github.com/scala-hamsters/hamsters-extensions/blob/master/hamsters-twitter-util/README.md)


See [hamsters-extensions](https://github.com/scala-hamsters/hamsters-extensions) for more information.

## Scaladoc

You can find the API documentation [here](http://scala-hamsters.github.io/hamsters/api).

## Special thanks

To [Laurencebeillaux](https://github.com/laurencebeillaux) who created the Hamsters logo.
