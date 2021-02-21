---
title: The "fundamental architecture in computer science"
---

As I get more experienced with writing code and testing it, I realize that I use a technique that I find really valuable. I believe that any developer would benefit about knowing it and using it. 

The fundamental idea is that all the code that we write belong to different "categories": *domain*, *infrastructure* and *application*. Separating those concerns make the developer job way easier.

This idea of separating *domain* and *infrastructure* concepts is at the foundation of the *Hexagonal Architecture* aka *Ports and Adapters*. 

In this article, I will not talk about *Hexagonal Architecture* or *Ports and Adapters* specifically. I will rather discuss underlying concepts, why those architectures are useful, what are its principles and present a concrete implementation. 

## Business logic: To be or not to be
The business logic describes the problem that the software is trying to solve. For instance, when creating a train booking software, the business logic would include the representation of the train, seats, passengers along with the logic of booking a seat. 
 
Often, the business logic uses the same vocabulary as the domain expert. A domain expert is someone specialized in the domain or the problem that you are solving with the software also known as Product Owner. You might have heard this vocabulary being called the [Ubiquitous Language](https://martinfowler.com/bliki/UbiquitousLanguage.html).

For the train example above, you can think that a domain expert would talk about passengers, seats, cars and train. These concerns belong to *domain*. On the contrary, domain experts would probably not talk about databases, message queues or REST APIs. These concerns belong to *infrastructure*.

The business logic by itself not really useful. The application needs to be able to communicate to the user or other applications and to "remember" things. That's where tools like databases, message queues or REST APIs comes useful. 


## The golden rule
Once *domain* and *infrastructure* categories are identified, the actual code can be separated out physically (in different packages or modules) respecting the following (golden) rule: **the domain should not depend on the infrastructure**. In other words, the domain should have no knowledge and no reference about databases, messages queues or REST APIs.

There are few very good reasons for this rule to exist. 

First, the *domain* *should be* the most valuable piece of the software. The domain should be as free as possible to change in order to adapt to a competitive market. The least constrained the *domain* is, the freest it is to change, the quicker the value can be brought to customers.

Secondly, testing become way easier. Testing is fundamental help (1) bringing consistent values to customers and (2) to give software the best chances to evolve over the time. Earlier, we mentioned that *domain* code is where the value of the software resides. In other words, that is the part of the software to get right and making sure it stays right. Hence, be heavily tested. 

Separating *domain* and *infrastructure* concerns facilitates testing. Indeed, requiring no reference to *infrastructure* concepts to write domain tests simplifies the process significantly. Concretely, you don't have to think about mocking the database because you don't need to. 

## Dependency inversion principle

When I was first introduced to the concept, I remember asking myself: "if I can't have any reference of the database in the domain, how do I save the entity?"

That's where the dependency inversion principle comes handy. You probably already heard about this concept, it is the "D" of SOLID. The idea is to interact with an interface rather than the actual implementation.

Concretely, as the *golden rule* does not allow any reference to database, the domain code uses an interface (known as *Port*). The implementation (known as *Adapter*) would be in the infrastructure layer. That way, the rule is not broken.

## Concrete example: registration of a user

Let's take a simple example to illustrate. Imagine you are asked to implement to book a seat for a passenger in a train. The rules are: 

- seats are identified by the car id and the seat number;
- you should check that the seat is not already booked in the database. If not, then book it for the passenger requesting it. 

> For simplification purposes, we imagine that there is only one train.

Take a few minutes to imagine how would you implement that. Keep in mind to separate the domain and the infrastructure layers.

> For this article, I am going to use Scala but this can be applied for any language allowing dependency inversion principle. 

### Domain layer

First, the domain model:
```scala
package domain.model

// The entity to be saved is SeatBooked
// It can also be called the "aggregate root"
case class SeatBooked(seat: Seat, customer: Customer)

// Seat is the "id" of SeatBooked
case class Seat(carId: String, seatNumber: Int)

// Illustrative implementation of a Customer 
case class Customer(name: String)
```

And then the function signature to implement:
```scala
package domain

class SeatReservationService {
  def book(seat: Seat, customer: Customer): Unit = ???
}     
```
Now, the repository. Following, the *dependency inversion principle*, let's start by the interface:

```scala
package domain

trait BookedSeatRepository {
  def find(seat: Seat): Option[SeatBooked] 
  // ^^^ The `Option` here expresses that there might 
  // or might not be any `SeatBooked` returned
  // by this function
  def save(seatBooked: SeatBooked): Unit
}
```
Usually, a simple interface like this example covers most use cases.

> One important thing to keep in mind is that the interface is gong to reside in the domain package. As a consequence, it can have no knowledge about the code related to the database. 

No need to implement the repository to create the business logic of the function register:

```scala
package domain

class SeatReservationService(bookedSeatRepository: BookedSeatRepository) {
  def book(seat: Seat, customer: Customer): Unit = {
    val seatBooked = 
      bookedSeatRepository.find(seat).isDefined
    if(! seatBooked){
      bookedSeatRepository.save(
        SeatBooked(seat, customer)
      )
    }
  }    
} 
```

> I'd like to emphase, how easy it is to test the business logic. Actually, there is no need to have a database configured in the project at all. For instance, the `BookedSeatRepository` can be mocked or implemented in-memory using a  `Map[Seat, Customer]`. That would be enough to test the business logic.

### Infrastructure layer

Finally, the implementation of the booked repository in the infrastructure layer. 
```scala
package implementation

class BookedSeatRepositoryPostgres 
  extends BookedSeatRepository {

  def find(seat: Seat): Option[SeatBooked] = ???
  def save(seatBooked: SeatBooked): Unit = ???
}   
```
I chose to use Postgres here as an example but you could imagine any other technology. 

> Note: I used `???` as the actual implementation is beside the point.

> This architecture allows you to have several implementations. For instance, having an in-memory implementation is very useful for integration tests (for instance, business end-to-end tests *à la* BBD).

### Application layer

There is one more layer that I haven't introduced yet: the *application* layer. Typically, the *application* layer contains the "main" function (executed when the application is launched). This layer has the responsibility to initialize *infrastructure* classes and pass them to the *domain*. 

> In some languages (like PHP), there is no "main" function. The way applications run is different from JVM based applications for instance. As such, the section is slightly less relevant. But the rest is :).

