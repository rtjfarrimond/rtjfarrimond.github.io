---
layout: post
title:  "Type Safety: Make your compiler work for you!"
date:   2020-06-06 10:19:45 +0100
tags: ["scala", "type safety", "clean code"]
category: scala
---

I like precise, expressive types. They serve as compiler enforcible
documentation that make understanding the intention of code more intuitive. In
this post we will look at how we can use types to enforce constraints, turning
potential bugs or runtime errors into issues that are caught at compile time.
We will also see that in doing so, we keep our code clean and easy for readers
to understand and maintain.

## Pessimistic Locking Example

Consider pessimistic locking as a motivating example. We need to acquire a lock
on some resource in order to carry out a computation, whilst avoiding
contention from other processes.

Suppose we have a set of methods to `get`, `lock`, and `unlock`
resources. An initial interface could look something like this:

```scala
import java.util.UUID

case class Resource(uuid: UUID)

// Stub implementations for illustration
class StubResourceService {
  def getResource(uuid: UUID): Resource = Resource(uuid)
  def lockResource(resource: Resource): Boolean = true
  def unlockResource(resource: Resource): Boolean = true
}
```

And the client code that uses the implementation above:

```scala
val svc = new StubResourceService
// svc: StubResourceService = repl.Session$App$StubResourceService@2160ea42

def potentiallyDangerousComputation(resource: Resource): Resource = {
    // Some logic that requires the resource to be locked
    resource
}

val resource = svc.getResource(UUID.randomUUID())
// resource: Resource = Resource(40341158-898f-4608-af5b-a5d7babbc932)

val errorOrResource =
    if (svc.lockResource(resource)) {
        potentiallyDangerousComputation(resource)
        if (!svc.unlockResource(resource)) {
            Left(s"Could not release lock on resource ${resource.uuid}")
        } else {
            Right(resource)
        }
    } else {
        Left("Could not acquire lock on resource")
    }
// errorOrResource: Either[String, Resource] = Right(
//   Resource(40341158-898f-4608-af5b-a5d7babbc932)
// )
```

## Problems with this approach

### 1) We cannot distinguish a locked resource from one that is not locked

As it is this implementation will work as expected, locking our resource before
carrying out the potentially dangerous computation, and releasing the lock when
it is done. However, there is nothing stopping us from ignoring the locking
constraints that we wish to impose by doing something like this:

```scala
val notLockedResource = svc.getResource(UUID.randomUUID())
// notLockedResource: Resource = Resource(669a5f88-f342-42ea-9172-9230addbe465)
val result2 = potentiallyDangerousComputation(notLockedResource)
// result2: Resource = Resource(669a5f88-f342-42ea-9172-9230addbe465)
```

This will compile and run no problem, but we now call
`potentiallyDangerousComputation` without first ensuring that our resource is
locked, which is exactly the situation we want to avoid.

### 2) Boolean results can be misleading

Additionally, there is nothing to stop us from calling `unlock` with a resource
that is not locked.

```scala
svc.unlockResource(notLockedResource)
// res0: Boolean = true
```

In the case of our stub code the `Boolean` result is misleading, since the
resource was not actually unlocked it was just never locked to begin with. In
real world application code we might even need to write code to explicitly
handle the case where the resource is not locked, for example if releasing the
lock involves deleting a lock file, which would not exist in this case:

```scala
import java.io.File

def unlockResourceByDeletingFile(resource: Resource): Boolean = {
    val file = new File(s"/tmp/${resource.uuid}")
    if (file.exists) {
        file.delete()
    } else {
        true // Misleading, the lock file never existed!
    }
}
```

## Ensuring resources are locked

To address the first problem, we might refactor to encapsulate the locking and
unlocking logic:

```scala
def lockedComputation(resource: Resource): Either[String, Resource] = {
    if (svc.lockResource(resource)) {
        // Some logic that requires the resource to be locked
        if (!svc.unlockResource(resource)) {
            Left(s"Could not release lock on resource ${resource.uuid}")
        } else {
            Right(resource)
        }
    } else {
        Left("Could not acquire lock on resource")
    }
}

lockedComputation(resource)
// res1: Either[String, Resource] = Right(
//   Resource(40341158-898f-4608-af5b-a5d7babbc932)
// )
```

