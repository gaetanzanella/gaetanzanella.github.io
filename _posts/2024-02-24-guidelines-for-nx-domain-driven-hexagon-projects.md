---
layout: post
title: "Guidelines for Nx Domain-Driven Hexagon projects"
author:
  name: "Gaétan Zanella"
  link: "https://github.com/gaetanzanella"
tags: Nx
---

[Domain-Driven Hexagon](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#domain-driven-hexagon) and [Nx](https://github.com/nrwl/nx) stand as two pillars within the Node.js ecosystem, widely recognized and adopted.

My team has made a strategic decision to leverage these frameworks as the foundation for our new Node.js development stack.

This journey had three parts:

1. Defining a practical approach of the Domain-Driven Hexagon architecture: the 4-Layers architecture.
2. Writing guidelines for Domain-Driven Hexagon projects powered by Nx.
3. Writing use cases.

Let's dive in each of them and see how combine Nx and Domain-Driven Hexagon principles to build robust applications.

## Context

First, let's start with some context: why would you ever need Nx and clear architecture guidelines?

Six months ago, a new client approached our agency with an ambitious project: the development of around 20 cloud components spread across 4 distinct cloud domains in, of course, a limited timeframe.

A typical cloud domain is composed of:

- 1 API container for serving domain-specific HTTP clients.
- 1 AWS-managed database (MySQL or DynamoDB).
- Data containers facilitating the import of crucial data from external sources like KDS or S3 into our databases.
- Dedicated Lambdas handling various small-scale operations within our AWS infrastructure, such as token manipulation and validation.

To meet these demands, we assembled a team of 15 developers and allocated a minimum of four months for development. As the project started, our primary concern was to establish a cohesive code architecture that would remain consistent across all repositories. It was imperative that any developer contributing to multiple repositories would seamlessly navigate the codebase. Moreover, our architecture had to prioritize code reuse and ensure the long-term maintainability of our codebase.

Thanks to prior experiences, [Domain-Driven Hexagon](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#domain-driven-hexagon) & [Nx](https://github.com/nrwl/nx) have quickly been identified as great tools for our quest.

This is where the need for matching them together and writing clear guidelines about it appeared.

## From domain-driven-hexagon to 4-layers architecture

The [Domain-Driven Hexagon](https://github.com/Sairyss/domain-driven-hexagon) guidelines are great but a bit verbose.
To streamline the process while retaining the essence of hexagonal principles, we've distilled the architecture into a single concise sentence:

Each component's codebase should be structured into four distinct layers: **core**, **infra**, **app**, and **bootstrap**.

Basically, this dictates that every component should contain, at minimum, four folders (or packages - see the last section).
Let's see them in detail.

### Core layer

The **Core layer** contains all the code related to the business part of a component.

It defines entities and provides a high level API to access the business rules. Most of the time, its public API is only classes called services and entities.

Its main elements are classic hexagonal concepts:

- Entities, the Enterprise-wide business rules and attributes, see [entities](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#entities).
- Services, classes used to execute domain logic, see [domain services](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#domain-services).
- Ports, interfaces that abstract technology details, see [ports](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#ports).

Melted together, they define the purpose of a component.

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

This layer is close to the **App layer**. It does not explicitly exist in the [hexagonal paper](https://github.com/Sairyss/domain-driven-hexagon?tab=readme-ov-file#domain-driven-hexagon) and it can be simply seen as a part of it.
Nevertheless adding this concept helps to move to Nx and its libs / apps distinction.
It is the purpose of the next section.

## Transposing 4-layers architecture to a Nx monorepo

Once we clearly divided our components in 4-layers, the next step was to distribute them in Nx monorepos.
Remember, the goal is to maximize code reuse and simplify maintenance.

We quickly realized that achieving a universally perfect arrangement for our components would be impractical. In reality, the transposition of layers within Nx monorepos may vary from one project to another.

Instead, alongside detailed use cases (visible in the next section), we defined rules for each of our Nx monorepos:

1. **The `apps` folder should be small.**

This is actually the mental model [detailed by Nx](https://nx.dev/concepts/more-concepts/applications-and-libraries#mental-model).

In our 4-layers approach, most of the time, the apps folders should only contain the **bootstrap** layer of the final components. Their **core**, **app** and **infra** layers should be in the `libs` folder.

2. **The use of independent packages is encouraged.**

Nx makes creating packages easy. It is a great way to share logic across multiple components and packages.

**Independent** means they do not share code with other packages, such as `http-client`, `database-client`, `third-party-client`, `tests-utils` etc.

We place them in a `libs/packages` folder.

3. **Respect the strict dependency rules between components layers.**

It is principle elicited in the [clean coder](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html):

> Source code dependencies always point inwards. As you move inwards the level of abstraction increases.

In our 4-layers approach, **Core** is the center circle, it has no dependency:

**Bootstrap** -depends-on-> **App** -depends-on-> **Core** <-depends-on- **Infra**.

For instance, in terms of importation, it means: files in a **App** folder cannot import code from the **Infra** folder.

4. **Group business rules in domain packages.**

When two components (api or lambda for instance) have logics close from one to the other, their **core** and **infra** layers can be grouped in a common **domain-${name}** package. In this case, they share the same **Core** api (services and entities) and different **App** layers.

5. **Respect the strict boundaries between components and domains.**

Two components can not dependent on each other as it could lead to circular dependencies. For instance:

Files in a **libs/my-app-a** folder cannot import code from the **libs/my-app-b** folder.

Files in a **libs/domain-a** folder cannot import code from the **libs/domain-b** folder.

All logics shared between the layers of two components should be placed in a dedicated **shared** package (**shared/api**, **shared/core** or **shared/infra**).

Let's explore the application of those rules in concrete examples.

## Use cases

### Independent applications

Let's consider a simple API **my-sales** which exposes two resources **users** and **order**.

A hierarchy could be:

{% highlight txt %}

```
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

{% endhighlight %}

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

{% highlight txt %}

```
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
{% endhighlight %}
```

The **bootstrap** layer of each package is in **apps** (**/apps/api-customer**, **/apps/lambda-customer**, **/apps/lambda-order**)

The **app** layer of each package has its own **package** (**/libs/api-customer**, **/libs/lambda-customer**, **/libs/lambda-order**).
In fact, **app** depends on the destination. Nest should not be present in a lambda package for instance.

The shared **core** and **infra** layers are grouped to form a "domain":

- **domain/customer** encapsulates the business rules shared by **lambda-customer** and **api-customer**.
- **domain/order** encapsulates the business rules of **lambda-order** which are independent. It could be placed directly in **lambda-order**.

The **shared** packages provide a way to share logics between layers. There is one package per layer.

Independent libraries reside in **packages**.

## Wrap up & What's next?

With the hexagonal principles firmly established, the process of transposing them into an Nx project becomes more manageable when adhering to strict guidelines.

The guidelines outlined in this post enabled us to construct and deploy 4 cohesive monorepos, marking one of our greatest successes. Admittedly, adhering strictly to timelines has been a challenge, but isn't that often the nature of projects?
