# Deadlocks
With the following code, a deadlock might happen if we run two-way
transfers between two accounts
```scala
class Account(private var amount: Int = 0) {
    def transfer(target: Account, n: Int) =
        this.synchronized {
            target.synchronized {
                this.amount -= n
                target.amount += n
            }
        }
}
```

We solve this problem by defining a hierachy between the different accounts.
Now, the transfers are always performed in the same way (from A to B)
```scala
val uid = getUniqueId()
private def lockAndTransfer(target: Account, n: Int) =
    this.synchronized {
        target.synchronized {
            this.amount -= n
            target.amount += n
        }
    }
def tansfer(target: Account, n: Int) =
    if (this.uid < target.uid) this.lockAndTransfer(target, n)
    else target.lockAndTransfer(this, -n) // the transfer always goes 
```

# Memory model
* Two threads writing to separate locations in memory do not
  synchronization
* A thread X that calls `join` on another thread Y is guaranteed to
  observe all the writes by thread Y after `join` returns.

# Computations using `parallel`
We can use the function `parallel` to make computations in parallel
```scala
val (v1, v2) = parallel(doSomething1(args1), doSomething2(args2))
```
The signature of `parallel` is the following. Arguments are by-name.
```scala
def parallel[A, B](taskA: => A, taskB: => B): (A, B) = { ... }
```

# More flexible construct with `task`
The function `task` launches a computation "in the background" and
the function `t.join` waits for the task to return a result
```scala
val t1 = task(e1)
val t2 = task(e2)
val v1 = t1.join
val v2 = t2.join
```

Here is a minimal `Task` interface
```scala
def task(c: => A): Task[A]

trait Task[A] {
    def join: A
}
```

We can define `parallel` using `task`
```scala
def parallel[A, B](cA: =>, cB: => B): (A, B) = {
    val tB: Task[B] = task { cb } // compute tB in the background
    val tA: A = cA // usual computation
    (tA, tB.join) // waits until tB finished before returning the tuple
}
```

# Using ScalaMeter
Add scalaMeter as a dependency
```scala
libraryDependencies += "com.storm-enroute" %% "scalameter-core" % "0.6"
```

Then, import ScalaMeter

```scala
import org.scalameter._

val time = measure {
    (o until 1000000).toArray
}
```

## ScalaMeter Warmers
```scala
val time = withWarmer(new Warmer.Default) measure {
    (0 until 1000000).toArray
}
```

We can configure ScalaMeter further
```scala
val time = config(
    Key.exec.minWarmupRuns -> 20,
    Key.exec.maxWarmupRuns -> 60,
    Key.verbose -> true
) withWarmer(new Warmer.Default) measure {
    (0 until 1000000).toArray
}
```
### Other measurers available
* `Measurer.Default` : plain running time
* `IgnoringGC` : running time without GC pauses
* `OutlierEliminiation` : removes statistical outliers
* `MemoryFootprint` : memory footprint of an object
* `GarbageCollectionCycles` : total number of GC pauses

```scala
withMeasurer(new Measurer.MemoryFootprint) measure {
    (0 until 1000000).toArray
}
```