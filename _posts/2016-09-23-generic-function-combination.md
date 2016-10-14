# Generic Function combination with typelevel projects

In this post, through some code examples, we'll see how we can take advantage of some typeclasses, data types and generic data structures to help function combinatation. We are not going to get into the implementation detail of functional programming or generic programming, it's merely a tiny demonstration of the power of them in the context of function combintion. 

## What is function combination

We combine functions together to form new functions all the time. A common way to achieve this is to write a new function in which it calls other functions. This new function defines the inputs as variables and pass them into functions. In many cases, the new function doesn't do anything but simply delegating to other functions. Let's call such functions delegating functions. Here is a simple example: 

```Scala
//suppose we have two existing funcitons
val departmentOf: EmployeeId => DepartmentId = ...
val extensionNumOf: DepartmentId => ExtensionNumb = ..

//we can construct a new funciton with them
val extensionNumOfEmployee: EmployeeId => ExtensionNum = 
  (eid: EmployeeId) => extensionNumOf(departmentOf(eid))
```
A more idiomatic way in functional programming, though, is to use function combinators. In this case we can use the `andThen` combinator from scala standard library. 
```Scala
val extensionNumOfEmployee: EmployeeId => ExtensionNum = 
  departmentOf andThen extensionNumOf
```
This approach is obviously more concise with less variable to care about and easier to read. However, we often times found ourself having to go back to delegating functions. This post is about how functional programming libaries like cats and generic programming libraries like shapeless can help us avoid delegating functions with more advanced function combination techniques. We are going to start with the easier cases and gradually move towards more sophisticated cases. 

## Function with multiple parameters

In our previous example, the second function takes as parameter the output of the first function. Here is another scenario. 
Suppose we have two functions
```Scala 
val budget: DepartmentId => Double = ...
val numOfEmployees: DepartmentId => Int = ..
val divide: (Double, Int) => Double = _ / _ 
```
How do we cominbe these three into one that gives us the budget per employee without delegating funcitons? 
```Scala
val budgetPerEmployee: DepartmentId => Double = ???
```
Let's achieve this with some help from cats. 

```Scala 
import cats.implicits._

val budgetPerEmployee: DepartmentId => Double = 
  (budget, numOfEmployees) map2 divide
```
We can do this because functions forms `ApplicativeFunctor`s and `ApplicativeFunctor`s support such product composition. Here we construct a function combinator by composing functions as `ApplicativeFunctor`s. We are taking advantage of the fact that Cats provides `Functor` and `Cartesian` instances (a different form of `ApplicativeFunctor`) for functions as well as a simple composition syntax as seen above. 


## Combine function using Monad and Applicative

## Combine functions with effects

### Async

### Error handling

## Combine functions with heterogeneous arguments

### generice function combinator 

### `contraMapR`

### `to` 

## put it together. 
