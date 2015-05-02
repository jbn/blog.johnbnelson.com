Title: Beware the Scala Value Class
Date: 2015-10-29 16:06
Category: Programming
Tags: scala
Slug: beware-the-scala-value-class
Summary: When refactoring from a primative type to a value class, the type system is less rigid then you would expect.

Imagine you are building an online game in Scala.

Your game has an economic aspect.

- Users accumulate `Credits`.
- Users can send other users these `Credits`.
- The result of one user sending `Credits` to another is logged in a `Transaction`.
- Rather than storing a reference to the ephemeral `User` object, you store
  the sender and recipient users' persistent `UserId`s.

The following implements this specification using [Scala's value classes](http://docs.scala-lang.org/overviews/core/value-classes.html).

    ::scala
    case class Credits(val id: Int) extends AnyVal
    case class UserId(val id: Int) extends AnyVal
    case class TransactionId(val id: Int) extends AnyVal
    case class Transaction(
      id: TransactionId,
      from: UserId,
      to: UserId,
      amount: Credits
    )
  
### Why use a value class?

You know the values of your identifiers and `Credits` will always
fit into an `Int`, so it seems like a bad idea -- performance wise --
to wrap the `Int` in a class. In doing so, you would incur the storage
cost of the underlying `Int`, plus the class that encapsulates it; and,
you would also incur a cost for the indirection when accessing the
encapsulated `Int`.

By using a value class, you get your reified ontology,
but you avoid allocating runtime objects to wrap the `Ints`. This is great
because you like type-safety and performant code -- that's what drew you to
Scala! But, after implementing some code, you realize your assumptions about
value classes may not have been valid. So, you test them. 

    ::scala
    // UserIds with identical underlying values should be equal().
    val johnny = UserId(1)
    assert(johnny == johnny)

    // UserIds with different underlying values should not be equal().
    val kluwe = UserId(5)
    assert(johnny != kluwe)


Both these assertions held.

When creating a `Transaction`, the type-safety of your ontology should
be enforced by the compiler. That is, you shouldn't be able to pass
a `Credit` to a method in a position that expects a `UserId`, even though
they are both value classes with the same underlying type.

This assumption holds.

    ::scala
    val transactionId = TransactionId(1)
    val credits = Credits(100)

    // The following compiles fine.
    val transaction = Transaction(transactionId, johnny, kluwe, credits)

    // The following would result in a type mismatch at compile time.
    val transaction = Transaction(johnny, transactionId, kluwe, credits)


Everything seems to be going fine. But, you know there is a bug in your
game. You eventually track it down to a section of code you wrote before
using value classes, when you were just passing around `Int`s.

So, you write some test code to see what happens if you compare a `UserId`
to an Int, the underlying type of the `UserId`.

    ::scala
    assert(johnny != 5)


The previous assertion held. At first, that seems correct. `johnny` has a
user id of 1, not 5. But, wait, that's not right. `johnny` has a `UserId` with
an underlying value of 5. **What happened to the type safety?!** 

Digging in, you issue another assertion test.

    ::scala
    assert(johnny == 1)


The previous assertion did not hold!

Summarizing what you just observed:

- User(1) == 1 is false
- User(1) != 5 is true

###What's happening? 

Well, `scalac` actually told you there was a problem, but you weren't paying
  attention. It issued two warnings:

> Warning:(XX, XX) comparing values of types UserId and Int using `!=' will always yield true

and
  
> Warning:(XX, XX) comparing values of types UserId and Int using `==' will always yield false


Ahah! When you refactored your code, substituting value classes
where there had previously been `Int`s, you missed some of the `Int`s. You
expected  Scala's scrupulous type system to diligently balk if you failed to
update some part of your code during this refactoring. It didn't. In its
defense, it did warn you; but, when it comes to Scala and types,
you are used to arresting errors, not timid warnings.

Stepping back, the observed behavior makes sense. Scala sees that you need a call to `equals `, so it instantiates the value class as a `UserId`. (The [SIP warns you](http://docs.scala-lang.org/overviews/core/value-classes.html) that this will happen in the situation.) You know that the first
rule of writing an `equals` method is to check that `that` is the same
`instanceOf` `this`. If it's not, you return `false`. When you compare
`User(1)` to `1`, `this` is not the same type as `that`. While this may be
the correct behavior with respect to what is expected of the `equals` method,
it is **Surprising**, and [programmer's don't like to be surprised](http://en.wikipedia.org/wiki/Principle_of_least_astonishment).

I'd like it if the compiler promoted this type of inferred
always-true-or-always-false-on-comparisons warning given a value class to an
error. Or, if that would unnecessarily complicate the compiler, I'd like it
if I could have a flag to promote a warning of this type to an error, in
general. I looked around but was unable to find that flag. I'm hoping it 
exists, and I'm just ignorant -- and someone will help me correct my 
deficiency.

