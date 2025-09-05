---
layout: post
title:  "An Overview of Microservices"
date:   2025-09-03 14:31:20 +0200
categories: jekyll update
tags: an overview of microservices
---

## Monolith

![alt text](/assets/an_overview_of_microservices/image_monolith.png)

This is a monolithic server we are familiar with.

  a monolith contains routing, middlewares, business logic and database access to implement all features of our app.


## Microservices

![alt text](/assets/an_overview_of_microservices/image_microservices.png)

a single microservice contains
routing, middleware, business logic and database access to implement **one feature** of our app.



**PROS**: if for some reasons, some services go down, the rest of services still work.

## The big challenge with microservices: Data management between services

	1: we want each service to run independently of other services.

avoid this scenario.

**CONS** if one database shutdowns, all services crush immediately.

![alt text](/assets/an_overview_of_microservices/image_scenario_1.png)


	2:  Database schema/structure might change unexpectedly.

avoid this scenario.

**CONS**:  we introduced dependency between services A et B.

![alt text](/assets/an_overview_of_microservices/image_scenario_2.png)


	3:  Some services might function more efficiently with different types of DB's.


## Practical case

Look at the diagram below, how do we design the database for the Service D

![alt text](/assets/an_overview_of_microservices/image_case_1.png)

There are 2 ways to design
- Sync 
- Async

I am going to show the async way to design the database
The database D has 2 tables , products and user_products. They are created passively. 

![alt text](/assets/an_overview_of_microservices/image_service_D_database.png)

This microservice uses Event Bus architecture.

Here is the workflow
- When a requests comes, it creates an order in the DB for C
- Then it sends an event to the Event-Bus.
- The Event-Bus sends back this event to the DB for service D

![alt text](/assets/an_overview_of_microservices/image_workflow.png)

Note on async communication
- PROS
  - Service D has zero dependencies on other services!
  - Service D will be fast

- CONS
  - Data duplication.
  - Harder to understand.