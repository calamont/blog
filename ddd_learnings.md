What is Domain Driven Design (DDD)? I used to bucket it with the other programming acronyms (TDD, SOLID, KISS, YAGNI, DRY, etc.). Probably something we should practice, but often don’t.

But now I've read Vaugn Vernon's book Implementing Domain Driven Design. I have some reservations recommending the book to _everyone_. But nearly every chapter had thought-provoking ideas about how to design software. As developers, we are told what constitutes "good code". There should be high cohesion, low coupling, single responsibilities, information hiding, and so on. But these ideas are abstract. While it's clear _why_ we want these properties, it’s not always obvious how they are achieved. Well, DDD is one way how. By following the approach of DDD, you should have clearer, extensible, less coupled, cohesive code that aligns strongly with the goals of the business.

## Key Learnings

What is DDD? The core tenant of DDD is that your code should model the domain you are building a solution for. As Vaughn Vernon writes:

> [We should build] software that is as close as possible to what the business leaders and experts would create if they were the coders… There are zero translations between the domain experts, the software developers, and the software.

Writing such code requires constant discussion and designing with the domain experts to create a Ubiquitous Language. Through this collaboration both the technical and non-technical team members will better understand their business, user needs, and value they are delivering. If this sounds vague, it’s because it is. But this book, and DDD more broadly, has many concrete patterns that help deliver. These cover system design, project architectures, and writing better classes and methods.

Here are some useful ideas I’ve learnt so far, for your convenience. Even if you don’t implement full-blown DDD, these are good programming practices:

