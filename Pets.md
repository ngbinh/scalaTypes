derived from http://tpolecat.github.io/2015/04/29/f-bounds.html

# Problems
> for any expression x with some type A <: Pet, ensure that x.renamed(...) also has type A at compile time.

# Subtyping solution

1. Base trait
```scala
trait Pet {
  def name: String
  def renamed(newName: String): Pet
}
```

2. Fish
```scala
case class Fish(name: String, age: Int) extends Pet {
  def renamed(newName: String): Fish = copy(name = newName)
}
```

Note: `renamed` can return `Fish` because return type is *covariant*

3. Some examples
```scala
val a = Fish("Jimmy", 2)
```

```scala
val b = a.renamed("Bob")
```

4. Problem: lax constraint on implementation
This is valid:
```scala
case class Kitty(name: String, color: String) extends Pet {
  def renamed(newName: String): Fish = new Fish(newName, 42) // oops
}
```

=> too much freedom!

5. Function to wrap over renamed
```scala
def getMarry[A <: Pet](a: A): A = a.renamed("Mrs " + a.name)
```

-> compilation error!

=> too restricted!

# F-Bounded Types
> parameterized over its own subtypes

1. Base trait

```scala
trait Pet[A <: Pet[A]] {
  def name: String
  def renamed(newName: String): A // note this return type
}
```

2. F-bounded Fish
```scala
case class Fish(name: String, age: Int) extends Pet[Fish] { // note the type argument
  def renamed(newName: String) = copy(name = newName)
}
```

3. and all is well!

```scala
val a = Fish("Jimmy", 2)
val b = a.renamed("Bob")
```

4. Generic function!
```scala
def getMarry[A <: Pet[A]](a: A): A = a.renamed("Mrs " + a.name)
```

should work!

5. implementation returning the wrong type?

```scala
case class Kitty(name: String, color: String) extends Pet[Fish] { // oops
  def renamed(newName: String): Fish = new Fish(newName, 42)
}

val k = Kitty("cat")
k.renamed("Bob")
```

But with *self-type* annotation:
```scala
trait Pet[A <: Pet[A]] {  this: A => // self-type
  def name: String
  def renamed(newName: String): A // note this return type
}
```

and all is well!

6. Or does it?
```scala
class Mammal(val name: String) extends Pet[Mammal] {
  def renamed(newName: String) = new Mammal(newName)
}

class Monkey(name: String) extends Mammal(name) // hmm, Monkey is a Pet[Mammal]
```

# Welcome to the glorious type class
Scala type class:
>  type classes are defined as a tool to provide ad-hoc polymorphism. In parametric polymorphism, you define a method in a parameterized class, and it works, whatever the parameter. In ad-hoc polymorphism, however, a function is implemented differently for different types of its arguments, like 1+2 and “x”+”y”.


## Pimp my library pattern

```scala
 "Binh ".toAwesomeness

class Awesomizer(string: String) {
    def toAwesomeness = string + " is awesome"
}

implicit def awesomizeYoString(string: String) = new Awesomizer(string)

 "type class ".toAwesomeness
```

##  ad-hoc polymorphism
> any type A can act as a Pet, given an instance of Pet[A].

1. Base trait
```scala
trait Pet[A] {
  def name(a: A): String
  def renamed(a: A, newName: String): A
}
```

2. Fish
```scala
case class Fish(name: String, age: Int)

// object Fish { <- doesn't work on REPL
  implicit val FishPet = new Pet[Fish] {
    def name(a: Fish) = a.name
    def renamed(a: Fish, newName: String) = a.copy(name = newName)
  }
// }

Fish("Bob", 42).renamed("Steve")

implicit class PetOps[A](a: A)(implicit ev: Pet[A]) {
  def name = ev.name(a)
  def renamed(newName: String): A = ev.renamed(a, newName)
}

Fish("Bob", 42).renamed("Steve")
```

## So type class heh?
Problems:
1. Boiler plate
-> https://github.com/mpilquist/simulacrum

```scala
load.plugin.ivy("org.scalamacros" % "paradise_2.11.7" % "2.1.0-M5")
load.ivy("com.github.mpilquist" %% "simulacrum" % "0.4.0")

import simulacrum._

@typeclass trait Semigroup[A] {
  @op("|+|") def append(x: A, y: A): A
}

implicit val semigroupInt: Semigroup[Int] = new Semigroup[Int] {
  def append(x: Int, y: Int) = x + y
}

import Semigroup.ops._
1 |+| 2 // 3
```

2. How's about collections?
```scala
trait Pet[A] {
  def name(a: A): String
  def renamed(a: A, newName: String): A
}

case class Fish(name: String, age: Int)

case class Kitty(name: String, color: String)
```
