---
layout: post
title: "What is Unit Test"
categories: 
tags: Testing
excerpt: By default, the Go HTTP Server puts no time limits for execution. It is the developer's responsibility to set appropriate timeouts to prevent connection leaks. This article compares two options to manage timeouts.     
---

There are two schools in 


ANSI/IEEE Std 1008-1987(R2009)
IEEE Standard for Software Unit Testing
Reaffirmed 9 December 2009
Approved 11 December 1986


Approved 18 April 2003
American National Standards Institute


https://github.com/Orthant/IEEE/blob/master/1008-1987.pdf


Software unit testing is a process that includes the performance of test planning, the acquisition of a test set,
and the measurement of a test unit against its requirements.

The unit testing process is composed of three phases that are partitioned into a total of eight basic activities
as follows:
1) Perform the test planning
a) Plan the general approach, resources, and schedule
b) Determine features to be tested
c) Refine the general plan
2) Acquire the test set
a) Design the set of tests
b) Implement the refined plan and design
3) Measure the test unit
a) Execute the test procedures
b) Check for termination
c) Evaluate the test effort and unit

## Definition

test unit: A set of one or more computer program modules together with associated control data, (for example, tables), usage procedures, and operating procedures that satisfy the following conditions:
1) All modules are from a single computer program
2) At least one of the new or changed modules in the set has not completed the unit test
3) The set of modules together with its associated data and procedures are the sole object of a testing
process

======================================================


## systems integration testing

Testing conducted on multiple complete, integrated systems to
evaluate their ability to communicate successfully with each other and to meet the overall integrated
systems’ specified requirements.

3.1.6 component: 

One part that makes up a system. A component may be hardware or software and may
be subdivided into other components. (adopted from IEEE Std 610.12-1990 [B3])

NOTE—The terms “module,” “component,” and “unit” are often used interchangeably or defined to be subelements of
one another in different ways depending on the context. The relationship of these terms is not yet standardized.

3.1.7 component integration testing: Testing of groups of related components. 

3.1.8 component testing: Testing of individual hardware or software components. (adopted from
IEEE Std 610.12-1990 [B3]) 