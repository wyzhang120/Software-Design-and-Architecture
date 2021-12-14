# Software Design & Architecture
This repo aims to track my notes of software design and architecture.

## SOLID Principles

Refer this [course][solid_principles_course] and [repo][solid_demo] by [Steve Smith](https://github.com/ardalis) for more details.

- Single responsibility

  Each software module should have one and only one reason to change.

- Open-close

  Software entities (classes, modules, functions, etc.) should be _open for extension_, but *closed for modification*.

- Liskov substitution

  Subtypes must be substitutable for their base types. Typical violations of LSP are:

  - type checking with `is` or `as` in polymorphic code
  - null checks
  - `NotImplementedException`

- Interface segregation

  Clients should not be forced to depend on methods they do not use. Interfaces should be tailored to the context rather than all-encompassing.

- Dependency inversion

  - High-level modules should not depend on low-level modules. Both should depend on abstractions.
  - Abstractions should not depend on details, but the other way around.
  
  Relevant concepts are IoC (inversion of control) container, DI (dependency injection) strategy pattern.
  
  <img src="./Figures/SOLID_principles.PNG" alt="SOLID principles" style="zoom:50%;" />
  
  
  
  *Figure* 1 Relationships between SOLID principles  ([Steve Smith][solid_principles_course])
  
  

## Domain Driven Design

“Domain-Driven Design is an approach to software development that centers the development on programming a domain
model that has a rich understanding of the processes and rules of a domain.” -- Martin Fowler

*DDD aims to tackle business complexity, not technical complexity.* Isolate domain logic from other parts of the application.

Refer to the [DDD fundamentals][ddd-fundamental] and [DDD in practice][ddd in practice] courses for more detailed explanations of the concepts. The [clean architecture][clean-architecture] repo demonstrates how to align the structure of your code with DDD concepts.

### Concepts

- Ubiquitous language

  Terminology from a domain model that programmers and domain experts use to discuss that particular sub-system. The ubiquitous language of a bounded context is ubiquitous throughout everything you do in that context – discussion, model, code, etc.

- Core domain

  The key differentiator for the customer’s business -- something they must do well and cannot outsource. It is the inner most layer of the onion architecture (Figure 2).

  <img src="Figures/onion_arch.PNG" alt="onion architecture" style="zoom:50%;" />

  *Figure* 2 Onion architecture ([Vladimir Khorikov][ddd in practice]). The innermost layer is the core domain; the two innermost layers contain domain classes. The onion architecture has good correspondence with *MVVM* (model, view, view model). Model are the innermost two layers; view model belongs to the application services layer; view is the UI layer. View model acts as a wrapper on top of entities; it provides functionality for the view. For an web application, controllers play the role of application services/view models.

- Subdomain

  Separate applications or features your software must support or interact with. It is a problem space concept. It is ideal to have one subdomain solved by one bounded context.

- Bounded context

  A specific responsibility, with explicit boundaries that separate it from other parts of the system. It is a solution space concept and *a boundary for ubiquitous language*; it spans across all layers of the onion architecture. Separate databases per each bounded context.

  "Explicitly define the context within which a model applies… Keep the model strictly consistent within these bounds, but don’t be distracted or confused by issues outside." -- Eric Evans

- Context mapping

  The process of identifying bounded contexts and their relationships to one another

- Shared kernel

  Part of the model that is shared by two or more teams, who agree not to change it without collaboration. It shares code between bounded contexts.

- Anti-corruption layer

  It translates between foreign systems’ models and our own using design patterns, e.g. Façade, Adapter,or custom translation classes or services.

  ![anti corruption layer](Figures/AntiCorruptionLayer.PNG)

  *Figure* Structure of the anti corruption layer  (Evans, 2003)

  <img src="Figures/CommBtwBCs.PNG" style="zoom:50%;" />

  *Figure* Communication between bounded contexts ([Khorikov][ddd in practice]). When two bounded contexts are deployed as individual processes, they are microservices and communicate through RSET API calls or message queues. When two bounded contexts run within a single process, they communicate via the proxy (anti corruption layer) or domain events.

- Domain services

  Domain services contain domain login and possess knowledge that doesn't belong to entities and value objects. They don't have state.

- Application services

  Application services are outside of the domain layer. They communicate with the outside world and don't contain domain logic.

### Entity

It has an ID. The base entity should be an abstract class rather than an interface. Object equality is determined by reference equality and identifier equality. States of entities are due to change through their lifecycle. Therefore, it is good practice to delegate behaviors of entities to value objects as much as possible, since VOs are immutable and easier to work with.

```c#
public abstract class Entity
    {
        public virtual long Id { get; protected set; }

        public override bool Equals(object obj)
        {
            var other = obj as Entity;
            if (ReferenceEquals(other, null))
                return false;
            if (ReferenceEquals(this, other))
                return true;
            if (GetType() != other.GetType())
                return false;
            if (Id == 0 || other.Id == 0)
                return false;

            return Id == other.Id;
        }

        public static bool operator ==(Entity a, Entity b)
        {
            if (ReferenceEquals(a, null) && ReferenceEquals(b, null))
                return true;
            if (ReferenceEquals(a, null) || ReferenceEquals(b, null))
                return false;
            return a.Equals(b);
        }

        public static bool operator !=(Entity a, Entity b)
        {
            return !(a == b);
        }

        public override int GetHashCode()
        {
            return (GetType().ToString() + Id).GetHashCode();
        }
    }
```





### Value object

Immutable, no ID. Object equality is determined by reference equality and structural equality. VOs should belong to entities rather than existing by their own. VOs should not have their own table/collection in the database. The lifetime of a VO should fully depend on its parent entity.

```c#
public abstract class ValueObject<T>
        where T : ValueObject<T>
    {
        public override bool Equals(object obj)
        {
            var valueObject = obj as T;
            if (ReferenceEquals(valueObject, null))
                return false;
            return EqualsCore(valueObject);
        }

        protected abstract bool EqualsCore(T other);

        public override int GetHashCode()
        {
            return GetHashCodeCore();
        }

        protected abstract int GetHashCodeCore();

        public static bool operator ==(ValueObject<T> a, ValueObject<T> b)
        {
            if (ReferenceEquals(a, null) && ReferenceEquals(b, null))
                return true;
            if (ReferenceEquals(a, null) || ReferenceEquals(b, null))
                return false;
            return a.Equals(b);
        }

        public static bool operator !=(ValueObject<T> a, ValueObject<T> b)
        {
            return !(a == b);
        }
    }
```



### Aggregate

An aggregate has a collection of entities. It holds some *invariants* to reside in the valid state. 

- An entity can only belong to a single aggregate.

- An aggregate must have a root entity on which the rest entities depend. The aggregate root is  the only entry point to access the entities within. In other words, foreign classes can only reference the aggregate root instead of any  entity that belongs to the aggregate.
- An aggregate is a single operation unit for the application. For example, it must be returned as a whole for the consuming application service.
- An aggregate is persisted in a transactional manner.

### Domain events

An aggregate root entity creates an event; the infrastructure dispatches the event. Be cautious on when to handle events, pre- vs post- persistence. Relevant concepts: [mediator pattern][mediator pattern], [event sourcing pattern][event sourcing]

The following code snippet from [DDD in practice][ddd in practice] demonstrates handling events via reflection.

```c#
public static class DomainEvents
    {
        private static List<Type> _handlers;

        public static void Init()
        {
            _handlers = Assembly.GetExecutingAssembly()
                .GetTypes()
                .Where(x => x.GetInterfaces().Any(
                    y => y.IsGenericType && y.GetGenericTypeDefinition() == typeof(IHandler<>)))
                .ToList();
        }

        public static void Dispatch(IDomainEvent domainEvent)
        {
            foreach (Type handlerType in _handlers)
            {
                bool canHandleEvent = handlerType.GetInterfaces()
                    .Any(x => x.IsGenericType
                        && x.GetGenericTypeDefinition() == typeof(IHandler<>)
                        && x.GenericTypeArguments[0] == domainEvent.GetType());

                if (canHandleEvent)
                {
                    dynamic handler = Activator.CreateInstance(handlerType);
                    handler.Handle((dynamic)domainEvent);
                }
            }
        }
    }
```



## Design Patterns

Refer to this [catalog](https://refactoring.guru/design-patterns/catalog) and [collection of courses](https://app.pluralsight.com/paths/skills/design-patterns-in-c).

### Factory

#### Factory method pattern

Factory method is a creational design pattern that provides an interface for creating objects in a superclass, but allows subclasses to alter the type of objects that will be created ([refactoring guru][factory method]). The following code snippet by [Fillip Ekberg][factory pattern] demonstrates how to create a base factory that generates various shipping providers . A base factory class defines an abstract method  `CreateShippingProvider` to be implemented by concreate shipping providers; `GetShippingProvider` provides access to the created object.

```c#
public abstract class ShippingProviderFactory
    {
        public abstract ShippingProvider CreateShippingProvider(string country);

        public ShippingProvider GetShippingProvider(string country)
        {
            var provider = CreateShippingProvider(country);

            if (country == "Sweden" && 
                provider.InsuranceOptions.ProviderHasInsurance)
            {
                provider.RequireSignature = false;
            }

            return provider;
        }
    }
```

#### Abstract factory

Abstract Factory is a creational design pattern that lets you produce families of related objects without specifying their concrete classes ([refactorying.guru][abstract factory]). A couple of creational methods are bundled in an abstract factory interface.

#### Implement factory pattern via reflection

This code snippet comes from the [SOLID principle demo code][solid_demo]. It shows how to implement a factory pattern via reflection (`Activator` class in C#) which eliminates the usage of `switch` clause. The `ArdalisRating` namespace has a couple of policy rater classes: `AutoPolicyRater`,`LandPolicyRater`,`LifePolicyRater`, etc. Constructors of the policy rater has one argument, an `ILogger` instance.

```c#
 public class RaterFactory
    {
        private readonly ILogger _logger;

        public RaterFactory(ILogger logger)
        {
            _logger = logger;
        }

        public Rater Create(Policy policy)
        {
            try
            {
                return (Rater)Activator.CreateInstance(
                    Type.GetType($"ArdalisRating.{policy.Type}PolicyRater"),
                        new object[] { _logger });
            }
            catch
            {
                return new UnknownPolicyRater(_logger);
            }
        }
    }
```

The client code uses this factor as follow.

```c#
string policyJson = _policySource.GetPolicyFromSource();
var policy = _policySerializer.GetPolicyFromString(policyJson);
var rater = _raterFactory.Create(policy);
```

Another example from [Phillip Ekberg's repo][factory pattern]. In the constructor, it loops over the assembly and finds all concreate class that implements the `IPurchaseProviderFactory` interface. `CreateFactoryFor` takes in a string and uses an `activator`to create a `PurchaseProviderFactory`.

```c#
public class PurchaseProviderFactoryProvider
    {
        private IEnumerable<Type> factories;

        public PurchaseProviderFactoryProvider()
        {
            factories = Assembly.GetAssembly(typeof(PurchaseProviderFactoryProvider))
                .GetTypes()
                .Where(t => typeof(IPurchaseProviderFactory)
                .IsAssignableFrom(t));
        }
    
        public IPurchaseProviderFactory CreateFactoryFor(string name)
        {
            var factory = factories
                .Single(x => x.Name.ToLowerInvariant()
                .Contains(name.ToLowerInvariant()));

            var instance = 
                (IPurchaseProviderFactory)Activator
                .CreateInstance(factory);

            return instance;
        }
    }
```



### Repository

Repositories let client work operate on data without concerning how data is persisted, as if data were in memory. One repository per each aggreagate.

```c#
public abstract class Repository<T>
        where T : AggregateRoot
    {
        public T GetById(long id)
        {
            using (ISession session = SessionFactory.OpenSession())
            {
                return session.Get<T>(id);
            }
        }

        public void Save(T aggregateRoot)
        {
            using (ISession session = SessionFactory.OpenSession())
            using (ITransaction transaction = session.BeginTransaction())
            {
                session.SaveOrUpdate(aggregateRoot);
                transaction.Commit();
            }
        }
    }
```

### CQRS 

Command Query Responsibility Segregation (CQRS) is a pattern that separates read and update operations for a data store ([Microsoft Docs][cqrs]) .  It solves the asymmetry of read and write workloads. Use commands to update data and queries to read data. Commands mutate the state of entities; they may be placed on a queue for asynchronous processing.

![CQRS](Figures/CQRS.PNG)

*Figure* CQRS ([Microsoft Docs][cqrs])

##### CQRS vs CQS

CQRS ([Greg Young, 2010][greg young]) originates from CQS (command query separation, Bertrand Meyer). In CQS, commands are produce side-effects (change state of the system) and return void. Queries have no side-effect and return non-void. CRQS further separates the read/write models. Commands in CQRS are serializable method calls. <img src="Figures/CQS_vs_CQRS1.PNG" alt="CQS vs CQRS1" style="zoom:50%;" />

<img src="Figures/CQS_vs_CQRS2.PNG" alt="CQS vs CQRS2" style="zoom:50%;" />

*Figure* CQS vs CQRS ([Vladimir Khorikov][cqrs in practice])

The code snippets below ([CQRS in practice][cqrs in practice]) compare commands in CQS and CQRS.

CQS command example, a controller method.

```c#
[HttpPut("{id}")]
public IActionResult EditPersonalInfo(long id, [FromBody] StudentPersonalInfoDto dto)
{
    ...
}
```

CQRS command, a custom class named in imperative tense

```c#
public sealed class EditPersonalInfoCommand: ICommand
{
    public long Id {get; set;}
    public string Name {get; set}
    public string Email {get; set}
}
```

Interfaces for CQRS command and command handler. `Result` is a custom class that has Http-like status code.

```c#
public interface ICommand
{
}

public interface ICommandHandler<TCommand> where TCommand : ICommand
{
    Result Handle(TCommand command);
}
```

##### Commands, Queries, and Events

Commands, queries, and events are all messages which are all part of the core domain.

| Commands                             | Queries                             | Events                       |
| ------------------------------------ | ----------------------------------- | ---------------------------- |
| Tell the application to do something | Ask the application about something | Inform external applications |
| Imperative tense                     | Start with "Get"                    | Past tense                   |
| `EditPersionalInfoCommand`           | `GetListQuery`                      | `PersonalInfoChangedEvent`   |

Another distinction between command and event is that a server can reject a command but not an event.

<img src="Figures/CommandEventInOnion.PNG" style="zoom:50%;" />

*Figure* Command and event in the onion architecture. Command and events are part of the core domain layer, whereas command handlers reside in the application services layer.

## References

1. [C# SOLID Principles][solid_principles_course]
2. [Demo code for SOLID principles][solid_demo]
2. [DDD fundamentals][ddd-fundamental]
2. [Clean architecture][clean-architecture]
2. Evans, E.,  2003. Domain driven design: tackling complexity in the heart of software. Addison-Wesley.
2. [DDD in practice][ddd in practice]
2. [Demo code for factory pattern][factory pattern]
2. [CQRS in practice][cqrs in practice]
2. Young, G., 2010. [CQRS documents][greg young]



[solid_principles_course]: https://app.pluralsight.com/library/courses/csharp-solid-principles/table-of-contents	"C# SOLID Principles"
[solid_demo]: https://github.com/ardalis/SolidSample " SOLID demo"

[ddd-fundamental]: https://app.pluralsight.com/library/courses/fundamentals-domain-driven-design/table-of-contents "DDD fundamentals"
[clean-architecture]: https://github.com/ardalis/CleanArchitecture "Clean architecture"
[ddd in practice]: https://app.pluralsight.com/library/courses/domain-driven-design-in-practice/table-of-contents "DDD in practice"
[cqrs]: https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs "CQRS pattern"
[mediator pattern]: https://refactoring.guru/design-patterns/mediator
[event sourcing pattern]: https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing
[factory pattern]: https://github.com/wyzhang120/c-sharp-design-patterns-factory-abstract "factory pattern"
[factory method]: https://refactoring.guru/design-patterns/factory-method
[abstract factory]: https://refactoring.guru/design-patterns/abstract-factory
[cqrs in practice]: https://app.pluralsight.com/library/courses/cqrs-in-practice/table-of-contents	" CQRS in practice"
[greg young]: https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf
