---
layout: post
title:  "Connecting to a gRPC Endpoint inside a Docker Container"
date:   2022-10-12 02:29:30 +0100
categories: golang grpc docker api
---

My team (Machine Learning Platform @ Deliveroo) is building a machine learning inference server that exposes gRPC API using KServe's V2 Inference Protocol gRPC API. After implementating for the `Infer` endpoint, I wanted to test it worked by sending a sample request. To test, I spun up the services with Docker Compose and queried the server using the `grpc_cli` tool but recieved an error:

```
> grpc_cli ls localhost:8080 -l
.....
ServerReflectionInfo rpc failed. Error code: 14, message: failed to connect to all addresses, debug info: {"created":"@1665593217.558709000","description":"Failed to pick subchannel","file":"/tmp/grpc-20220715-11323-r9pkki/src/core/ext/filters/client_channel/client_channel.cc","file_line":3261,"referenced_errors":[{"created":"@1665593217.558709000","description":"failed to connect to all addresses","file":"/tmp/grpc-20220715-11323-r9pkki/src/core/lib/transport/error_utils.cc","file_line":167,"grpc_status":14}]}
```

Thanks to Stack Overflow user olafure, turns out that I had set the default gRPC address to localhost, but when running a service in a Docker container that address isn't accessible from outside the container. Instead using `0.0.0.0` or leaving the address empty will allow the service to listen on all the containers addresses. 

https://stackoverflow.com/questions/43911793/cannot-connect-to-go-grpc-server-running-in-local-docker-container