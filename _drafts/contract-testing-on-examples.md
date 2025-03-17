---
title: Contract Testing on Examples
date: 2025-03-17
categories: [Programming, Testing]
---

Many companies are using microservices architecture. It has many advantages, but it also has some disadvantages.
One of them is that the whole system is more complicated, and e2e testing could become not scalable approach. In the case 
of a few services, it is not a problem, as we can start all services and run e2e tests, but having tens or more
services creates a problem, as we need to start all services, and it could take a lot of time, is not reliable, and debugging
failures could be hard. 

Instead of e2e tests, we can test every service separately, and use contract testing to verify the communication 
between services.

## What is contract testing?

![Contract testing](/assets/img/2025-03-24/contract-testing.png)

Basically, in contract testing we have two sides: provider and consumer. 
Provider is a service that provides some API, and consumer is a service that uses this API. In the consumer tests we're
defining the contract, what is the request, fields and what is the response, and then the production code is run
against mocked provider, as the result we have a contract. Then in the provider tests, the same contract is used to
verify if the provider is working correctly, so we send the request defined by the consumer and check if the response
is as defined also in the contract. The type of communication doesn't matter, the tests could be written for 
REST API, gRPC, messaging etc. 

In contract tests we don't test the behavior, only the contract, so we test if the request and response are as expected, 
there are required fields, and types are correct. Everything that can break the consumer 
of the provided API should be verified. However, we don't test the business logic, as it should be tested 
in the unit or integration tests, so we don't test e.g. if something was correctly saved in the database. 

Comparing to e2e tests, contract tests are much faster, they are run only for one service, failures are clear, and they 
are more reliable. 

## Example

Let's say that we have three services: 
- Backend service written in Go that provides REST API to get user details
- Frontend service written in React that uses the backend service to get user details
- Another Backend service written in PHP that uses the go backend service to get user details

### React Frontend


## Breaking change in provider contract

## Using invalid field in consumer

??

-- github repository

## Summary


