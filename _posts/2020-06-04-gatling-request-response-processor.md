---
layout: post
title: '[Gatling-1] Request and Response Processor'
date: '2020-06-04'
published: true
tags: [gatling,automation,performance-test,qa]
---
In my recent assignment, we developed APIs without any authentication.However, The APIs are protected by PGP encryption. The request sent to and response served by it will be signed and encrypted by PGP keys.

There are few more obligations to consume the APIs correctly. The APIs only support POST request and  every journey needs to be started with a few unique identifiers in the request payload. Example, An user Id, Time Stamp, Application Id etc. Also, **if a request has 60s older timestamp then server returns a specific error response**.

Gatling is quite famous in ThoughtWorks and we also decided to have our performance tests in Gatling as it enables to write the tests in source code. For above mentioned challenges like encryption, uniqueness in request we thought these can be achieved easily by writing our own custom processors & transformer in Gatling.


### Test strategy
In order to prepare the payload for request, Gatling provides a few convenient options like feeders, file/string based request body and Request body processors.
Below, we will see how these awesome features of Gatling can be utilised correctly in a simulation.

#### File based request
A request body can be used from a template json(fixture.json) file and it can be directly feeded-in as a payload to API request. As in our case we needed unique user id, timestamp and application id so we marked these objects as `${variableName}`.A sample fixture.json:

```json
{
  "userId" : "${userid}",
  "applicationId": "${applicationId}",
  "epochMillis": 123123123123
}
```


#### Feeders
Feeders can be used to feed in the data in request body. The values marked as `${someValue}` in request payload will be replaced by the feeders automatically.

```java
val feeder = Iterator.continually(Map(
    "userId" -> fetchFromTestEnv().userId,
    "applicationId" -> fetchFromTestEnv().applicationId
  ))
```

We added RestAssured capability in our code to fetch the required identifiers.The above piece of code will get executed for each thread of gatling user and will also update the `${userid}` & `${applicationId}` at run time. The feeders can also talk to some other datasources like CSV file, MySQL DB etc.

**One important point to note, the time taken by feeder to fetch the required data will not be included in actual performance test response.**


#### Request Body processor
Request body processor can be used to have custom code to process the request payload before queuing them for test. We kept our encryption and timestamp logic in body processor.
**One may argue that similar to userId & applicationId, the timestamp logic can also be added in feeder itself. However, the feeders get executed earliest in the life-cycle of Gatling simulation so I preferred to have it later in the cycle to minimise the failures because of 60s check.**

```scala
def bodyProcessor(): Body => ByteArrayBody =
    (body: Body) => {
      // Other case statements need to be added as per request Body used in simulation
      val stream = body match {
        case b: CompositeByteArrayBody => {
          b.asStream.map(content => {
            val util = new PGPUtil()
            // To update the timestamp
            var payload = new String(content.readAllBytes()).replace("123123123123", TimeUtil.epochMilliSeconds().toString)

            // To encrypt
            util.encrypt(payload.getBytes())
          })
        }
      }
      ByteArrayBody(stream)
    }
```

#### Response Transformer
As mentioned above, the response is also encrypted so we needed a mechanism to decrypt it before adding any checks in simulation. Below piece of code will decrypt the response.

```scala
def responseTransformer(response: Response): ByteArrayResponseBody = {
    val processedResponse = new ByteArrayResponseBody(new PgpDecrypter(response.body.stream), UTF_8)
    processedResponse
  }
```

#### A Sample Simulation
By combining all above mechanisms, our simulation looks as below:

```scala
val feeder = Iterator.continually(Map(
    "userId" -> fetchFromTestEnv().userId,
    "applicationId" -> fetchFromTestEnv().applicationId
  ))

  // Http Configuration
  val httpConf = http("Login")
    .post("/login")
    .body(ElFileBody("fixture.json")) // Template based payload
    .processRequestBody((bodyProcessor()) // Body Processor
    .transformResponse { (session, response) =>
      response.copy(body = responseTransformer(response))
    } // Response Transformer
    .check(bodyString.saveAs("response")) // Save response in plain text
    .check(jsonPath("$.some.jsonpath").exists) // Assertion against plain text response


  // Scenario
  val login = scenario("Login")
    .exec(httpConf)

  // User Injection
  setUp(login.inject(constantUsersPerSec(10) during 60).protocols(httpProtocol))
```

To ensure that time taken to process the request does not get added in the simulation result, I added intentional `Thread.sleep(50000)` in feeder and body processors. The simulation result with and without them had similar results which confirm that processing time is not added to actual api response time.Thanks to Gatling to provide such nice features.

I am still exploring for a better approach to have timestamps in request payload as there are still some timestamp related error responses if tests run for longer duration.
