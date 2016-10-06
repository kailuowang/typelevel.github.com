# Generic Function combination with typelevel projects

In this post, through some code examples, we'll see how we can take advantage of some typeclasses, data types and generic data structures to help function combinatation. 

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

## Combine function without context

## Combine function using Monad and Applicative

## Combine functions with effects

### Async

### Error handling

## Combine functions with heterogeneous arguments

### generice function combinator 

### `contraMapR`

### `to` 

## put it together. 
