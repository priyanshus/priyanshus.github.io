---
layout: post
title: '[Gatling-3] User load models in Gatling'
date: '2020-07-23'
published: true
tags: [gatling,automation,performance-test,qa]
---
One thing which I found fascinating in Gatling is control over the user injection in the test. Gatling has support for two models (Open & Closed) for user injection. The Open model is mainly focused on controlling arrival rate of the users inside the system. The closed model controls concurrency of the users connected to the system.

Lets take few use cases to perform load test:
1. Alice needs to perform load test on a NetBanking application being developed by her team. Alice knows that, the users of NetBanking application will keep on logging in to the system irrespective of the numbers of users already connected to the system.
After a certain period, the system will have enough load to slow down the performance however there are still more users are inlined to log in.

2. Bob works in a company which develops a call distribution software for call centres. The software has main task of distributing the incoming calls among 60 employees of a call centre. If more than 60 calls received at any particular moment, the software keeps additional calls in queue and distributes it only when employees finish their ongoing calls.

**Open Model**

Alice knows that it is hard to control the arrival rate of users to the NetBanking app. To simulate this, Alice needs to inject users growing over a period without bothering about the concurrent users connected to the system. To generate such loads, Alice can choose any of the below user profiles:

a. `rampUsers(5000) during(600 seconds)` - Injects 5000 users in 10 minutes.

b. `atOnceUsers(100)` - Injects 100 users at once.

c. `constantUsersPerSec(60) during (600 seconds)` - Injects 60 users every second for 10 minutes. In this profile 60 users will be added to the system every second at regular intervals.

**Closed Model**

The second use case tells Bob that when system is at full capacity, the calls needs to be in queue to be assigned to someone. Bob knows that 60 is the threshold value for the system and anything beyond 60 will not be distributed until one employee completes the ongoing calls.

To simulate such use case, Bob needs a user profile where first 60 users will be injected as a spike load and next few users will be inlined to get assigned to the system. As soon as 1 user completes his journey, the next inlined user will be assigned to the system to maintain the concurrency of 60 users. Bob has below options to pick the load for his system:

a. `constantConcurrentUsers(60) during(600 seconds)` - Injects 60 concurrent users for 10 minutes. The first 60 users are injected as a spike to the system and 61st, 62nd .. users will be added only when one of the users from first 60 completes the journey.

b. `rampConcurrentUsers(0) to(60) during(10 second)` - Ramp users from 0 to 60 within a second. This can be seen as when call centre opens in the morning the first 60 calls need to be distributed linearly within 10 seconds.

#### Picking the appropriate Model
In general, public websites follow open model of user connection. As in open model, concurrency is a consequence of arrival of the users.

The closed model maintains the concurrency in the system without controlling the arrival rate of the users. The number of requests hitting to the system is not controlled in this model so one should be cautious while using this model.

While picking up the load models, one should be considerable about the system behaviour and its architecture to match the real world load profiles.
