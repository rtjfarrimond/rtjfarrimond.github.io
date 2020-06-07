---
layout: post
title:  "Type Safety: Make your compiler work for you!"
date:   2020-06-06 10:19:45 +0100
tags: ["scala", "type safety", "clean code"]
category: scala
---

I like precise, expressive types. They serve as compiler enforcible
documentation that make understanding the intention of code more intuitive.

Consider pessimistic locking as a motivating example. We need to acquire a lock
on some resource in order to carry out a computation, whilst avoiding
contention from other processes.

## Pessimistic Locking Example

We might have a set of methods to `get`, `lock`, and `unlock`
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

And our client code might look something like:

```scala
val svc = new StubResourceService
// svc: StubResourceService = repl.Session$App$StubResourceService@324a5538

def potentiallyDangerousComputation(resource: Resource): Resource = {
    // Some logic that requires the resource to be locked
    resource
}

val resource = svc.getResource(UUID.randomUUID())
// resource: Resource = Resource(3bafd273-d43b-458f-8c5f-239cf9b70c0b)

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
//   Resource(3bafd273-d43b-458f-8c5f-239cf9b70c0b)
// )
```

## Problems with this approach

As it is this implementation will work as expected, locking our resource before
carrying out the potentially dangerous computation, and releasing the lock when
it is done. However, there is nothing stopping us from ignoring the locking
constraints that we wish to impose by doing something like this:

```scala
val notLockedResource = svc.getResource(UUID.randomUUID())
// notLockedResource: Resource = Resource(f08303de-1950-49e2-86e6-ae101cb8550a)
val result2 = potentiallyDangerousComputation(notLockedResource)
// result2: Resource = Resource(f08303de-1950-49e2-86e6-ae101cb8550a)
```

This will compile and run no problem, but we now call
`potentiallyDangerousComputation` without first ensuring that our resource is
locked, which is exactly the situation we want to avoid.

Also, there is nothing to stop us from calling `unlock` with a resource that is
not locked.

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
//   Resource(3bafd273-d43b-458f-8c5f-239cf9b70c0b)
// )
```

But `lockedComputation` has multiple concerns, it has to handle the locking and
unlocking as well as performing the work it is actually intended to carry out.
This is clearly a violation of the [Single Responsibility Principal][srp].

What if we instead introduce a new type and update our service to use it:

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
// saferComputation(notLockedResource)
//                  ^^^^^^^^^^^^^^^^^
```

```scala
val betterSvc = new StubBetterResourceService
// betterSvc: StubBetterResourceService = repl.Session$App$StubBetterResourceService@28bdf256
val resource2 = betterSvc.getResource(UUID.randomUUID())
// resource2: Resource = Resource(b29f7368-3386-455b-9730-6dfc7e45baa1)
val processedResource = betterSvc.lockResource(resource2)
    .map(saferComputation)
    .flatMap(betterSvc.unlockResource)
// processedResource: Either[String, Resource] = Right(
//   Resource(b29f7368-3386-455b-9730-6dfc7e45baa1)
// )
```

## Caveat

It is of course still possible to get around the type check as simply as:

```scala
// Still dangerous!
saferComputation(LockedResource(notLockedResource))
// res3: LockedResource = LockedResource(
//   Resource(f08303de-1950-49e2-86e6-ae101cb8550a)
// )
```

We could put further measures in place to make it impossible to directly
instantiate a `LockedResource`, however we hope that it is clear from the
current implementation that this is not the correct way to create one in
application code (though it could be useful in test).


[srp]: https://en.wikipedia.org/wiki/Single-responsibility_principle
