---
layout: post
title:  "Cats Validated"
date:   2020-06-06 10:19:45 +0100
tags: ["scala", "cats", "validated", "validation", "typeclass", "typeclasses"]
category: scala
---

I am currently working my way through the excellent and free book [Scala With
Cats][swc], an introduction to functional programming in Scala using the
[cats][cats] library. In [section 6.4][validatedSection] the `Validated` data
type is introduced, which provides a convenient and powerful mechanism to
validate data (for example, from web forms) and accumulate errors to report
back to clients of your code.

In this post I will provide some code examples that are taken from my example
sbt project, which you can check out on [Github here][ghProject].

## Motivation

I won't dwell on the motivation for `Validated` for too long here, referring
the interested reader instead to the [relevant chapter in Scala With
Cats][validatedSection].  In a nutshell, whereas the [`Either` data
type][either] from the standard library provides 'fail fast' semantics (that is
to say, if there are multiple validation failures for a given object only the
first will be reported and subsequent validations will not be checked)
`Validated` fails slowly. This means that by using `Validated`, all validation
checks will be carried out in a way that accumulates errors, allowing all
failures to be reported at once to clients of our application.

## Example

Our example follows the hypothetical scenario of validating orders from a web
store. Let's take a look at our domain: at the highest level we have the
`Order` case class, which is composed of a `Customer`, a `List[Item]`, and a
`PaymentDetails`. In addition to these four case classes, we also have an
`Address` class, which `Customer` has an instance of. We will construct a
validator for each of these classes, and compose them together to validate our
top level `Order` class.

```scala
case class Customer(
    firstName: String,
    lastName: String,
    dateOfBirth: LocalDate,
    address: Address)

case class Address(
    houseNumberOrName: String,
    firstLine: String,
    secondLine: Option[String],
    city: String,
    county: Option[String],
    postcode: String)

import java.util.UUID

case class Item(
    uuid: UUID,
    name: String,
    weightKg: Int,
    cost: Int,
    ageRestriction: Option[Int])

case class PaymentDetails(
    accountHolder: String,
    accountNumber: String,
    sortCode: String)

case class Order(
    customer: Customer,
    items: List[Item],
    paymentDetails: PaymentDetails)

```


## A Validator Typeclass

I want to caveat this section by mentioning that it is not necessary to
implement a `Validator` [typeclass][catsTypeclass] to use `cats.Validated`, it
is simply the design pattern that I used in the example project. In fact, it
may not be an appropriate pattern for all use cases and there are a couple of
reasons that I am not entirely satisfied with it even in the example project,
which I will address towards the end of this post.

The typeclass we define below adds the extension method `.validate` to classes
for which there is a typeclass instance available, allowing us to call for
example `order.validate` for some instance of `Order` if a `Validator[Order]`
is in scope. If you are already familiar with the typeclass pattern, or
otherwise want to get to the actual validation code, feel free to skip ahead to
the next section armed with this knowledge.

### Trait

We will define our typeclass `Validator[V]` as a trait with a single method,
`validate` that takes a `V` instance  and returns a
`ValidatedNec[ValidationError, V]`. We will leave the description of what
exactly the `ValidatedNec` type is for the section below, but notice that is is
a type constructor of two arguments just like `Either`. This is because it
follows the same semantics, with the first type being returned in the failure
case, and the second in the success case.

We will also define a custom error type, `ValidationError`, that we will use in
the case of validation failure. It will take two string parameters, `id` - an
identifier for the field that was invalid, and `message` - a description of
the validation check that failed.

```scala
import cats.data.ValidatedNec

case class ValidationError(id: String, message: String)

trait Validator[V] {
  def validate(v: V): ValidatedNec[ValidationError, V]
}
```

### Companion Object

Next we define an Ops class in the companion object with a method that will
allow us to add the `.validate` extension method to any type `V` for which a
validator instance `Validator[V]` is in scope. We will also add some
validations here that will be common across multiple `Validator` instances:

```scala
import cats.syntax.validated._ // for invalidNec and valid

object Validator {
  implicit class ValidatorOps[V](val v: V) {
    def validate(
        implicit validator: Validator[V]
    ): ValidatedNec[ValidationError, V] =
        validator.validate(v)
  }

  def nonEmptyString(s: String)(fieldName: String):
      ValidatedNec[ValidationError, String] =
    if (s == "")
      ValidationError(fieldName, "Must not be the empty String").invalidNec
    else
      s.valid

  def regexValidator(
    string: String,
    regex: Regex,
    id: String,
    errorMessage: String
  ): ValidatedNec[ValidationError, String] =
    string match {
      case regex(_*) => string.valid
      case _         => ValidationError(id, errorMessage).invalidNec
    }
}
```

### Our first instance

We are now ready to create our first `Validator` instance,`Validator[Item]`. We
want to ensure that both the `weightKg` and `cost` fields are positive
integers.

```scala
import validator.Validator._ // for nonEmptyString

object ItemValidator extends Validator[Item] {
  override def validate(item: Item): ValidatedNec[ValidationError, Item] = {
    (
      item.uuid.valid,
      nonEmptyString(item.name)("item.name"),
      validatePositiveInteger(item.weightKg)("item.weightKg"),
      validatePositiveInteger(item.cost)("item.cost"),
      item.ageRestriction.valid
    ).mapN(Item)
  }

  private def validatePositiveInteger(i: Int)(fieldName: String):
      ValidatedNec[ValidationError, Int] =
    if (i > 0) i.valid
    else ValidationError(fieldName, "must be a positive integer").invalidNec
}
```

We can see this validator in action by testing against valid and invalid input:

```scala
val validItem: Item = Item(UUID.randomUUID(), "name", 1, 10, None)
validItem.validate

val invalidItem: Item = validItem.copy(weightKg = -42, cost = -42)
invalidItem.validate
```

## Validated

Let's unpack some of the entities that we introduced in the previous section:
* `ValidatedNec`
* `.valid`
* `.invalidNec`
* `mapN`

<!-- TODO: Reword below to fit better with this section, moved from above. -->
Similarly to `Either` this is a type
constructor that takes two arguments, the first being the type in the case that
the instance is invalid, and the second being the type in the case where it is
valid. In fact, `ValidatedNec` is itself [a type alias][catsValidatedNec] for
`Validated[NonEmptyChain[E], A]`


[cats]: https://typelevel.org/cats/
[catsTypeclass]: https://www.scalawithcats.com/dist/scala-with-cats.html#the-type-class
[catsValidatedNec]: https://typelevel.org/cats/api/cats/data/index.html#ValidatedNec[+E,+A]=cats.data.Validated[cats.data.package.NonEmptyChain[E],A]
[either]: https://www.scala-lang.org/api/2.9.3/scala/Either.html
[ghProject]: https://github.com/rtjfarrimond/order-validator
[swc]: https://underscore.io/books/scala-with-cats/
[validatedSection]: https://www.scalawithcats.com/dist/scala-with-cats.html#validated
