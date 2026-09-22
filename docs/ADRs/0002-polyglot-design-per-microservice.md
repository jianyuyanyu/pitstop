# ADR-0002: Polyglot Design per Microservice

## Status
Accepted

## Date
2026-04-06

## Context
The solution contains three core domain services: CustomerManagement, VehicleManagement, and WorkshopManagement. Each service owns its data and encapsulates its domain logic, but the complexity and nature of that logic differs significantly across these services.

A microservices architecture ([ADR-0001](0001-microservices-architecture.md)) already grants each service its own deployment unit and data store. That independence is commonly used only for operational isolation, while every service is still built with the same internal design style "for consistency" — a uniform approach to data management and domain modelling applied across all services. Alternatively, each service could be designed fit-for-purpose based on the complexity and characteristics of its own domain, using this independence for internal design as well, not just deployment.

## Decision
Each microservice in Pitstop is designed **fit-for-purpose**, using the architecture and patterns its own domain complexity warrants, rather than a single design style applied uniformly across all services. This is a **polyglot approach to software architecture at the microservice level**: different services may — and here, deliberately do — use different internal architectures.

Concretely: **Event Sourcing and Domain-Driven Design (DDD)** are applied exclusively to `WorkshopManagementAPI`. `CustomerManagementAPI` and `VehicleManagementAPI` use a straightforward CRUD approach with Entity Framework Core.

## Rationale
Microservices architecture's main structural benefit — independent, per-service ownership of code and data — is only half-used if every service is then forced into the same internal design regardless of its domain's actual complexity. Applying a single design style everywhere trades away that benefit for uniformity that doesn't serve the business.

Workshop Management is the core bounded context of the domain — it is where the primary business activity (planning and executing vehicle maintenance) takes place and where the most business rules live. This justifies the investment in a richer design:

- Multiple business rules govern when and how maintenance jobs can be planned (`WorkshopPlanningRules`, `MaintenanceJobRules`).
- The state of a `WorkshopPlanning` aggregate evolves through a meaningful sequence of events (`MaintenanceJobPlanned`, `MaintenanceJobFinished`, etc.) that are worth preserving as first-class facts.
- DDD patterns (aggregates, value objects, domain exceptions) provide the right tools to model and protect this complexity.

Customer and Vehicle Management, by contrast, are supporting contexts with simple lifecycle semantics (register, update). There are no complex business rules, no meaningful state transitions, and no value in storing a history of changes. A CRUD approach with EF Core is the right fit for these services.

This contrast is also intentional from a pedagogical standpoint: the solution demonstrates that within a single microservices system, **different services warrant different design approaches**. There is no one-size-fits-all answer — the design should be fit-for-purpose per service. This principle generalizes beyond this specific pairing: a microservices architecture is meant to let each service be built with the tools that best fit its own context (which can also include polyglot persistence or even polyglot programming languages per service), not merely to have module boundaries without the freedom that boundary is supposed to enable.

## Consequences

### Positive
- Each service is designed at the level of complexity its domain actually requires, avoiding both under-engineering (missing business rule protection) and over-engineering (unnecessary event sourcing in CRUD services).
- WorkshopManagementAPI serves as a concrete example of how to implement DDD, CQRS, and event sourcing in .NET.
- The contrast between services illustrates to developers that architectural patterns are tools to be applied selectively, not universally, and that this choice is made per service rather than decided once for the whole solution.
- New services added to the solution should evaluate their own domain complexity to choose an appropriate design, rather than defaulting to copying an existing service's style.

### Negative
- The solution is less uniform: developers need to understand two distinct design styles when working across services.
- The WorkshopManagementAPI is significantly more complex to understand and extend than the CRUD services, which may steepen the learning curve for that specific service.
- Without this ADR as a guardrail, the inconsistency could be mistaken for accidental drift rather than intentional design, and "fixed" by well-meaning but unwarranted refactoring toward uniformity.

## Alternatives Considered
- **One-size-fits-all internal architecture (event sourcing across all services)**: Would impose unnecessary complexity on CustomerManagement and VehicleManagement, which have no need for event history or complex state transitions. Rejected as over-engineering for those contexts.
- **One-size-fits-all internal architecture (CRUD across all services)**: Would fail to adequately model the business rules and state transitions in WorkshopManagement, and would miss the opportunity to demonstrate advanced DDD/event sourcing patterns. Rejected as under-engineering for that context.
- **Leave the general principle implicit**: Document only the concrete Workshop-vs-CRUD split without stating the underlying polyglot-per-service principle. Rejected because future services would risk being designed without considering that a different style may be more appropriate for them.