Other layers (*domain* and *infrastructure*) have no knowledge about the *application* layer. Here is a dependency diagram to illustrate:
<img alt="Screenshot 2021-02-15 at 19 48 49" src="https://user-images.githubusercontent.com/4415022/107989728-c68ae380-6fca-11eb-9b33-f5096b8a6583.png">

For our example, the application layer would contain only the main class:
```scala
package application

object Main {
  def main(args: Array[String]): Unit = {
    // instanciation of classes
    val bookedSeatRepository: BookedSeatRepository = 
      new BookedSeatRepositoryPostgres()
    val seatReservationService = 
      new SeatReservationService(bookedSeatRepository)
  
    // calling the service to register the user
    val seat = Seat("A", 14)
    val customer = Customer("Damien")
    seatReservationService.book(seat, customer) 
  }
}  
```
For the moment, the application is not really useful as no http service is calling method `book` (the point is is to illustrate).

### Wrapping up

Overall, its roughly looks like that: 
![Screenshot 2021-02-16 at 21 29 34](https://user-images.githubusercontent.com/4415022/108123660-1dadb880-709e-11eb-931c-16eeff604510.png)

If you look at the direction of the arrows, you can the that the rule is respected.

## Challenges of implementing this pattern

I remember, when I started using this architecture, I faced several challenges.

### Where to put what?
Beginners of this pattern might have trouble figuring out the *right* boundaries of each layer. 

It can be putting a class (or the concern) in the wrong layer. For instance, I don't think that the configuration related code should be in *infrastructure* but rather in *application*. It is an *application* related concern. It only matters when the application is running. Therefore, putting it in the *domain* or *infrastructure* layer would be less relevant. 

Sometimes, a more subtile error can happen: the concern is in the *right* layer but some classes are passed on to another layer. This is known as a "leak". For instance, an infrastructure concern is passed on to the domain. Typically, it happens when the response class of the http service is returned by accident to the domain. This can also happen in the case of a Web Server when the http request class would be passed on to the domain as a parameter. In that case, the golden rule is not respected.

My advise is to ask yourself for each class (or each responsibility) in which layer it belongs. 

### Frameworks
Users of frameworks can be a bit confused when it comes to fit this pattern with the framework conventions. A tip is to consider that framework are part of external layers like *application* or *infrastructure*. As such, try to keep the framework code as far away from the domain as possible.

Actually integrating this pattern in a framework can be difficult. Frameworks can be quite invasive. I believe most modern frameworks can be configured to fit the requirement of the architecture.

Otherwise, having a set of libraries over a framework allows a greater flexibility in order to put the concerns in *right* layer

## Conclusion
In this article, we saw we can improve the code we write by: 

- **identifying the code belonging to those 3 categories: *domain*, *infrastructure* and *application*;**
- separating it out the code physically (i.e. in different packages or modules);
- respecting the golden rule: **the domain should not depend on the infrastructure**.

The dependency inversion principle is a cornerstone to implement this architecture.

Event though, implementing this architecture can be challenging, it has multiple benefits:

- It allows to focus on the business model, the most valuable piece of the software. This leads to more values being brought to the customer quicker.
- Testing becomes much easier. In particular, it allows to focus the testing effort on what matters, the domain.
- I find easier to reason about the software when concerns are well separated into those different layers.


-----

> I'd like to thanks Arnaud LEMAIRE and Thomas PIERRAIN for sharing those ideas. I really encourage you to read and watch their articles and conferences (in French). 

> Thanks to Anaël, Amanda and Clément for their insightful advises and proof-reading the article.