---
title: "Monadic Resource Management in Scala"
date: 2022-02-12T16:30:59+01:00
publishDate: 2022-02-12T16:30:59+01:00
draft: false
toc: false
images:
categories:
  - code
tags:
  - scala
  - jvm
  - functional-programming
  - learning
---

{{<figure src="/img/markus-spiske-C0koz3G1I4I-unsplash.jpg" caption="Photo by [Markus Spiske](https://unsplash.com/@markusspiske) in [Unsplash](https://unsplash.com/s/photos/pieces)">}}

In my company we have a considerably big Scala codebase. We use Scala for data pipelines, function apps, web services... Everything is based on a common set of libraries that standardizes access to resources, enforces some conventions, and of course provides several utilities. One of this is a tiny little method which implements automatic context/resource management for [`AutoCloseable`](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/AutoCloseable.html) objects.

```scala
def withCloseable[C <: AutoCloseable, R](closeable: => C)(f: C => R): R = {
  // simplified implementation
  val result = f(closeable)
  closeable.close()
  result
}
```

In this post I will explain how I —as a learning exercise— (over)refactored the previous method to make it work with one of Scala's most powerful idioms. A word of warning, this post may be challenging for Scala newbies, but I will include links to the underlying concepts along the text.

Back to the method, it is quite similar to [Python's `with` statement](https://www.python.org/dev/peps/pep-0343/): it creates a safe zone in which you can be sure that the resource you're trying to use is going be closed or released whenever the code block finishes.

```scala
withCloseable(openInputStream()) { myInputStream =>
  myInputStream.read()
}

// Here `myInputStream` is *closed*
```

By using this method we can forget about closing the resource. It is quite ubiquitous in our codebase, and it's really handy and simple when it's used alone. But, when it comes to composing `AutoCloseable` resources things get a bit dirtier.

```scala
withCloseable(OauthClient(conf)) { oauthClient =>
  withCloseable(ServiceClient(conf, oauthClient)) { serviceClient =>
    withCloseable(LoggingClient(conf)) { loggingClient =>
      // Here we have safe access to all three clients.
    }
  }
}
```

This works great too, but it is not quite good-looking, IMHO. It starts resembling [old JavaScript](http://callbackhell.com)... I was thinking on how I could make it look better when I realized Scala has a really nice construction that seemed quite a nice fit for this: **for-comprehensions**.

```scala
for {
  oauthClient   <- withCloseable(OauthClient(conf))
  serviceClient <- withCloseable(ServiceClient(conf, oauthClient))
  loggingClient <- withCloseable(LoggingClient(conf))
} yield {
  // Safe access to all three clients
}
```

This looks more reasonable to my eyes. No more unnecessary indentations and it has a more clear and pleasant look. The problem is we need to do some tweaks before make this work. Also, let's try to make the new API compatible with the previous one.

## How to for-comprehend

Scala's for-comprehensions are a powerful syntax sugar to compose [Monads](https://www.baeldung.com/scala/monads). I'm not going to waste keystrokes explaining [what](https://stackoverflow.com/questions/44965/what-is-a-monad) [Monads](https://stackoverflow.com/questions/2704652/monad-in-plain-english-for-the-oop-programmer-with-no-fp-background/2704795) [are](https://www.youtube.com/watch?v=FZAmPhjV11A) [in](https://towardsdatascience.com/monads-from-the-lens-of-imperative-programmer-af1ab8c8790c) [detail](https://medium.com/att-israel/its-time-you-learn-about-monads-4ebe687e3ec7), but for the purposes of this post, you just *basically* need to take into account that arrows `<-` in a for-comprehension are nothing else than calls to the `flatMap` method of a given object, and that the `flatMap` method is the cornerstone of Monads.

In *regular* syntax the previous snipped would look like this:

```scala
// Warning: this code does not compile yet!

withCloseable(OauthClient(conf)).flatMap { oauthClient =>
  withCloseable(ServiceClient(conf, oauthClient)).flatMap { serviceClient =>
    withCloseable(LoggingClient(conf)).map { loggingClient =>
      // Safe access to all three clients
    }
  }
}
```

As you can see, by using the regular style we end up with code similar to the original. We can see calling `withCloseable()` should return some kind of **underlying type** which implements the `flatMap` method. The `map` method is also needed, but it can be implemented in terms of `flatMap`. Let's see how we can achieve this.

## Introducing OpenedResource

Let's analyze `withCloseable`'s original signature:

```scala
def withCloseable[C <: AutoCloseable, R](closeable: => C)(f: C => R): R
```

It takes two argument lists. The first one receives the `AutoCloseable` instance [lazily](https://docs.scala-lang.org/tour/by-name-parameters.html). The second one is the actual *safe zone* where we have access to our `AutoCloseable` object, which can also be written as a code block (see previous code snippets). We've seen though that the `flatMap` method should be available *right after* the first argument list. So calling `withCloseable` with only the first argument list should return our underlying data structure: `OpenedResource`.

```scala
def withCloseable[C <: AutoCloseable](closeable: => C): OpenedResource[C] =
  new OpenedResource(closeable)

class OpenedResource[C <: AutoCloseable](closeable: => C) 
```

What about `withCloseable`'s second argument list? If we want to replicate it in `OpenedResource` we need its instances to be callable, i.e. we need the [`apply`](https://blog.matthewrathbone.com/2017/03/06/scala-object-apply-functions.html) method.

```scala
class OpenedResource[C <: AutoCloseable](closeable: => C) {

  def apply[R](f: C => R): R = {
    // simplified implementation
    val result = f(closeable)
    closeable.close()
    result
  }

}
```

Here we essentially moved the second argument list of the original `withCloseable` method into its own wrapper type. In other words, `OpenedResource` is the representation of the *safe zone* for a closeable instance.

## Composing OpenedResource

As said, for-comprehensions are allowed in Scala for types that implement `flatMap` and `map` methods. They are both higher-order functions (HOF) that allow composition and transformations of the underlying values of the monad/wrapper object. And both they always have the very same signatures on every type they are implemented on.

```scala
class OpenedResource[C](closeable: => C) {

  def apply[R](f: C => R): R = // omitted code

  def flatMap[B <: AutoCloseable](f: C => OpenedResource[B]): OpenedResource[B] = ???

  def map[B <: AutoCloseable](f: C => B): OpenedResource[B] = ???

}
```

In the context of `OpenedResource`, `flatMap` should provide *safe* access to the underlying `AutoCloseable` resource. Wait a second, this is precisely what the `apply` method does! We can use it to implement `flatMap`!

```scala
def flatMap[B <: AutoCloseable](f: C => OpenedResource[B]): OpenedResource[B] = 
  apply(f)
```

The only nuance here is that the `apply` call would is forced to return an `OpenedResource` instance, which is the return type of `f`, but that is exactly what we want. And now that we have `flatMap` we can use it to implement `map` by just making the compiler happy.

```scala
def map[B <: AutoCloseable](f: C => B): OpenedResource[B] =
  flatMap(c => new OpenedResource(f(c)))
```

## Testing time!

Let's write a unit test to verify it works.

```scala
"withCloseable allow for-comprehensions" in {
  val closeRecords = mutable.MutableList[Int]()

  class MyResource(val n: Int) extends AutoCloseable {
    override def close(): Unit = closeRecords += n
  }

  val result: OpenedResource[MyResource] = for {
    i1 <- withCloseable(new MyResource(1))
    i2 <- withCloseable(new MyResource(2))
  } yield {
    // safe zone with access to both closeable resources
    new MyResource(i1.n + i2.n)
  }

  closeRecords.toList should contain theSameElementsAs List(2, 1)
}
```

This test is very simple: we define an `AutoCloseable` child class which appends a value in a list whenever its `close` method is called. The expected result is that the latest call to `withCloseable` will release the resource first, and thus that is why the expected list is in the reversed order of calls to `withCloseable`.

This test works! But notice there is a quite important nuance: the `result` type is `OpenedResource[MyResource]`! This is inconvenient:
- **Our** `flatMap` method ensures `AutoCloseable` composition by enforcing its return type to be an extension of `AutoCloseable`. In other words, in the `yield` block we cannot return any other type than `AutoCloseable` instances.
- We need a way to access or unwrap `OpenedResource`'s underlying value.

Let's focus on the first point. This code

```scala
val result: OpenedResource[Int] = for {
  i1 <- withCloseable(new MyResource(1))
  i2 <- withCloseable(new MyResource(2))
} yield {
  i1.n + i2.n
}
```

won't compile since `Int` —the value of the type we are returning in the `yield` section— does not extend `AutoCloseable`.

One option could be wrapping our types in a fake `AutoCloseable` class with a `fake` close method, but that would make the whole thing extremely inconvenient.

Could we perhaps just remove type bounds in `OpenedResource`'s definition? That would make the test compile but the internals of `OpenedResource` won't, since the compiler is not aware that the value it's wrapping has a `close` method.

What if we had some way to tell the compiler to derive a type that provides the resource-releasing API —the `close` method— for `AutoCloseable` types and yet allow regular types to fit in so they can be the result of a for-comprehension?

## Enter type classes

Well, it turns out we have a way. [Type classes](https://www.baeldung.com/scala/type-classes) can isolate the resource-releasing logic from the types we pass to `OpenedResource` and thus make it generic for any type.

```scala
trait Closer[C] {
  def close(closeable: C): Unit
}
```

First thing to define on a type class is the type class interface. Our use case is quite simple. We need an interface to `close` resources.

Let's refactor our `OpenedResource` class.

```scala
class OpenedResource[C](closeable: => C)(implicit closer: Closer[C]) {

  def apply[R](f: C => R): R = {
    val result = f(closeable)
    closer.close(closeable)
    result
  }

  def flatMap[B: Closer](f: C => OpenedResource[B]): OpenedResource[B] = apply(f)

  def map[B: Closer](f: C => B): OpenedResource[B] = flatMap(c => new OpenedResource(f(c)))

}
```

Instead of enforcing `C` to be `AutoCloseable`, we take that logic out to a `Closer[C]` instance, and then we use that instance to free the resource. We also use [context bounds](https://docs.scala-lang.org/scala3/book/ca-context-bounds.html) in `flatMap` and `map` so that we make sure that the return type also has a `Closer` instance available.

But, where are those `Closer` (type class) instances? Well, we can provide them automagically using `implicits` or `given`/`using` in Scala 3.

```scala
object Closer {
  implicit def autoCloseableCloser[C <: AutoCloseable]: Closer[C] = new Closer[C] {
    override def close(closeable: C): Unit = closeable.close()
  }

  implicit def nonAutoCloseable[T]: Closer[T] = new Closer[T] {
    override def close(closeable: T): Unit = ()
  }
}
```

Here we declare two `Closer` instances. One for `AutoCloseable` types and another generic one for the rest of types. If we create an `OpenedResource` instance with an `AutoCloseable` argument, the compiler will inject `autoCloseableCloser`, whereas it will inject `nonAutoCloseable` for any other type. This last one implements a fake `close` method by just returning `Unit`; but it works for any type and we do not need to write it anywhere else!

If we get back to our test, now we can do the following:

```scala
"withCloseable allow for-comprehensions" in {
  val closeRecords = mutable.MutableList[Int]()

  class MyAutoCloseable(val n: Int) extends AutoCloseable {
    override def close(): Unit = closeRecords += n
  }

  val result: OpenedResource[Int] = for {
    i1 <- withCloseable(new MyAutoCloseable(1))
    i2 <- withCloseable(new MyAutoCloseable(2))
  } yield i1.n + i2.n

  closeRecords.toList should contain theSameElementsAs List(2, 1)
}
```

And this code compiles! Whenever `withCloseable` creates `OpenedResource` instances the compiler will inject `autoCloseableCloser` because the argument we are passing to `withCloseable` is an `AutoCloseable` instance!

Also, notice we are now allowed to get an `OpenedResource[Int]` instance as the result of the for-comprehension.

## Unwrapping the result

Well, there was a second nuance we did not addressed. How do we unwrap that `Int` in the test `result`? That's fairly trivial:

```scala
class OpenedResource[C](closeable: => C)(implicit closer: Closer[C]) {

  // code omitted

  def get: C = apply(identity)

}
```

Here we added the `get` method to obtain the wrapped value in `OpenedResource` and also make sure we release the resource in the case it's actually a releasable object. We just use the `identity` function on `apply` —remember, `apply` runs the function it receives and after that it `closes` the resource.

## (Proper) alternatives to resource management

This post was the story of how I over-engineered a solution for a first-world problem™ just for fun and for the sake of learning and understanding how to leverage Scala's idioms. The original API is more than fine: it just works and it's way simpler, and thus easier to maintain. That would be enough to discard my approach. Suffice to say, this did not make it into production :) Still, I enjoyed myself writing it, and it was a very nice learning exercise.

There are proper and nicer existing alternatives to resource management, like Scala 2.13's [`Using`](https://www.scala-lang.org/api/2.13.6/scala/util/Using$.html), or Cats Effect's [`Resource`](https://typelevel.org/cats-effect/docs/std/resource).

We engineers like to reinvent the wheel where most probably others made nicer wheels, but it is still a worthy exercise for ourselves, because we can learn a lot in the process.
