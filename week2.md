# Parallel sort
```scala
def parMergeSort(xs: Array[Int], maxDepth: Int): Unit = {
    val ys = new Array[Int](xs.length)

    def sort(from: Int, until: Int, depth: Int): Unit = {
        if (depth == maxDepth) {
            // if depth == maxDepth, then flip = true, then xs was the destination
            // array at the previous split -> we can sort xs directly
            quickSort(xs, from, until - from)
        } else {
            val mid = (from + until) / 2
            parallel(sort(mid, until, depth + 1), sort(from, mid, depth + 1))

            val flip = (maxDepth - depth) % 2 == 0
            val src = if (flip) ys else xs
            val dst = if (flip) xs else ys
            merge(src, dst, from, mid, until)
        }
    }
    def merge(src: Array[Int], dst: Array[Int], from: Int, mid: Int, until: Int): Unit =
        // ...

    def copy(src: Array[Int, target: Array[Int], from: Int, until: Int, depth: Int): Unit = {
        if (depth == maxDepth) {
            Array.copy(src, from, target, from, until - from)
        } else {
            val mid = (from + until) / 2
            val right = parallel(
                copy(src, target, mid, until, depth + 1),
                copy(src, target, from, mid, depth + 1)
            )
        }
    }
    // if maxDepth - 0 is even, then the result is
    // currently stored in ys, we need to copy
    if (maxDepth % 2 == 0) copy(ys, xs, 0, xs.length, 0)

    sort(0, xs.length, 0)
}
```

# Data operations in parallel
## Map
Properties to keep in mind
* `list.map(x => x) == list`
* `list.map(f.compose(g)) == list.map(g).map(f)`

Recall that `(f.compose(g))(x) = f(g(x))`

We can define our map operation in parallel on an array
```scala
def mapASegPar[A, B](inpu: Array[A], left: Int, right: Int, f: A => B, out: Array[B]): Unit = {
    if (right - left < threshold)
        mapAsegSeq(inp, left, right, f, out)
    else {
        val mid = left + (right - left) / 2
        parallel(mapASegPar(inp, left, mid, f, out),
                 mapASegPar(inp, mid, right, f, out))
    }
}
```

We can do it on a immutable tree as well
```scala
sealed abstract class Tree[A] {
    val size: Int
}
case class Leaf[A](a: Array[A]) extends Tree[A] {
    override val size = a.size
}
case class Node[A](l: Tree[A], r: Tree[A]) extends Tree[A] {
    override val size = l.size + r.size
}

def mapTreePar[A:Manifest, B:Manifest](T: Tree[A], f: A => B) : Tree[B] =
    t match {
        case Leaf(a) => {
            val len = a.legnth
            val b = new Array[B](len)

            var i = 0
            while (i < len) {
                b(i) = f(a(i))
                i = i + 1
            }
            Leaf(b)
        }
        case Node(l, r) => {
            val (lb, rb) = parallel(mapTreePar(l, f), mapTreePar(r, f))
            Node(lb, rb)
        }
    }
```

## Fold operations
```scala
List(1,3,8).foldLeft(100)((s, x) => s - x) == ((100 - 1) - 3) - 8 == 88
List(1,3,8).foldRight(100)((s, x) => s - x) == 1 - (3 - (8 - 100)) == -94
List(1,3,8).foldLeft((s, x) => s - x) == (1 - 3) - 8 == -10
List(1,3,8).foldLeft((s, x) => s - x) == 1 - (3 - 8) == 6
```
### Associative operation
`f` is associative iff `f(x, f(y, z)) = f(f(x, y), z)`

We can also write `x * (y * z) = (x * y) * z`

If the operation is associative, the parentheses do not matter. We can
place them where we want, it does not change the result.

We can make tree reduce parallel
```scala
sealed abstract class Tree[A]
case class Leaf[A](value: A) extends Tree[A]
case class Node[A](left: Tree[A, right: Tree[A]]) extends Tree[A]

def reduce[A](t: Tree[A], f: (A, A) => A): A = t match {
    case Leaf(v) => v
    case Node(l, r) => {
        val (lV, rV) = parallel(reduce[A](l, f), reduce[A](r, f))
        f(lV, rV)
    }
}
```

Suppose we have a function `toList` and a function `map` (both for trees)
```scala
def toList[A](t: Tree[A]): List[A] = t match {
    case Leaf(v) => List(v)
    case Node(l, r) => toList[A](l) ++ toList[A](r)
}
```

We can express `toList` using `map` and `reduce` :
`toList(t) == reduce(map(t, List(_)), _ ++ _)`

### Consequence of associativity stated as tree reduction
If `f` is associative, `t1` and `t2` are tree and `toList(t1)==toList(t2)`,
then `reduce(t1, f) == reduce(t2, f)`

