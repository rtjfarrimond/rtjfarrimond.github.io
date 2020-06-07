I like precise, expressive types. They serve as compiler enforcible
documentation that make understanding the intention of code more intuitive.

Consider pessimistic locking as a motivating example. We need to acquire a lock
on some resource in order to carry out a computation, whilst avoiding
contention from other processes.

## Pessimistic Locking Example

We might have a set of methods to `get`, `lock`, and `unlock`
resources. An initial interface could look something like this:

```scala mdoc
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

```scala mdoc
val svc = new StubResourceService

def potentiallyDangerousComputation(resource: Resource): Resource = {
    // Some logic that requires the resource to be locked
    resource
}

val resource = svc.getResource(UUID.randomUUID())

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
```

## Problems with this approach

As it is this implementation will work as expected, locking our resource before
carrying out the potentially dangerous computation, and releasing the lock when
it is done. However, there is nothing stopping us from ignoring the locking
constraints that we wish to impose by doing something like this:

```scala mdoc
val notLockedResource = svc.getResource(UUID.randomUUID())
val result2 = potentiallyDangerousComputation(notLockedResource)
```

This will compile and run no problem, but we now call
`potentiallyDangerousComputation` without first ensuring that our resource is
locked, which is exactly the situation we want to avoid.

Also, there is nothing to stop us from calling `unlock` with a resource that is
not locked.

```scala mdoc
svc.unlockResource(notLockedResource)
```

In the case of our stub code the `Boolean` result is misleading, since the
resource was not actually unlocked it was just never locked to begin with. In
real world application code we might even need to write code to explicitly
handle the case where the resource is not locked, for example if releasing the
lock involves deleting a lock file, which would not exist in this case:

```scala mdoc
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

```scala mdoc
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
```

But `lockedComputation` has multiple concerns, it has to handle the locking and
unlocking as well as performing the work it is actually intended to carry out.
This is clearly a violation of the [Single Responsibility Principal][srp].

What if we instead introduce a new type and update our service to use it:

```scala mdoc
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

```scala mdoc
def saferComputation(lockedResource: LockedResource): LockedResource = {
    // Some logic that requires us to guarnatee that the resource is locked
    lockedResource
}
```

The simple wrapper around `Resource` makes it impossible to call
`saferComputation` without having first locked the resource,
since we will get an error at compile time if we try to pass it an instance of
`Resource`.

```scala mdoc:fail
saferComputation(notLockedResource)
```

```scala mdoc
val betterSvc = new StubBetterResourceService
val resource2 = betterSvc.getResource(UUID.randomUUID())
val processedResource = betterSvc.lockResource(resource2)
    .map(saferComputation)
    .flatMap(betterSvc.unlockResource)
```

## Caveat

It is of course still possible to get around the type check as simply as:

```scala mdoc
// Still dangerous!
saferComputation(LockedResource(notLockedResource))
```

We could put further measures in place to make it impossible to directly
instantiate a `LockedResource`, however we hope that it is clear from the
current implementation that this is not the correct way to create one in
application code (though it could be useful in test).


[srp]: https://en.wikipedia.org/wiki/Single-responsibility_principle
