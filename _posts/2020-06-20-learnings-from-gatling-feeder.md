---
layout: post
title: '[Gatling-2] Learnings from Gatling Feeder'
date: '2020-06-20'
published: true
tags: [gatling,automation,performance-test,qa]
---
[In my previous blog]({{ site.baseurl }}{% link _posts/2020-06-04-gatling-request-response-processor.md %}), I outlined our feeder strategy for Gatling. The feeder had given a responsibility to fetch the test users from Mountebank server as below:

```java
val feeder = Iterator.continually(Map(
    "userId" -> fetchFromMountebank().userId,
    "applicationId" -> fetchFromMountebank().applicationId
  ))
```

A few days later, while looking at our Grafana dashboard we observed that the spikes in Grafana are shorter than the requests triggered from the Gatling test. The Gatling test was supposed to hit the server by 30 users per second but in our Grafana dashboard, it was much shorter.

#### Issue
After careful observation of Gatling logs, we realized that the `fetchFromMountebank()` in feeder gets executed after user injection to the scenario. So If `fetchFromMountebank()` is having execution time greater than 1sec/30 = ~0.033s then the feeder is not capable to generate 30 users in a second. In our case `fetchFromMountebank()` was taking much more than 0.03s.


Execution Log

![]({{site.baseurl}}/assets/img/posts/gatling-logs-for-feeder-execution.png)

In our simulation,**The user
injection profile says hit the server by 30 users in a second but feeder is having some lazy operation and it is not capable of generating 30 users within a second.** It leads to a situation where the server is being hit by 30 users but not in a second. The Gatling report will not tell you this. The report will have the correct count of number of requests and corresponding time taken to serve the response. But in actual, the server never received 30 requests in a second. So, the numbers shown in the report can be misleading.

Above can be validated only by looking into some kind of monitoring tool for your server. In our case, this was revealed by Grafana.

#### Fix
We had to move the `fetchFromMountebank()` from feeder to some other place which does not get executed after user injection by Gatling. The best place for this was `before{}`. Also, We had to switch to CSV based feeder.

We also found that our `processRequestBody()` is having a similar time-consuming operation so we did a check for that as well and decided to make time-taking operation as a cacheable block.

#### Learnings
- Avoid using any piece of code which can impact your user injection profile. Be careful while having heavy operation in `feeders` or `processRequestBody()`. Even object instantiation in such blocks can spoil the user injection profile.
- While simulating the load on your local machine, keep the debug logs enabled. It helps you to understand the gatling code flow and issues like above can be identified easily.
- If possible, validate the load on your server. The load injected by Gatling should be in line with the figures revealed by server logs or monitoring dashboard. Especially cross-check that the users hitting the server in same time window as it is specified in the test.
- Understand the user load profile correctly and use the appropriate setup to inject the load.
For example, `constantUsersPerSec(30)` specifies that Gatling will inject 30 users every second. In this setup, there is a possibility that at some point of time number of users connected to the server may go higher than 30.

Wherein `constantConcurrentUsersPerSec(30)` specifies the number of users connected to the server will always remain same. In this case Gatling will always be connected with the server only by 30 concurrent users.