## Prallel array reduce
```scala
def reduceSeg[A](inp: Array[A], left: Int, right: Int, f: (A, A) => A): A = {
    if (right - left < threshold) {
        var res = inp(left)
        var i = left + 1
        while (i < right) {
            res = f(res, inp(i))
            i = i + 1
        }
        res
    } else {
        val mid = left + (right - left) / 2
        val (a1, a2) = parallel(reduceSeg(inp, left, mid, f),
                                reduceSeg(inp, mid, right, f))
        f(a1, a2)
    }
}
def reduce[A](inp: Array[A], f: (A, A) => A): A =
    reduceSeg(inp, 0, inp.legnth, f)
```

# Other consequences of associativity
Associativity is not the same as commutativity

## Operations that are both associative and commutative
* Addition and multiplication of integers
* Addition and multiplication modulo a positive integer
* Union, intersection and [symmetric difference](https://en.wikipedia.org/wiki/Symmetric_difference)
* Union of bags (multisets) that preserves duplicate elements
* Boolean operations `&&`, `||`, `xor`
* Addition and multiplication of polynomials
* Addition of vectors
* Addition of matrices of fixed dimension

### Array norm example
$$\sum_{i = s}^{t-1} \lfloor \lvert p \rvert ^p \rfloor $$

We can express this sum as `reduce(map(a, power(abs(_), p)), _ + _)`

## Operations that are associative but not commutative
* Concatenation of lists or strings
* Matrix multiplication
* Composition of relations $r \odot s = \{(a, c) | \exists b  s.t. (a,b) \in r \wedge (b, c) \in s \}$
* Composition of functions `f(g(x))`

There are functions that are commutative but not associative, such as $f(x, y) = x^2 + y^2$

In general, if $f(x, y)$ is commutative and $h_1(z)$, $h_2(z)$ are arbitrary
functions, then any function defined by
$$ g(x, y) = h_2(f(h_1(x), h_1(y))) $$
is commutative, but it often loses associativity even if `f` was associative

**Warning** : floating point addition and multiplication is commutative but not associative

## Making an operation commutative is easy
Suppose we have a binary operation `g` and a strict total ordering
`less` (a way to compare two values)

Then this operation is commutative
```scala
def f(x: A, y: A) = if (less(y, x)) g(y, x) else g(x, y)
```

This implies that `f(x, y) == f(y, x))` because the operation
is always done in the same order

We do not have such tricks to get associative functions.

## Associative operations on tuples
Suppose `f1: (A1, A1) => A1` and `f2: (A2, A2) => A2` are associative
Then `f: ((A1, A2), (A1, A2)) => (A1, A2)` defined by
```scala
f((x1, x2), (y1, y2)) = (f1(x1, y1), f2(x2, y2)) 
```
is also associative

### Example : average
```scala
val sum = reduce(collection, _ + _)
val length = reduce(map(collection, (x: Int) => 1), _ + _)
sum / length
```
We can also write this function using a single reduce
```scala
f((sum1, len1), (sum2, len2)) = (sum1 + sum2, len1 + len2)
val (sum, length) = reduce(map(collection, (x: Int) => (x, 1)), f)
sum / length
```

## Associativity through symmetry and commutativity
Although commutativity of f alone does not imply associtivity, it implies
it if we have an additional property

We define `E(x, y, z) = f(f(x, y), z)` and we have `E(x, y, z) = E(y, z, x)`, which means
that `f(f(x, y), z) = f(f(y, z), x)`

Then, if `f` is commutative and arguments of `E` can rotate then `f`
is also associative

*Proof* :
```
f(f(x, y), z) = f(f(y, z), x) = f(x, f(y, z))
```

### Example : addition of modular fractions (video 2.5)
We have `plus((x1, y1), (x2, y2)) = (x1*y2 + x2*y1, y1*y2)`.
Plus is commutative, hence `E((x1, y2), (x2, y2), (x3, y3))` rotates.
Hence, `plus` is associative.

## A family of associative operations on sets
We define a binary operation on sets $A, B$ by $f(A, B) = (A \cup B)^{*}$
where $*$ is any operator on sets (closure) with these properties:
* $A \subseteq A^{*}$ (expansion)
* if $A \subseteq B$ then $A^* \subseteq B^*$ (monotonicity)
* $(A^*)^* = A^*$ (idempotence) (applying the operator twice is the same as once)

We see immediately that $f$ is commutative (because the $*$ is on the union)

We want to prove that every such $f$ is commutative

We have that
$$f(f(A, B), C) = ((A \cup B)^* \cup C)^*$$
We would like to show that 
$$f(f(A, B), C) = ((A \cup B)^* \cup C)^* = (A \cup B \cup C)^*$$
because it would mean that the arguments would rotate and it would imply that
f is associative (by theorem of associativity through symmetry and commutativity)

We prove $((A \cup B)^* \cup C)^* = (A \cup B \cup C)^*$
Since $A \cup B \subseteq A \cup B \cup C$, by monotonicity:
$$(A \cup B)^* \subseteq (A \cup B \cup C)^*$$

Similarly,
$$C \subseteq A \cup B \cup C \subseteq (A \cup B \cup C)^*$$

By combining these two results, we get
$$(A \cup B)^* \cup C \subseteq (A \cup B \cup C)^*$$

Then by monotonicity (and using the previous result) and by idempotence
$$((A \cup B)^* \cup C)^* \subseteq ((A \cup B \cup C)^*)^* = (A \cup B \cup C)^*$$

We have shown that $f$ is an associative function.

# Parallel scan
## Parallel scan using `mapSeg` and `reduceSeg1`
```scala
def scanLeft[A](inp: Array[A], a0: A, f: (A, A) => A, out: Array[A]) = {
    val fi = {
        // reduces elements between 0 and i (non-inclusive)
        (i: Int, v:A) => reduceSeg1(inp, 0, i, a0, f)
    }

    // for each element, we reduce from o to its index --> stupid
    mapSeg(inp, 0, inp.length, fi, out)
    val last = inp.length - 1
    out(last + 1) = f(out(last), inp(last))
}
```

We use trees to improve our implementation
```scala
sealed abstract class Tree[A]
case class Leaf[A](a: A) extends Tree[A]
case class Node[A](l: Tree[A], r: Tree[A]) extends Tree[A]

sealed abstract class TreeRes[A] {
    val res: A
}
case class LeafRes[A](override val res: A) extends TreeRes[A]
case class NodeRes[A](l: TreeRes[A], override val res: A, r: TreeRes[A]) extends TreeRes[A]
```

Our implementation is the following
```scala
def upsweep[A](t: Tree[A], f: (A, A) => A): TreeRes[A] = t match {
    case Leaf(v) => LeafRes(v)
    case Node(l, r) => {
        val (tL, tR) = paralle(upsweep(l, f), upsweep(r, f))
        NodeRes(tL, f(tL.res, tR.res), tR)
    }
}

def downsweep[A](t: TreeRes[A], a0: A, f: (A, A) => A): Tree[A] = t match {
    case LeafRes(a) => Leaf(f(a0, a))
    case NodeRes(l, _, r) => {
        val (tL, tR) = parallel(downsweep[A](l, a0, f),
                                downsweep[A](r, f(ao, l.res), f))
        Node(tL, tR)
    }
}

// adds element x in the leftmost position of t
def prepend[A](x: A, t: Tree[A]): Tree[A] = t match {
    case Leaf(v) => Node(Leaf(x), Leaf(v))
    case Node(l, r) => Node(prepend(x, l), r)
}

def scanLeft[A](t: Tree[A], a0: A, f: (A, A) => A): Tree[A] = {
    val tRes = upsweep(t, f)
    val scan1 = downsweep(tRes, a0, f)
    prepend(a0, scan1)
}
```

We want a more efficient implementation
```scala
sealed abstract class TreeResA[A] {val res: A}
case class Leaf[A](from: Int, to: Int, override val res: A) extends TreeResA[A]
case class Node[A](l: TreeResA[A], override val res: A, r: TreeResA[A]) extends TreeResA[A]

def upsweep[A](inp: Array[A], from: Int, to: Int, f: (A, A) => A): TreeResA[A] = {
    if (to - from < threshold)
        Leaf(from, to, reduceSeg1(inp, from + 1, to, inp(from), f))
    else {
        val mid = from + (to - from) / 2
        val (tL, tR) = parallel(upsweep(inp, from, mid, f),
                                upsweep(inp, mid, to, f))
        Node(tL, f(tL.res, tR.res), tR)
    }
}

def reduceSeg1[A](inp: Array[A], left: Int, right: Int, a0: A, f: (A, A) => A): A = {
    var a = a0
    var i = left
    while (i < right) {
        a = f(a, inp(i))
        i = i+1
    }
    a
}

def downsweep[A](inp: Array[A], a0: A, f: (A, A) => A, t: TreeResA[A], out: Array[A]): Unit =
    t match {
        case Leaf(from, to, res) =>
            scanLeftSeg(inp, from, to, a0, f, out)
        case Node(l, _, r) => {
            parallel(
                downsweep(inp, a0, f, l, out),
                downsweep(inp, f(ao, l.res), f, r, out
            )
            )
        }
    }

```