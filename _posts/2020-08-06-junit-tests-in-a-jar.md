---
layout: post
title: "Ship Junit tests in a Jar"
date: "2019-06-08"
tags: [junit,automation,qa]
---
Do you have Junit tests and you want to ship them in a Jar file so there is no need to set up the source code to run the tests.Or, there is a need of running the tests in a docker container with no access to maven repository.

I recently came across a similar situation where the tests were required to run in a network less container. My test project was using Gradle so I decided to write a small Gradle task to ship the tests in a Jar with all the required resources to run the tests.

```java
task buildJar(type: Jar) {
    dependsOn configurations.runtimeClasspath
    from sourceSets.main.output
    from sourceSets.test.output
    from sourceSets.main.resources
    from sourceSets.test.resources
    from {
        configurations.runtimeClasspath.findAll { it.name.endsWith('jar') }.collect { zipTree(it) }
    }
    exclude 'META-INF/*.RSA'
    exclude 'META-INF/*.SF'
    exclude 'META-INF/*.DSA'
}
```
The above Gradle task will package the main, test, dependencies and resources in a Jar.

#### How to run the tests from jar
Junit provides a console platform launcher to launch the tests from CLI.(https://junit.org/junit5/docs/5.0.0-M5/user-guide/#running-tests-console-launcher)
The jar which we generated above can be input as additional classpath to Junit console launcher and the tests can be executed by writing a simple command as below:

```
java -jar junit-platform-console-standalone-X.X.X.jar -cp the-jar-generated-from-gradle-task.jar -p 'test.package.name.to.run'
```
