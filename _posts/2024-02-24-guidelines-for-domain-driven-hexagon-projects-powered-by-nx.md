---
layout: post
title: "Guidelines for Domain-Driven Hexagon projects powered by Nx"
author:
  name: "Gaétan Zanella"
  link: "https://github.com/gaetanzanella"
tags: Nx
---

[Domain-Driven Hexagon](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#domain-driven-hexagon) and [Nx](https://github.com/nrwl/nx) are two very famous repositories of the Node world. They are the bones my team chose to support our new NodeJS development stack.

Let's see how we transposed the Domain-Driven Hexagon guidelines in Nx projects in three parts:

- The main concepts of the Domain-Driven Hexagon we selected.
- The rules to apply in order to transpose them in a Nx project.
- Some use cases.

## Context

Recently, a new client asked us to build around 20 cloud components distributed in 4 cloud domains.
A typical cloud domain is composed of:

- 1 API container, to serve domain HTTP clients (apis or front apps).
- 1 database managed by AWS (MySQL or DynamoDB).
- Several data containers, most of the time, we have to import data from reference sources exposed by KDS or S3 into our database.
- Several lambdas, dedicated to small operations of our AWS infrastructure (Modifying/validating tokens for instance).

When the project started, the primary concern of my team was to design a code architecture that would be consistent across all our repositories. If a developer contributes to multiple repositories, she or he should not be lost. Furthermore, our architecture should maximize code reuse and ensure the long term maintainability of our code.

Thanks to prior experiences, [Domain-Driven Hexagon](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#domain-driven-hexagon) & [Nx](https://github.com/nrwl/nx) have quickly been identified as great tools for our quest:

- The scope of a cloud domain should greatly correspond to one monorepo.
- Nx libraries would ensure code reuse between components of the same domain.
- Domain-driven-hexagon principles are well known by our team and the developers of the world.

## From domain-driven-hexagon to 4-layers architecture

The [Domain-Driven Hexagon](https://github.com/Sairyss/domain-driven-hexagon) guidelines are great but a bit verbose.
Even if all the hexagonal principles still applied, we extracted a simple main rule from it:

The code of a component should always be decoupled in 4 layers: **core**, **infra**, **app**, **bootstrap**.

Basically, it means each component codebase should have at least 4 folders (or packages ? answer in the next section :eyes:).
Let's see them in detail.

### Core layer

The **Core layer** contains all the code related to the business part of one or multiple components.

It defines entities and provides a high level API to access the business rules. Most of the time, its public API is only classes called services and entities.

Its main elements are:

- Entities, see [entities](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#entities).
- Services, see [domain services](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#domain-services).
- Commands, they act as inputs for the services methods.
- Ports, see [ports](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#ports).

### Infra layer

The **Infra layer** provides the requirements defined by the core layer. Its main elements are:

- Adapters, the implementations of the ports of Core. Adapters related to databases are usually called "repositories". See [adapters](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#adapters).
- Low level classes, such as "clients" which encapsulate data exchanges or data treatment (**HTTPClient**, **DatabaseClient** **MyThirdPartyServiceClient**, etc).

### App layer

The **App layer** adapts the api provided by **Core layer** to its final usage. In our case, it mainly consists of API and lambdas.

For all, its main elements are:

- Mappers/Parsers, they are used to map DTO to entities/commands exposed by **Core**.

For APIs:

- Controllers, see [NestJS](https://docs.nestjs.com/controllers).
- DTOs, the schemas and types for REST resources.

For lambdas:

- Handler, a functional class that can be triggered.
- DTOs, the schemas and types required to validate the inputs of the lambda.

For simplicity, we banned [Application Services](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#application-services) but they could be considered one day.

### Bootstrap layer

The **Bootstrap layer** contains configurations and the main function of a final component. Its main elements are:

- **main**, the main function, it launches the app using the api provided by **App**.
- **configuration**, the schemas and types required by the configuration of the app.

This layer is close to the App layer. It does not explicitly exists in the [hexagonal paper](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#domain-driven-hexagon) and it can be simply seen as a part of it.
Nevertheless adding this concept helps to move to Nx and its libs / apps distinction. It is the purpose of the next section.

## Transposing 4-layers architecture to a Nx monorepo

Once we clearly divided our components in 4-layers, the next step was to distribute them in Nx monorepos.
Remember, the goal is to maximize code reuse and simplify maintenance.

Quickly, we admitted finding a universal and perfect arrangement for our components will not be possible. Actually, the layers transposition in Nx monorepo may vary from one project to the other.

Instead, alongside detailed use cases (visible in the next section), we defined rules for each of our Nx monorepos:

- **The `apps` folder should be small.**

This is actually part of the mental model [detailed by Nx](https://nx.dev/concepts/more-concepts/applications-and-libraries#mental-model).

In our 4-layers approach, most of the time, the apps folders should only contain the **bootstrap** layer of the final components. Their **core**, **app** and **infra** layers should be in the `libs` folder.

- **The use of independent packages is encouraged.**

Nx makes creating packages easy. It is a great way to share logic across multiple components and packages.

**Independent** means they do not share code with other packages, such as `http-client`, `database-client`, `third-party-client`, `tests-utils` etc.

We place them in a `libs/packages` folder.

- **Respect the strict dependency rules between components layers.**

It is principle elicited in the [clean coder](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html):

> Source code dependencies always point inwards. As you move inwards the level of abstraction increases.

In our 4-layers approach, **Core** is the center circle, it has no dependency:

**Bootstrap** -depends-on-> **App** -depends-on-> **Core** <-depends-on- **Infra**.

For instance, in terms of importation, it means: files in a **App** folder cannot import code from the **Infra** folder.

- **Group business rules in domain packages.**

When two components (api or lambda for instance) have logics close from one to the other, their **core** and **infra** layers can be grouped in a common **domain-${name}** package. In this case, they share the same **Core** api (services and entities) and different **App** layers.

- **Respect the strict boundaries between components and domains.**

Two components can not dependent on each other. It could lead to circular dependencies. For instance:

Files in a **libs/my-app-a** folder cannot import code from the **libs/my-app-b** folder.

Files in a **libs/domain-a** folder cannot import code from the **libs/domain-b** folder.

All logics shared between the layers of two components should be placed in a dedicated **shared** package (**shared/api**, **shared/core** or **shared/infra**).

Let's explore the application of those rules in concrete examples.

## Use cases

### Independent applications

Let's consider a simple API **my-sales** which exposes two resources **users** and **order**.

A hierarchy could be:

```txt
.
├── apps/
│   └── api-my-sales*/
│       ├── main.ts
│       ├── project.json
│       └── ...
└── libs/
    ├── api-my-sales*/
    │   ├── project.json
    │   ├── _my-sales.module.ts_
    │   ├── api/
    │   │   ├── dtos/
    │   │   │   ├── users/
    │   │   │   │   ├── get-users.response.dto.ts
    │   │   │   │   └── get-users.request.dto.ts
    │   │   │   └── orders/
    │   │   │       ├── get-orders.response.dto.ts
    │   │   │       └── get-orders.request.dto.ts
    │   │   └── controllers/
    │   │       ├── users.controller.ts
    │   │       └── orders.controller.ts
    │   ├── core/
    │   │   ├── entities/
    │   │   │   ├── user.entity.ts
    │   │   │   └── order.entity.ts
    │   │   ├── services/
    │   │   │   ├── user/
    │   │   │   │   └── user.service.ts
    │   │   │   └── order/
    │   │   │       └── order.service.ts
    │   │   └── ports/
    │   │       └── ...
    │   └── infra/
    │       └── ...
    └── packages/
        └── my-lib*/
            └── project.json
```

In this case, folders are preferred instead of packages to materialize the 4-layers architecture:

- We simply separate the **bootstrap** layer (in **/apps/api-my-sales**) from **core**, **infra** and **app** (in **/libs/api-my-sales**).
- Nevertheless, as for packages, the dependency rules should be respected. They are just not enforced by explicit package dependencies.

Each layer is split by feature (**core/services/users**, **core/services/orders** etc).

Independent libraries resides in **packages**.

### Sharing business rules between applications

Let's consider a more complex example:

- 1 api **customer-api** which exposes one resource **customer**
- 1 lambda **customer-lambda** which shares the same core & infra layers of **customer-api**
- 1 lambda **order-lambda** independent

In this case, applications layers can be distributed across multiple packages to be shared, such as:

```txt
.
├── apps/
│   ├── *api-customer/
│   │   ├── main.ts
│   │   ├── project.json
│   │   └── ...
│   ├── *lambda-customer/
│   │   ├── main.ts
│   │   ├── project.json
│   │   └── ...
│   └── *lambda-order/
│       ├── main.ts
│       ├── project.json
│       └── ...
└── libs/
    ├── *api-customer/
    │   ├── project.json
    │   ├── my-api.module.ts
    │   └── api/
    │       ├── dtos/
    │       │   └── ...
    │       └── controllers/
    │           └── ...
    ├── *lambda-customer/
    │   ├── project.json
    │   └── lambda/
    │       └── ...
    ├── *lambda-order/
    │   ├── project.json
    │   └── lambda/
    │       └── ...
    ├── domain/
    │   ├── *customer/
    │   │   ├── project.json
    │   │   ├── core/
    │   │   │   └── ...
    │   │   ├── infra
    │   │   └── ...
    │   └── *order/
    │       ├── project.json
    │       ├── core/
    │       │   └── ...
    │       ├── infra
    │       └── ...
    │── shared/
    │   ├── *api/
    │   │   ├── project.json
    │   │   └── ...
    │   ├── *lambda/
    │   │   ├── project.json
    │   │   └── ...
    │   ├── *core/
    │   │   ├── project.json
    │   │   └── ...
    │   └── *infra/
    │       └── project.json
    │           └── ...
    └── packages/
        └── *my-lib/
            ├── project.json
            └── ...

```

The **bootstrap** layer of each package is in **apps** (**/apps/api-customer**, **/apps/lambda-customer**, **/apps/lambda-order**)

The **app** layer of each package has its own **package** (**/libs/api-customer**, **/libs/lambda-customer**, **/libs/lambda-order**).
In fact, **app** depends on the destination. Nest should not be present in a lambda package for instance.

The shared **core** and **infra** layers are grouped to form a "domain":

- **domain/customer** encapsulates the business rules shared by **lambda-customer** and **api-customer**.
- **domain/order** encapsulates the business rules of **lambda-order** which are independent. It could be placed directly in **lambda-order**.

The **shared** packages provide a way to share logics between layers. There is one package per layer.

Independent libraries reside in **packages**.

## What's next?

Hope you liked
