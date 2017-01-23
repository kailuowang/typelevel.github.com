# Generic Function combination with libraries from Typelevel

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

## Combine functions with multiple parameters

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
We can do this because functions forms `ApplicativeFunctor`s and `ApplicativeFunctor`s support such product composition. Here we construct a function combinator by composing functions as `ApplicativeFunctor`s. We are taking advantage of the fact that Cats provides `Functor` and `Cartesian` instances (a different form of `ApplicativeFunctor`) for `Function1` with input type fixed.  

## Combine functions with effects

Now what if our functions is asynchrnous? Or what if our functions don't return the result directly, rather, they return results in some context, such as a `Future` or `Option`. For example:
```Scala
val departmentOf: EmployeeId => Future[DepartmentId] 
val extensionNumOf: DepartmentId => Future[ExtensionNumb] 
```
We cannot call `departmentOf andThen extensionNumOf`, nor can we use the `ApplicativeFunction` product composition metioned above. This is where `Kleisli` comes to rescue. `Kleisli` is a class that wraps a `A => F[B]` and, in some sense, let us treat it as a `A => B`.
```Scala
case class Kleisli[F[_], A, B](run: A => F[B])
```
For example, now we can chain the two async functions above by wrapping them to `Kleisli` first. 
```Scala 
val extensionOfEmployee: Kleisli[Future, EmployeeId, ExtensionNum] =
  Kleisli(departmentOf) andThen Kleisli(extensionNumOf)
  
extensionOfEmployee.run(employeeId) //returns Future[ExtensionNum]
```
`Kleisli` also forms `Monad` which is `ApplicativeFunctor` as well. So we can use the same `ApplicativeFunctor` composition mentioned above. 
```Scala 
val budget: DepartmentId => Future[Double] 
val numOfEmployees: DepartmentId => Future[Int] 
val divide: (Double, Int) => Future[Double]

val budgetPerEmployee: Kleisli[Future, DepartmentId, Double]
  (Kleisli(budget), Kleisli(numOfEmployees)) map2 Kleisli(divide)  
```

### Combine functions with nested effects

Now suppose, our functions also uses `Either` for error handling. 
```Scala
val extensionNumOf: DepartmentId => Future[Eiter[MyError, ExtensionNum]]
val departmentOf: EmployeeId => Future[Either[MyError, DepartmentId]]
```
The function now returns a `Future` of `Either` a `MyError` or a `ExtensionNum`. `Kleisli` alone won't be able to handle it. This is where MonadTransformer (or `Nested` in some case) can help. For `Either` cats provides a `EitherT` wraps a `F[Either[A, B]] ` so that it can be treated as a `Monad` containting `B`, as long as `F` forms a `Monad`. For example, take our `extensionNumOf` function, we can make it a function that returns a `EitherT` 
```Scala
val extensionNumOf1: DepartmentId => EitherT[Future, MyError, ExtensionNum] = extensionNumOf andThen EitherT
```
Now with the help of `EitherT` and `Kleisli` we can combine the function again using `andThen`

```Scala
val extensionOfEmployee: Kleisli[EitherT[Future, MyError, ?], EmployeeId, ExtensionNUm] =
  Kleisli(departmentOf andThen EitherT) andThen Kleisli(extensionNumOf andThen EitherT)
```

Ok, that's a bit too much going on to read. We can define some types to simplify it. 

```Scala 
object MyTypes {
  // represent a Future[Either[MyError, T]]
  type Result[T] = EitherT[Future, MyError, T]
  
  //represent a Function from A => Result[T]
  type K[A, B] = Kleisli[Result, A, B]
}
```
Then we can just use these types accross the application and make our code a lot cleaner. 
```Scala
val departmentOf: K[EmployeeId, DepartmentId]
val extensionNumOf: K[DepartmentId, ExtensionNum]

val extensionOfEmployee: K[EmployeeId, ExtensionNUm] =
  departmentOf andThen extensionNumOf
  
val budget: K[DepartmentId, Double] 
val numOfEmployees: K[DepartmentId, Int] 
val divide: K[(Double, Int), Double]

val budgetPerEmployee: K[DepartmentId, Double]
  (budget, numOfEmployees) map2 divide
```

## Combine functions with heterogeneous arguments

The functions we examined above takes 1 or 2 primitive arguments, they also work nicely together with common argument type and return types. Real-world functions usually has multple heterogeneous arguments. To demonstrate that, let's introduce a more realistic application - The student registration system for HSWW (Hogwarts School of Witchcraft and Wizardry). 

In this application we have several functions providing business logic implementations, each one taking a quite different set of arguments and producing different types of results. 

```Scala
  def wandService: K[GetWandRequest, Wand]
  def mentorService: K[AssignMentorRequest, Mentor]
  def dormService: K[AssignDormRoomRequest, DormRoom]

  //protocols
  case class GetWandRequest(name: String, address: String)
  case class AssignMentorRequest(wand: Wand, specialty: Specialty)
  case class AssignDormRoomRequest(name: String, sex: Sex, mentor: Mentor)
```

Here we encode the argument signatures into case class definitions, that is each function takes one argument of a case class which has multiple fields. Also we used our `K` type alias, which represents a function with some effects. 

The registration application gets a request of type `Map[String, String]` and needs to return a `Future[Either[Error, RegistrationRecord]]`
The `RegistrationRecord` is defined as
```Scala
case class RegistrationRecord(basicInfo: BasicInfo,
                                mentor: Mentor, 
                                dormRoom: DormRoom)
```
In which the `mentor` and `dormRoom` should be assigned using the services. 
The `BasicInfo` is defined as 
```Scala
case class BasicInfo(name: String, address: String, age: Int, sex: Sex)
```
It is parsed from the `Map[String, String]` request. 

### generice function combinator 

### `contraMapR`

### `to` 

## put it together. 
