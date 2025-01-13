# An Initiation to Domain-Driven Design
<time datetime="2025-01-13 07:00">13th Jan 2025</time>

What is Domain-Driven Design (DDD)? I used to group it with the other programming acronyms (TDD, SOLID, KISS, YAGNI, DRY, etc.). Maybe something developers _should_ practice, but often don’t. That changed after I read Vaughn Vernon's book [**Implementing Domain Driven Design**](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/).

As developers, we are told what constitutes "good code". There should be high cohesion, low coupling, single responsibilities, information hiding, and so on. But these ideas are abstract. While the benefits of these characteristics are clear, it’s not always obvious how to achieve them. Well, DDD is one way. It describes a number of patterns that will help you write clearer, less coupled, extensible, cohesive code that aligns strongly with the business' goals.

A core tenet is that your code should model the domain you are building a solution for. If a business process change is conceptually simple, so is the required code change. No more explaining to a product manager why a basic feature request will actually take weeks to implement. I am relatively new to this methodology. So I won't provide a more detailed description than that. Instead, I will share some useful DDD patterns I've come across. These are good general programming practices. You don't need to "do" DDD to follow these. But if you think they are insightful, I would encourage you to check out the resources [listed at the end](#further-reading).

**Some useful DDD patterns:**<br/>
1. [Understand what you're building, and why.](#understand-what-youre-building-and-why)<br/>
2. [Understand your relationship with other services.](#understand-your-relationship-with-other-services)<br/>
3. [Understand the type of object you’re creating.](#understand-the-type-of-object-youre-creating)<br/>
4. [Use immutable Value Objects.](#use-immutable-value-objects)<br/>
5. [Use rich domain models.](#use-rich-domain-models)<br/>
6. [Hide your domain models from the outside world.](#hide-your-domain-models-from-the-outside-world)<br/>

## Understand what you’re building, and why

Don’t make assumptions about what you’re building. Discuss and explore it with the domain expert. This could be your product manager or clients. Through this collaboration, both the technical and non-technical team members will better understand their business, user needs, and value they are delivering. The process of exploration may even help the domain experts uncover areas of uncertainty. Plan and design. Do not immediately jump into coding.

Part of this process involves the creation of a **Ubiquitous Language**, a shared language describing the domain. A common approach for this modelling process is [Event Storming](https://www.eventstorming.com/). The **Ubiquitous Language** is used by both domain experts and within the codebase. As Vaughn Vernon writes in **Implementing Domain-Driven Design**:

> We should build software that is as close as possible to what the business leaders and experts would create if they were the coders… There are zero translations between the domain experts, the software developers, and the software.

Imagine an application for booking tickets at a cinema. What's a good name for a function that saves the finalised booking in the backend? Should the method be `createBooking` or `confirmBooking`? The word "create" leaks developer knowledge into the codebase. They know a new record is being created in the database. And, ultimately, that is what this function does. But the name `confirmBooking` better describes the real life use case. "Create" is not the verb a customer would associate with the confirmation step of the booking process.


## Understand your relationship with other services

Before designing a service, take a moment to understand how it fits into your wider technology ecosystem. Some services exist to support unrelated parts of the business. And there are services that your new service will need to interact with. Understanding this space is called **Context Mapping**. You might incidentally do this when designing a new system. However, for DDD it is an _explicit_ process. It goes so far as to characterise the relationships underpinning these services. Below are a few examples. Check out more at [this useful repository](https://github.com/ddd-crew/context-mapping).

#### Partnership
The upstream and downstream teams succeed or fail together. Models and interfaces are developed to suit both their needs and any features and planned between them to minimise harm.

#### Customer-Supplier Development
The success of the upstream team (the supplier) is independent of the downstream team. But they take the downstream's needs into account when planning.

#### Conformist
The upstream team does not accommodate the downstream team’s needs. The downstream team conforms to the models of the upstream team, whatever they are or change to.

A lot of confusion and frustration might be avoided if these relationships are agreed between participating teams ahead of time.

## Understand the type of object you’re creating

In object-oriented programming (OOP), our codebases are full of classes. These are often loosely categorised into vague concepts like services and repositories. However, these categories and their definitions vary from one project (or developer) to the next. So in each new codebase, we spend time understanding the responsibilities of each class.

Without formal definitions, it is easy for the scope of these classes to grow. And then the level of abstraction becomes lost. Fortunately, DDD provides concrete definitions for model categories, and rules for how they can interact. The vocabulary is simple, yet empowering. When you don’t have to define the building blocks of your application, you can focus more on what you’re building. These are the “tactical patterns” of DDD. Some examples are:

#### Value Objects
Immutable objects that represent a value with no identity. These can be replaced, but cannot change, over time. Two **Value Objects** are considered equal if they hold the same values (i.e. structural equality).

#### Entities
Objects that have a unique identity. Two **Entities** with the same identifier are considered equal, even if all other properties are different in value. These represent concepts in your domain that you need to observe and track changes to over time.

#### Aggregates
These are a grouping of **Entities** and **Value Objects** that represent a consistency boundary. If there is some invariant or business rule that must be maintained when updating a group of objects (and rolled back everywhere upon failure), then they belong together in an **Aggregate**. One **Entity** will serve as the **Aggregate Root**, through which all behaviour is orchestrated.

What if another **Entity** is changed at the same time as the **Aggregate**? If there is no requirement to maintain consistency between the two objects (and _really_ challenge yourself on this point!) then it shouldn't be part of the **Aggregate**. Instead, it can be updated through an event-driven process that is eventually consistent. The goal is to make each **Aggregate** as lightweight as possible.

#### Domain Services
A **Domain Service** helps realise business logic and coordinates actions between your domain models (i.e. **Aggregates**). Importantly, only use **Domain Services** when the behaviour being orchestrated cannot logically belong to a single **Aggregate** (see [below](#use-rich-domain-models)).

#### Application Services
These allow your domain logic to interface with the external world (API, persistence stores, etc). No business logic belongs here. Each method represents a single business use case. And each method achieves its goal by interacting with the **Aggregates** and **Domain Services**.

## Use immutable Value Objects
Try using **Value Objects** wherever possible. Replacing both primitives and classes with **Value Objects** will make your code more self-documenting, easier to maintain, and less prone to bugs.

Let us consider a Kotlin application for booking tickets at a cinema. Perhaps it was first written like below.

```kotlin
class TicketDetail(
    var ticketType: TicketType,
    var filmId: UUID,
    var sessionTime: LocalDateTime,
    var seatTheatre: Int,
    var seatRow: Char,
    var seatNumber: Int,
    var price: BigDecimal,
    var discount: BigDecimal,
)

fun confirmBooking(
    customerId: UUID,
    cinemaId: UUID,
    ticketDetails: List<TicketDetail>,
) {
    // Business logic...
}
```

We can improve this by introducing **Value Objects**. **Value Objects** are immutable, so we replace all use of `var` with `val`. And we use `data class` to enable structural equality when comparing objects. The same can be achieved by record classes in Java and data classes in Python.

```kotlin
data class CustomerIdentifier(val value: UUID)
data class CinemaIdentifier(val value: UUID)
data class FilmIdentifier(val value: UUID)

data class Seat(
    val theatre: Int,
    val row: Char,
    val number: Int,
)

data class TicketPrice(
    val price: BigDecimal,
    val discount: BigDecimal,
) {
    // Validates that discount < price
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
    val sessionTime: LocalDateTime,
    val ticketDetails: List<TicketDetail>,
) {
    // Validates at least one seat being booked
    // Validates all seats positions are unique
}

fun confirmBooking(
    details: BookingDetails,
) {
    // Business logic...
}
```

These changes introduce a number of benefits.

**Improved readability**<br/>
Each type actually means something in our domain. They are not primitives. Also, related concepts can be grouped together. For example, all aspects of the seating location (theatre, row, seat number) are grouped into the `Seat` class. This improves readability and reduces the number of method parameters.

**Better type safety**<br/>
Now it’s harder to accidentally swap the order of the method parameters that previously had the same type (i.e. `UUID`). For static languages, the compiler will complain if the values for `customerId` and `cinemeaId` are swapped. For dynamic programming languages, such errors can now be caught by the linter (if you use one).

**Improved evolvability.**<br/>
What if we want to change the data type representing an identifier (e.g. change the cinema identifier to `Int` )? Previously, we would need to update _all_ class and function definitions using that identifier. Now we only need to update a few lines in the `CinemaIdentifier` class.

**Validation**<br/>
Basic data validation is included in the **Value Object** construction. This is much better than scattering around (and possibly repeating) validation in the methods the data is used. It improves reliability, because the data is _guaranteed_ to be validated. And it improves readability, as the business logic can now focus on the use case.

**Immutability**<br/>
**Value Objects** are immutable. Once one is instantiated and validated, you know it is valid, always. Because **Value Objects** are immutable, any methods are side-effect free. **Value Objects** can be passed around a multi-threaded application with confidence.

## Use rich domain models
Don't compose your application with only weak (or anaemic) domain models. These are classes that hold data but don't include any business logic. The result is that all logic is pushed into what developers often label "services" (but these are different to the DDD definition of Services [provided above](#domain-services)).

For an application with anaemic domain models, a developer might write a class like below to perform the business logic.
```kotlin
class BookingService {

    fun confirmBooking(details: BookingDetails) {
        // Business logic...
    }

    fun updateBooking(bookingId: BookingIdentifier, updatedDetails: BookingDetails) {
        // Business logic...
    }

    fun cancelBooking(bookingId: BookingIdentifier) {
        // Business logic...
    }

}
```

What's wrong with this? Well, the term "booking service" is pretty vague. It currently handles the logic for the creation, modification, and cancellation of a single booking. But should it include logic for listing historical bookings made by a customer? Or searching for a particular booking? Hmm, maybe, but possibly not... Regardless, if a product manager requests this functionality, I bet the logic will be added to this class. And through this process the class grows in scope and loses cohesion. And each new functionality will require more injected dependencies. Over time this scope creep creates [God Classes](https://en.wikipedia.org/wiki/God_object).

What is the alternative? In DDD domain models aren't necessarily simple data holders. They represent a core concept in your domain. Thus, they should not only hold the data, but also the behaviours that involve that data. These are called rich domain models.

In the previous section I treated the `BookingDetails` as a **Value Object**. But if we put our DDD hats on, we can see a booking is actually better modelled as an **Entity**. That's because each booking is unique. It is a _thing_ that can be updated over its life. Thus we need to track it with a unique identifier. With some minor adjustments, we can improve our booking service from before.

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
    val bookingId: BookingIdentifier = BookingIdentifier.new(),
    val customerId: CustomerIdentifier,
    val cinemaId: CinemaIdentifier,
    val filmId: FilmIdentifier,
    var sessionTime: LocalDateTime,
    var ticketDetails: List<TicketDetail>,
) {

    fun changeSeats(newSeats: List<Seat>) {
        ticketDetails = ticketDetails.zip(newSeats).forEach {ticket, seat ->
            ticket.withNewSeat(seat)
        }
    }

    fun cancelBooking() {
        // Business logic...
    }

}
```

Our booking class clearly represents a single booking, and actions associated with it. It is unlikely a future developer would bastardise this class by adding methods like `getAllBookings` or `findBooking`. Thus, we have better preserved the cohesion of our codebase going into the future. And maximising the maintainability of a codebase is critical for any long-lived application. There is no longer a "create" function, as this is simply performed through the act of instantiating the class. And the actions that can be made on this **Entity** are clearly defined by the class methods. Instead of the vague `updateBooking` method, we have `changeSeats`. The code reflects the actual use case and tells the developer why it's there.

## Hide your domain models from the outside world

All projects have dependencies for communicating with the outside world. These could help serve RESTful endpoints, communicate with other services via RPC, or connect to a database. While necessary, none of this has anything to do with the purpose of the service. So, there is no reason to couple these external dependencies to any core business logic. This principle is often referred to as [clean architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html). DDD is not the only proponent of this style. But, with its strong emphasis on considered domain models, it does scream out for such an approach.

Any layered architecture can achieve the separation of concerns required for clean architecture. My favourite is the Ports and Adapters (i.e. Hexagonal) style. It strongly promotes the use of clean domain models. The "ports" surround your core logic and serve as points of entry to your application. These ports are typically defined as interfaces. The concrete implementations of these interfaces are the "adapters" that leverage libraries and other dependencies to serve their function.

<figure>
![](images/ddd_clean_architecture.png)
<figcaption>An illustration of the Ports and Adapters architecture, where arrows point in the direction of dependency. If needed, we can easily change our datastore from PostgreSQL to MongoDB. Inbound and outbound ports don't only interface with APIs and databases. This is only an example.</figcaption>
</figure>

What's wrong with a simple three-layer architecture, with a presentation, application, and data-access layer? If done well, nothing. But, particularly if combined with an object-relational mapping (ORM) framework, it can enable developers to take shortcuts. These reduce the maintainability of the project over its life. For example, with the ORM classes it is possible for the presentation layer to directly interact with the database. And a lazy developer might choose to do this for a simple CRUD operation, instead of passing data unnecessarily through the application layer. Over time, such a shortcut might occur for another reason. And then again, and again. The result is logic spread over all the layers of your application, defeating the original purpose of the layered architecture.

Further, ORMs promote database driven design. It's easier to pretend your ORM classes are the core models of your application. But that means your "model" actually reflects how the data is organised in the database, not how it's represented in real life. And if these ORM classes are used as your core "models", then good luck ever switching frameworks down the line. The ORM dependency is now tightly coupled to all your logic. You will effectively need to re-write your entire application. I have also seen the same mistake occur for gRPC generated data access classes. Best use a clean architecture and leave these dependencies out of your core business logic 😌

## Further reading
Am I an expert in DDD? Nope. But some of the insights above motivated me to apply it more, and to keep learning. If you feel the same, I've listing some additional resources below.<br/><br/>
[Domain-Driven Design Reference](https://www.domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf) - Eric Evans<br/>
[Domain-Driven Design](https://www.oreilly.com/library/view/domain-driven-design-tackling/0321125215/) - Eric Evans<br/>
[Implementing Domain-Driven Design](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/) - Vaughn Vernon<br/>
[Domain-Driven Design Crew GitHub repository](https://github.com/ddd-crew)<br/>
[EventStorming website](https://www.eventstorming.com/)<br/>
