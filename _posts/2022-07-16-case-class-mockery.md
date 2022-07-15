---
layout: post
title: Companion Objects Make a Mockery of Testing Case Classes
categories: scala, testing
date: 2022-07-15 22:33 +0100
---

Checking scala outputs by comparing them to case class instances in the 
test can be tempting, but it's too easy to accidentally turn them into
mocks by introducing companion objects.


What do I mean by "mocks"?  I mean that they fall on the "mock" side
of the ["mock vs stub"](https://martinfowler.com/articles/mocksArentStubs.html)
division.

Obviously, they are not really mocks, the companion objects are actual
production code, but once you introduce them,
you are no longer performing state verification, but behaviour verification.
What's more, this transition happens not in your test suite, but in your 
production code, and it can happen seamlessly and completely by accident.


Let's say we have this `Fruit` trait, and some case classes that inherit from it
```scala
trait Fruit {
  val ripeness:Int
  def isRipe:Boolean = ripeness >= 2
}

case class Apple(ripeness:Int) extends Fruit
case class Banana(ripeness:Int) extends Fruit
```

We have some function that populates a list of various Fruits, and we 
have some tests to ensure it does what we want:

```scala
getFruitBowl(2,1) shouldBe List(Banana(1), Banana(1), Apple(1))))
```

This test passes, calling `getFruitBowl` with the arguments `2` and `1`
returns a list containing two `Banana`s and one `Apple`, each with 
a ripeness of `1`.

That's what it looks like.  But what it is actually saying is that it 
returns three things, the first two are the same as calling `Banana.apply(1)`
and the third, the same as calling `Apple.apply(1)`.

As long as these `Fruits` remain case classes on their own, 
with no companion object shenanigans, then there is no difference 
between the two.

However, as soon as you add a companion object, anything can happen, 
and your tests will only detect it if the footprint changes.

Sometimes, bananas ripen so quickly that they are a different colour when 
you get them home, to when you put them in your shopping basket.  To 
reflect this, you introduce a companion object.

```scala
object Banana{
  def apply(ripeness:Int) = new Banana(ripeness + 1)
}
```

So now, when you call `Banana(1)`, you get a `Banana` with a ripeness of `2`.
What happens in your test suite?  Nothing changes, nothing fails.  To a 
casual observer, the test still says "Banana with ripeness 1", but the real
meaning has shifted to "The result of calling apply on the Banana object",
which is `Banana` with a ripeness of 2.

The example above is highly simplified, and one would probably not even
consider writing tests like that, instead explicitly verifying state.
Where tests like this are likely, are in situations where an input is 
parsed into some kind of tree, e.g.

```scala
parseGreeting("hello, big bad wolf") should be GreetingPhrase(
Greeting("hello"), Addressee(Modifiers(Adj("big"), Adj("bad")), N("wolf"))
)
```

Explicit state verification in that situation can be a little more cumbersome, but 
could save you from some nasty surprises in the future.