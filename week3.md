# Data-parallelism
* **Task parallel programming** : A form of parallelization that distributes
  **execution processes** across computing nodes
* **Data-parallel programming** : A form of paralleization that distributes
  data cross computing nodes.

## Initializing array values
```scala
def initializeArray(xs: Array[Int])(v: Int): Unit = {
    for (i <- (0 until xs.length).par) { // .par makes the for loop parallel
        xs(i) = v
    }
}
```

## Mandelbrot set 
A complex number $c$ is part of the *Mandelbrot set* if 
the sequence $f_c(0), f_c(f_c(0)), f_c(f_c(f_c(0))), \dots$ with
$f_c(z) = z^2 + c$ remains bounded in absolute value

```scala
private def computePixel(xc: Doube, yc: Double, maxIterations: Int): Int = {
    var i = 0;
    var x, y = 0.0
    while (x * x + y * y < 4 && i < maxIterations) { // we check if |c^2| < 2
        val xt = (x * x - y * y) + xc // (a + ib)^2 + (a + ib) = a^2 + 2aib - b^2 + a + ib
        val yt = (2 * x * y) + yc
        x = xt
        y = yt
        i += 1
    }
    color(i) // associates a color with the "speed" at which it diverges/converges
}

def parRender(): Unit = {
    for (idx <- (0 until image.legnth).par) { // we compute each pixel in parallel
        val (xc, yc) = coordinatesFor(idx) // converts absolute array index to 2d coordinates
        image(idx) = computePixel(xc, yc, maxIterations)
    }
}
```

## Workload
A *workload* is a function that maps each input element to the amount of work
required to process it.

A uniform workload (constant) is easy to parallelize.

# Parallel collections
Most collection operations can become data-parallel. The `.par` method
converts a sequential collection to a parallel collection.
```scala
(1 until 1000).par
    .filter(n => n % 3 == 0)
    .count(n => n.toString == n.toString.reverse)
```

Some operations can not execute in data-parallel mode, for example `foldLeft`
It cannot be implemented in data-parallel because of its signature
```scala
def foldLeft[B](z: B)(f: (B, A) => B): B
```

However, we can define a function `fold` with a easier signature to deal with
```scala
def fold(z: A)(f: (A, A) => A): A
```

We can now define `max` or `sum` easily
```scala
def sum(xs: Array[Int]): int = {
    xs.par.fold(0)(_ + _)
}
```

We try to implement a 
```scala
def play(a: String, b: String): String = List(a, b).sorted match {
    case List("paper", "scissors") => "scissors"
    case List("paper", "rock") => "paper"
    case List("rock", "scissors") => "rock"
    case List(a, b) if a == b => a
    case List("", b) => b
}
```

## Preconditions of the `fold` operation
The following conditions must hold
```scala
f(a, f(b, c)) == f(f(a, b), c)
f(z, a) == f(a, z) == a
```
The element `z` is the neutral element. We say `z` and `f` form a *monoid*.
Commutativity does not matter.

## The `aggregate` operation
```scala
def aggregate[B](z: B)(f: (B, A) => B, g: (B, B) => B): B
```
We can count the number of vowels in a sequence of chars.
Here, `0` and `_ + _` form a monoid.
```scala
Array('E', 'P', 'F', 'L').par.aggregate(0)(
    (count, c) => if (isVowel(c)) count + 1 else count,
    _ + _
)
```