But `lockedComputation` has multiple concerns, it has to handle the locking and
unlocking as well as performing the work it is actually intended to carry out.
This is clearly a violation of the [Single Responsibility Principle][srp].

Instead, we could introduce a new type and update our service to use it:

```scala
case class LockedResource(resource: Resource)

class StubBetterResourceService {
  def getResource(uuid: UUID): Resource = Resource(uuid)

  def lockResource(resource: Resource): Either[String, LockedResource] =
      Right(LockedResource(resource))

  def unlockResource(lockedResource: LockedResource): Either[String, Resource] =
      Right(lockedResource.resource)
}
```

This addresses the second problem, `unlockResource` now requires a
`LockedResource` instance, and can return a descriptive error message in the
case that the lock could not be released, since the return type is now
`Either[String, Resource]`.

We can now try again to address the first problem by refactoring
`potentiallyDangerousComputation` to require a `LockedResource` instance,
allowing us to separate concerns of locking and unlocking and simply do the
computation within this function.

```scala
def saferComputation(lockedResource: LockedResource): LockedResource = {
    // Some logic that requires us to guarnatee that the resource is locked
    lockedResource
}
```

The simple wrapper around `Resource` makes it impossible to call
`saferComputation` without having first locked the resource,
since we will get an error at compile time if we try to pass it an instance of
`Resource`.

```scala
saferComputation(notLockedResource)
// error: type mismatch;
//  found   : repl.Session.App.Resource
//  required: repl.Session.App.LockedResource
// saferComputation(LockedResource(notLockedResource))
//                                 ^^^^^^^^^^^^^^^^^
```

```scala
val betterSvc = new StubBetterResourceService
// betterSvc: StubBetterResourceService = repl.Session$App$StubBetterResourceService@7aa45d4f
val resource2 = betterSvc.getResource(UUID.randomUUID())
// resource2: Resource = Resource(946f6d6b-34bb-4af9-84f4-d371ad2916f8)
val processedResource = betterSvc.lockResource(resource2)
    .map(saferComputation)
    .flatMap(betterSvc.unlockResource)
// processedResource: Either[String, Resource] = Right(
//   Resource(946f6d6b-34bb-4af9-84f4-d371ad2916f8)
// )
```

## Caveat

It is of course still possible to get around the type check as simply as:

```scala
// Still dangerous!
saferComputation(LockedResource(notLockedResource))
// res3: LockedResource = LockedResource(
//   Resource(669a5f88-f342-42ea-9172-9230addbe465)
// )
```

If we wanted to be strict about preventing this we could put measures in place,
for example by modelling `LockedResource` as a `class` with a private default
constructor:

```scala
class SaferLockedResource private (resource: Resource) {
  override def toString: String = s"SaferLockedResource($resource)"
}

object SaferLockedResource {
  def acquireLock(
    resource: Resource
  ): Either[String, SaferLockedResource] = {
    // Logic to lock the resource
    Right(new SaferLockedResource(resource)) // Stub for example
  }
}

val acquiredLockedResource =
  SaferLockedResource.acquireLock(resource)
// acquiredLockedResource: Either[String, SaferLockedResource] = Right(
//   SaferLockedResource(Resource(40341158-898f-4608-af5b-a5d7babbc932))
// )
```

```scala
val directlyLockedResource = new SaferLockedResource(resource)
// error: constructor SaferLockedResource in class SaferLockedResource cannot be accessed in object App
// val directlyLockedResource = new SaferLockedResource(resource)
//                                  ^^^^^^^^^^^^^^^^^^^
```

However, we may instead decide that the intended use is clear enough and decide
against imposing this level of constraint, for example in order to simplify
testing.


## Conclusion

We have seen that by creating specific types, we can make our compiler work for
us to enforce certain preconditions before executing a particular piece of
code. In doing so we also make the preconditions clear to readers, making the
code easier to understand and maintain. Finally, we saw how types can allow us
to separate concerns more easily, and ensure that our code adheres to the
single responsibility principle.

Thank you for taking the time to read this post. If you have any thoughts or
feedback I'd love to hear it, so do feel free to reach out on Twitter or
LinkedIn :)


[srp]: https://en.wikipedia.org/wiki/Single-responsibility_principle
