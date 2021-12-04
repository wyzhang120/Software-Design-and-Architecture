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
  
  ![SOLID principles](./Figures/SOLID_principles.PNG)
  
  
  
  *Figure* 1 Relationships between SOLID principles  ([Steve Smith][solid_principles_course])
  
  

## Domain Driven Design

“Domain-Driven Design is an approach to software development that centers the development on programming a domain
model that has a rich understanding of the processes and rules of a domain.” -- Martin Fowler

*DDD aims to tackle business complexity, not technical complexity.* Isolate domain logic from other parts of the application.

Refer to the [DDD fundamentals][ddd-fundamental] course for more detailed explanation of the concepts. The [clean architecture][clean-architecture] repo demonstrates how to align the structure of your code with DDD concepts.

### Concepts

- Core domain

  The key differentiator for the customer’s business -- something they must do well and cannot outsource

- Subdomain

  Separate applications or features your software must support or interact with. It is a problem space concept.

- Bounded context

  A specific responsibility, with explicit boundaries that separate it from other parts of the system. It is a solution space concept. Separate databases per bounded context.

  "Explicitly define the context within which a model applies… Keep the model strictly consistent within these bounds, but don’t be distracted or confused by issues outside." -- Eric Evans

- Context mapping

  The process of identifying bounded contexts and their relationships to one another

- Shared kernel

  Part of the model that is shared by two or more teams, who agree not to change it without collaboration. It shares code between bounded contexts.

- Ubiquitous language

  Terminology from a domain model that programmers and domain experts use to discuss that particular sub-system. The ubiquitous language of a bounded context is ubiquitous throughout everything you do in that context – discussion, model, code, etc.

- Anti-corruption layer

### Entity



### Value object



### Aggregate



### Domain events



## Design Patterns

Refer to this [catalog](https://refactoring.guru/design-patterns/catalog) and [collection of courses](https://app.pluralsight.com/paths/skills/design-patterns-in-c).

### Factory

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



## References

1. [C# SOLID Principles][solid_principles_course]
2. [Demo code for SOLID principles][solid_demo]
2. [DDD fundamentals][ddd-fundamental]
2. [Clean architecture][clean-architecture]



[solid_principles_course]: https://app.pluralsight.com/library/courses/csharp-solid-principles/table-of-contents	"C# SOLID Principles"
[solid_demo]: https://github.com/ardalis/SolidSample " SOLID demo"

[ddd-fundamental]: https://app.pluralsight.com/library/courses/fundamentals-domain-driven-design/table-of-contents "DDD fundamentals"
[clean-architecture]: https://github.com/ardalis/CleanArchitecture "Clean architecture"