1. [Understand what you're building, and why.](#understand-what-youre-building-and-why)
2. [Understand your relationship with other services.](#understand-your-relationship-with-other-services)
3. [Understand the type of object you’re creating.](#understand-the-type-of-object-youre-creating)
4. [Use immutable Value Objects.](#use-immutable-value-objects)
5. [Use rich domain models.](#use-rich-domain-models)
6. [Business logic belongs in a domain model, not a service class.](#business-logic-belongs-in-a-domain-model-not-a-service-class)
7. [Hide your domain models from the outside world.](#hide-your-domain-models-from-the-outside-world)

## Understanding what you’re building, and why

Don’t make assumptions about what you’re building. Discuss and explore it with domain expert. This could be your product manager or clients. The process of exploration will allow you to understand the use case, and may even help the domain experts uncover areas of uncertainty.  Plan and design. Do not immediately jump into coding.

Something about shared language - then say where this language is relevant is the Bounded Context.

As noted earlier, the language used in our codebase should mirror that used by the domain experts. Through this we minimise any translation that is needed between describing how the software works and the real life processes it is modelling.

Let us think of an application for booking tickets at a cinema. How would you describe finishing the booking process. Should the method be `createBooking` or `confirmBooking`? I think the later better reflects the language that describes the process in real life. You would never see "create booking" written on cinema's website. The word "create" is leaking the developers knowledge of the backend process into the codebase (as we know a new row is being written into a database somewhere).

## Understand your relationship with other services

Part of DDD involves zooming out and understanding where your context interacts with others. (To simplify, we’ll assume each service within a microservices architecture represents some context in the business.) This is called a Context Map, and includes the connections between Bounded Contexts (i.e. services). No one would build a new service without doing this in some form. But DDD goes a step further and asks you to also think about the type of these relationships and integration patterns that will be used.

Here are a few relationship examples:

**Partnership**
The upstream and downstream teams succeed or fail together. Models and interfaces are developed to suit both their needs and any features and planned between them to minimise harm.

**Customer-Supplier Development**
The success of the upstream team (the supplier) is independent of the downstream team. But they take the downstream's needs into account when planning.

**Conformist**
The upstream team does not accommodate the downstream team’s needs. The downstream team conforms to the models of the upstream team, whatever they are or change to.

A lot of confusion and frustration might be avoided if these relationships are agreed between participating teams ahead of time.

## Understand the type of object you’re creating

In OOP, our code bases are full of classes. These are often loosely categorised into concepts like services, repositories, data classes, etc. However, these categories and their definitions vary from one project (or developer) to the next. So in each new code base, we spend time understanding the behaviour and responsibilities of each class.

Importantly, when these concepts have no definitions, it is easy for the scope of a class to grow. Then the level of abstraction becomes muddled. (TODO: Delete this sentence? - Also, terms like “service” are so overloaded to be meaningless.)

Fortunately, DDD provides concrete definitions for object categories, and rules for how they can interact. The vocabulary is simple, yet empowering. When you don’t have to think about defining and creating the building blocks, you can focus more on what you’re building

These are the “tactical patterns” of DDD. Some examples are:

**Value Objects**
Immutable objects that represent a value with no identity. These can be replaced, but cannot change, over time. Two value objects are considered equal if they hold the same values (i.e. structural equality).

**Entities**
Objects that have a unique identity. Two entities with the same id are considered equal, even if all other properties are different in value. These represent concepts in your domain that you need to observe and track changes to over time.

**Aggregates**
Aggregates are a grouping of Entities and Value Objects that represent a consistency boundary. If there is some invariant or business rule that must be maintained when updating a group of objects (and rolled back everywhere upon failure), then they exists in an Aggregate. An Entity will serve as the Aggregate Root, through which all behaviour is orchestrated. Make these Aggregates as light weight as possible.

**Domain services**
Help realise business logic and coordinates actions between your domain models (i.e. Aggregates). Only use domain services when the behaviour being orchestrated cannot logically belong to a single Aggregate. 

**Application services**
Allows your domain logic to interface with the external world (API, persistence stores, etc). No business logic belongs here. Each method represents a single business use case. And each method achieves its goal by interacting with the domain models and domain services.

## Use immutable Value Objects
Try using Value Objects where ever possible. Replacing both primitives and classes with Value Objects will make code more self-documenting, easier to maintain, and less prone to bugs.

Let us consider an application for booking movie tickets at a cinema. Perhaps it was first written like below:

```kotlin
class TicketDetail(
    var ticketType: TicketType,
    var filmId: UUID,
    var sessionTime: Local,
    var seatTheatre: Integer,
    var seatRow: Char,
    var seatNumber: Integer,
    var price: Decimal,
    var discount: Decimal,
)

fun confirmBooking(
    customerId: UUID,
    cinemaId: UUID,
    ticketDetails: List<TicketDetail>,
) {
    ...
}
```

We can improve this by introducing value objects. Value objects are immutable, so we replace all use of `var` with `val`. And we use `data class` to enable structural equality when comparing objects. The same can be achieved by record classes in Java and dataclasses in Python.

```kotlin
data class CustomerIdentifier(val value: UUID)
data class CinemaIdentifier(val value: UUID)
data class FilmIdentifier(val value: UUID)

data class Seat(
    val theatre: Integer,
    val row: Char,
    val number: Integer,
)

data class TicketPrice(
    val price: Decimal,
    val discount: Decimal,
) {
    // Validates discount <= price
}

 data class TicketDetail(
    val ticketType: TicketType,
    val seat: Seat,
    val price: TicketPrice,
) {
}

data class BookingDetails(
    val customerId: CustomerIdentifier,
    val cinemaId: CinemaIdentifier,
    val filmId: FilmIdentifier,
    val sessionTime: Local,
    val ticketDetails: List<TicketDetail>,
) {
    // Validates at least one seat being booked
    // Validates all seats are unique
    // Validates session not in the past
}

fun confirmBooking(
    details: BookingDetails,
) {
    ...
}
```

These changes introduce a number of benefits.

1. Improved readability

Each type actually means something in our domain. They are not primitives and related concepts can be bundled together. For example, all aspects of the seating location (theatre, row, seat number) are grouped into the `Seat` class. This reduces the number of parameters on the method. This bundling improves readability and also reduces the number of parameters that need to be passed to classes and functions.

2. Better type safety

Now it’s harder to accidentally swap the order of the method parameters. The compiler will complain if the value for `customerId` and `cinemeaId` are swapped. Before, this would have been acceptable as they were simply UUIDs.

3. Improved evolvability.

Now if we want to change the data type to represent an identifier (say `UUID` to `Integer` ) then we only need to update a few lines in the identifier class. Previously, we would have needed to update the function and class definitions for everywhere these objects were passed.

4. Validation

Basic validation on the data can be included in the Value Object. Without this, such validation would be scattered around (and possibly repeated) in the methods the parameters are passed to.

5. Immutability
   
Value Objects are immutable, once one is instantiated and validated, you know it is valid, always.
Because Value Objects are immutable, any methods are side-effect free. Value Objects can be passed around a multi-threaded application with confidence.

## Use rich domain models
One DDD crime I am previously guilty of is using weak (or anaemic) domain models. These are classes that hold data but don't include any business logic. All logic is pushed into what developers often labelled "services" (but I stress this is different to the DDD definition of Services provided above).

For an application with anaemic domain models, a developer might write a class like below to perform the business logic.
```kotlin
class BookingService {

    fun confirmBooking(details: BookingDetails) {
        ...
    }

    fun updateBooking(bookingId, BookingIdentifier, updatedDetails: BookingDetails) {
        ...
    }

    fun cancelBooking(bookingId: BookingIdentifier) {
        ...
    }

}
```

What's wrong with this? Well, the term "booking service" is pretty vague. It currently handles the logic for the creation, modification, and cancellation of a single booking.But should it include logic for listing _all_ historical bookings made by a customer? Or searching for a particular booking. Hmm, maybe, but possibly not... Regardless, if a product manager requests this functionality, I bet this class is where this logic will be written. And through this process the class grows in scope and loses cohesion. And each new functionality will require more injected dependencies. In the short term, this isn't an issue. But that's almost always true. Over time this scope creep creates God Classes.

What is the alternative? A tenant of DDD is that the domain models are not simple data holders. They represent a core concept in your domain. Thus, they should not only hold the data, but also the behaviours that surround that data. These are then rich domain models.

For simplicity, I treated the `BookingDetails` as a Value Object in the previous section. But if we put our DDD hats on, we can see a booking is actually better modelled as an Entity. That's because each booking is unique. It is a _thing_ that can be updated over its life. Thus we need to track it with a unique identifier. With some minor adjustments, we can improve our booking service from before.

```kotlin

 data class TicketDetail(
    val ticketType: TicketType,
    val seat: Seat,
    val price: TicketPrice,
) {
    fun withNewSeat(newSeat: Seat) {
        return new TicketDetail(ticketType, newSeat, price)
    }
}

class Booking(
    val bookingId: CustomerIdentifier = BookingIdentifier.new(),
    val customerId: CustomerIdentifier,
    val cinemaId: CinemaIdentifier,
    val filmId: FilmIdentifier,
    var sessionTime: Local,
    var ticketDetails: List<TicketDetail>,
) {
    // Handles all required validation on instantiation

    fun changeSeats(newSeats: List<Seat>) {
        ticketDetails = ticketDetails.zip(newSeats).forEach {ticket, seat ->
            ticket.withNewSeat(seat)
        }
    }

    fun cancelBooking() {
        ...
    }

}
```

Our booking class clearly represents a single booking, and actions associated with it. It is unlikely a future developer would bastardise this class by adding methods like `getAllBookings` or `findBooking`. Thus, we have better preserve the cohesion of our code base going into the future. There is no longer a "create" function, as this is simply performed through the act of instantiating the class.

that is unique Let's rethink our booking backend. It is clear now that a booking is a _thing_ that has a lifecycle. Because it needs to be tracked 
Well we can put the logic for the creation, modification, and cancellation of a booking into the `Booking` class which holds its details. In the previous section, we had a `BookingDetails` Value Object. That could be a valid desgin, depending on the user actions. But upon further though, it is clear that this object is an Entity. It will have a unique ID that describes it, but certain values can change (i.e. seat number, potentially session time). 

## Business logic belongs in a domain model, not a service class

A tenant of DDD is that the domain models are not simple data holders. They represent a core concept in your domain. Thus, they should not only hold the data, but also the behaviours that surround that data. This is known as a rich domain model (in opposition to an anaemic domain model ).

(Should I add this?) As noted earlier, there Domain Services are defined in DDD, but these only exist to coordinate business logic between entities that really doesn’t belong to a single, specific entity.

What’s the benefit? It keeps your code cohesive. The related data and all related methods exist in the same class. Concepts that are unrelated don’t get pulled in by a lazy developer because it would be so obviously wrong.

```kotlin
data class Booking(
    customerId: CustomerIdentifier,
    createdAt: Instant,
    eventId: EventIdentifier,
)

class BookingService {

    fun 
}
```

If we have anaemic domain models that are just data classes, we then need a service to perform any business logic. But once we have one service, we tend to use it for everything that is loosely connected. So it grows, and grows. And now there are methods that have nothing to do with each other. And the constructor pulls in dozens of dependencies, each only needed by only one or two of its methods. The code is not cohesive, highly coupled, unclear to read, and hard to refactor.



## Hide your domain models from the outside world

Maintain a pure internal structure to your project. Your core business logic should be independent of any current chosen serialisation protocols, message brokers, database engine.

Lean into ports and adapters (hexagonal) architecture to achieve this. Don’t use protobuf classes through out your business logic. Any update to these will likely break your business logic.



Think about what truly needs to be consistent, everything else can be async

Pushing aggregates to be as light weight as possible. Only keep in that consistency boundary what absolutely has to be there.

An aggregate doesn’t represent a data hierarchy. It is a consistency/transactional boundary. si

