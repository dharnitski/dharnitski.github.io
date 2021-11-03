---
layout: post
title: "Simplifying SQL tests with builder pattern"
categories: 
tags: Go SQL TDD 
excerpt: Proper uint testing using SQL mocks requires lots of boilerplate code to cover all important scenarios. Using Builder pattern we can significantly reduce amount of code and simplify tests maintenance.    
---

## Why so much repetitive code?

Let's check a simple example to see how easily tests setup became complicated to maintain.

Assume we are building API for ECommerce system. API checks current stock for an item and if item available reduces available number of items and creates an order.

We have 2 tables with data for this sample.

Item 

| name     | type | description             |
|----------|------|-------------------------|
| id       | int  | Primary Key             |
| quantity | int  | Item available Quantity |

Order

| name     | type | description             |
|----------|------|-------------------------|
| id       | int  | Primary Key             |
| item_id  | int  | Foreign Key to Item     |
| quantity | int  | Item ordered Quantity   |

SQL statements to create an order.

* Select number of available items
* Start transaction
* Update number of available items
* Create an order
* Commit transaction

Besides implementing happy path we need to take care of all violations from successful execution.
We have one business reason here to not create an order. We checking items count and system should prevent orders creation if number of items is less than we need.

In addition to business reasons system may not create an order if we cannot execute SQL operation. This code uses remote database and we should respect the fact that DB connection can be interrupted at any moment, or database is out of resources, or table is dead locked or we faced one of many other reasons to get an error for operation instead of successful result.

Good software implementation shou