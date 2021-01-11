---
layout: post
title: "CouchBase TestContainer for Integration Testing"
date: "2021-01-11"
tags: [java,testcontainer,automation,qa]
---
Throwaway instance of CouchBase running in Docker container by [TestContainer](https://www.testcontainers.org/).  

```java

DockerImageName couchbaseImage = DockerImageName.parse("couchbase:latest").asCompatibleSubstituteFor("couchbase/server");

// Will setup a TestBucket
BucketDefinition bucketDefinition = new BucketDefinition("TestBucket")
        .withPrimaryIndex(true);

CouchbaseContainer couchbaseContainer = new CouchbaseContainer(couchbaseImage)
        .withCredentials("someuser", "somepassword")
        .withExposedPorts(8091,8092,8093,8094,8095)
        .withBucket(bucketDefinition);

// To expose the ports
couchbaseContainer.setPortBindings(Arrays.asList("8091:8091", "8092:8092", "8093:8093", "8094:8094", "8095:8095"));

couchbaseContainer.start();
```
