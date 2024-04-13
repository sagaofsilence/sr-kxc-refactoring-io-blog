---
authors: [sagaofsilence]
title: "Apache HttpClient Configuration"
categories: ["Java"]
date: 2024-02-11 00:00:00 +1100
excerpt: "Apache HttpClient Configuration."
image: images/stock/0125-tools-1200x628-branded.jpg
url: apache-http-client-config
---

Configuring Apache HttpClient is essential for tailoring its behavior to meet specific requirements and optimize performance. From setting connection timeouts to defining proxy settings, configuration options allow developers to fine-tune the client's behavior according to the needs of their application. In this section, we will explore various configuration options available in Apache HttpClient, covering aspects such as connection management, request customization, authentication, and error handling. Understanding how to configure the client effectively empowers developers to build robust and efficient HTTP communication within their applications.

## The "Create a HTTP Client with Apache HttpClient" Series

This article is the second part of a series:

1. [Introduction to Apache HttpClient](/create-a-http-client-with-apache-http-client/)
2. [Apache HttpClient Configuration](/apache-http-client-config/)
3. [Classic APIs Offered by Apache HttpClient](/apache-http-client-classic-apis/)
4. [Async APIs Offered by Apache HttpClient](/apache-http-client-async-apis/)
5. [Reactive APIs Offered by Apache HttpClient](/apache-http-client-reactive-apis/)

{{% github "https://github.com/thombergs/code-examples/tree/master/create-a-http-client-wth-apache-http-client" %}}

\
Let us now learn commonly used options to configure Apache HttpClient for web communication.

## HttpClient Client Connection Management
Connection management in Apache HttpClient refers to the management of underlying connections to remote servers. Efficient connection management is crucial for optimizing performance and resource utilization. Apache HttpClient provides various options for configuring connection management.

`PoolingHttpClientConnectionManager` manages a pool of client connections and is able to service connection requests from multiple execution threads. It pools connections as per route basis. It maintains a maximum limit of connections on a per route basis and in total. By default it creates up to 2 concurrent connections per given route and up to 20 connections in total. For real-world applications we cam increase these limits if needed.

This example shows how the connection pool parameters can be adjusted:

```java
  public CloseableHttpClient getPooledCloseableHttpClient(
      String host,
      int port,
      int maxTotalConnections,
      int defaultMaxPerRoute,
      long requestTimeoutMillis,
      long responseTimeoutMillis,
      long connectionKeepAliveMillis) {
    PoolingHttpClientConnectionManager connectionManager =
        new PoolingHttpClientConnectionManager();
    connectionManager.setMaxTotal(maxTotalConnections);
    connectionManager.setDefaultMaxPerRoute(defaultMaxPerRoute);

    Timeout requestTimeout = Timeout.ofMilliseconds(requestTimeoutMillis);
    Timeout responseTimeout = Timeout.ofMilliseconds(responseTimeoutMillis);
    TimeValue connectionKeepAlive 
      = TimeValue.ofMilliseconds(connectionKeepAliveMillis);
    RequestConfig requestConfig =
        RequestConfig.custom()
            .setConnectionRequestTimeout(requestTimeout)
            .setResponseTimeout(responseTimeout)
            .setConnectionKeepAlive(connectionKeepAlive)
            .build();

    HttpHost httpHost = new HttpHost(host, port);
    connectionManager.setMaxPerRoute(new HttpRoute(httpHost), 50);
    return HttpClients.custom()
        .setDefaultRequestConfig(requestConfig)
        .setConnectionManager(connectionManager)
        .build();
  }
```
This code snippet creates a customized `CloseableHttpClient` with specific connection pool properties. It first initializes a `PoolingHttpClientConnectionManager` and configures the maximum total connections and default connections per route. Then, it sets timeout values for connection requests and responses, as well as the duration for keeping connections alive.

The `RequestConfig` is configured with the specified timeout values and connection keep-alive duration. It then creates an `HttpHost` based on the provided host and port.

Finally, it sets the maximum connections per route for the specified host and port, and builds the `CloseableHttpClient` with the custom request configuration and connection manager.

This customized `CloseableHttpClient` is useful for controlling connection pooling behavior, managing timeouts, and optimizing resource utilization in HTTP communication.

### Connection Pooling
Apache HttpClient utilizes connection pooling to reuse existing connections instead of establishing a new connection for each request. This minimizes the overhead of creating and closing connections, resulting in improved performance.

### Max Connections
Developers can specify the maximum number of connections allowed per route or per client. This prevents resource exhaustion and ensures that the client operates within specified limits.

### Connection Timeout
It defines the maximum time allowed for establishing a connection with the server. Setting an appropriate connection timeout prevents the client from waiting indefinitely for a connection to be established.

### Socket Timeout
Socket timeout specifies the maximum time allowed for data transfer between the client and the server. It prevents the client from blocking indefinitely if the server is unresponsive or the network is slow.

### Connection Keep-Alive
Keep-Alive is a mechanism that allows multiple requests to be sent over the same TCP connection, thus reducing the overhead of establishing new connections. Apache HttpClient supports Keep-Alive by default, but developers can configure its behavior according to their requirements.

By understanding and configuring connection management settings, developers can optimize resource utilization, improve performance, and ensure the robustness of their HTTP communication.

## Caching Configuration
The caching HttpClient inherits all configuration options and parameters of the default non-caching implementation (this includes setting options like timeouts and connection pool sizes). For caching specific configuration, you can provide a CacheConfig instance to customize behavior.

First of all, we need to add the maven dependency for Apache HttpClient cache.
```xml
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5-cache</artifactId>
    <version>5.3.1</version>
</dependency>

``` 

Here is an example to build a client supporting cache:

```java
CacheConfig cacheConfig =  CacheConfig.custom()
                                      .setMaxCacheEntries(maxCacheEntries)
                                      .setMaxObjectSize(maxObjectSize)
                                      .build();
CachingHttpClientBuilder builder = CachingHttpClients.custom(); 
CloseableHttpClient client = builder.setCacheConfig(cacheConfig).build();

```

This code snippet first creates a `CacheConfig` object using the `CacheConfig.custom()` builder method. This configures parameters such as the maximum number of cache entries (`maxCacheEntries`) and the maximum size of cached objects (`maxObjectSize`).

Next, it creates a `CachingHttpClientBuilder` and set the `CacheConfig` object to the builder using `builder.setCacheConfig(cacheConfig)`.

Finally, it builds the `CloseableHttpClient` by calling `builder.build()`, which creates an HTTP client with caching enabled according to the specified configuration.

This setup enables caching of HTTP responses, which can improve performance by serving cached responses for repeated requests, reducing the need for repeated network requests and server processing.

## Configuring Request Interceptors
A custom request interceptor in Apache HttpClient allows developers to intercept outgoing HTTP requests before they are sent to the server. Typically, a custom response interceptor implements the `HttpRequestInterceptor` interface and overrides the `process()` method. This interceptor can modify the request headers, request parameters, or even the entire request body based on specific requirements. For example, it can add authentication headers, logging headers, or handle request retries.

Let us implement a request interceptor:
```java
public class CustomHttpRequestInterceptor implements HttpRequestInterceptor {
  @Override
  public void process(HttpRequest request, EntityDetails entity, HttpContext context)
      throws HttpException, IOException {
    request.setHeader("x-request-id", UUID.randomUUID().toString());
    request.setHeader("x-api-key", "secret-key");
  }
}
```

The `CustomHttpRequestInterceptor` implements the `HttpRequestInterceptor` interface provided by Apache HttpClient. This interceptor is designed to intercept outgoing HTTP requests before they are sent to the server. In the `process()` method the interceptor modifies the request object by adding custom headers. In this specific implementation,
 it sets `x-request-id` header to a randomly generated UUID string. We commonly use such header to uniquely identify each request, which can be helpful for tracking and debugging purposes. Then it sets `x-api-key header` to a predefined secret key. This header would be used for authentication or authorization purposes, allowing the server to verify the identity of the client making the request.
 
Overall, this interceptor enhances outgoing HTTP requests by adding custom headers, which can serve various purposes such as request identification, security, or API key authentication.

Now let us build a HTTP client using this request interceptor:
``` java
HttpRequestInterceptor interceptor = new CustomHttpRequestInterceptor();
HttpClientBuilder builder = HttpClients.custom();
CloseableHttpClient client = builder.addRequestInterceptorFirst(interceptor).build();

```

In this code snippet, we first create an instance of `CustomHttpRequestInterceptor` to customize outgoing HTTP requests. Then we build the client using `HttpClientBuilder`.

Then we call `addRequestInterceptorFirst()` method passing the interceptor object as an argument. This method adds the `interceptor` as the first request interceptor in the chain of interceptors. It also has a method `addRequestInterceptorLast()` to add the interceptor at the end.

By adding the interceptor first, it ensures that the custom headers set by the `CustomHttpRequestInterceptor` will be included in all outgoing HTTP requests made by the HttpClient instance. Finally, it calls the `build()` method to create the `CloseableHttpClient` instance with the configured request interceptor.


## Configuring Response Interceptors
A custom response interceptor intercepts and processes HTTP responses received from the server before they are returned to the client. It allows developers to customize and modify the response data based on specific requirements.

Typically, a custom response interceptor implements the `HttpResponseInterceptor` interface and overrides the `process()` method. Inside this method, developers can access and manipulate the HTTP response object, such as modifying headers, inspecting status codes, or extracting content.

Custom response interceptors are useful for tasks like logging responses, handling error conditions, extracting specific information from responses, or performing additional processing before passing the response back to the client code. They provide flexibility and extensibility to tailor the behavior of HTTP responses according to application needs.

Let us implement a response interceptor:
```java
@Slf4j
public class CustomHttpResponseInterceptor implements HttpResponseInterceptor {
  @Override
  public void process(HttpResponse response, EntityDetails entity, HttpContext context)
      throws HttpException, IOException {
    log.debug("Got {} response from server.", response.getCode());
  }
}
```

The `CustomHttpResponseInterceptor` implements the `HttpResponseInterceptor` interface provided by Apache HttpClient. This interceptor is designed to intercept incoming HTTP responses before they are returned to the client. In the `process()` method the interceptor logs the status code of the response object.

Now let us build a HTTP client using this request interceptor:
``` java
HttpResponseInterceptor interceptor = new CustomHttpResponseInterceptor();
HttpClientBuilder builder = HttpClients.custom();
CloseableHttpClient client = builder.addResponseInterceptorFirst(interceptor).build();

```
In this code snippet, we first create an instance of `CustomHttpResponseInterceptor` to handle incoming HTTP responses. Then we build the client using `HttpClientBuilder`.

Then we call `addResponseInterceptorFirst()` method passing the interceptor object as an argument. This method adds the `interceptor` as the first response interceptor in the chain of interceptors. It also has a method `addResponseInterceptorLast()` to add the interceptor at the end.

By adding the interceptor first, it ensures that the response code logged before the response is manipulated further. Finally, it calls the `build()` method to create the `CloseableHttpClient` instance with the configured response interceptor.

## Configuring Execution Interceptors
On the other hand, an execution interceptor allows developers to intercept the execution of HTTP requests and responses. It operates at a lower level compared to request interceptors and can intercept various stages of the request execution process, such as before sending the request, after receiving the response, or when an exception occurs during execution. Execution interceptors can be used for tasks like logging, caching, or error handling. They provide finer-grained control over the HTTP request execution process and enable developers to implement custom logic based on specific requirements or conditions.

{{% info title="ExecChain and Scope" %}}
In Apache HttpClient 5, `ExecChain` and `ExecChain.Scope` play key roles in request execution and interception.

`ExecChain` represents the execution chain for processing HTTP requests and responses. It defines the core method `proceed()`, which is responsible for executing the request and returning the response. This interface allows for interception and modification of the request and response at various stages of execution.

`ExecChain.Scope`, on the other hand, represents the scope within which a request is executed. It provides contextual information about the execution environment, such as the target host and the request configuration. This scope helps in determining the context of request execution, allowing interceptors and handlers to make informed decisions based on the execution context.

{{% /info %}}


Let us implement a execution chain interceptor:
```java
@Slf4j
public class CustomHttpExecutionInterceptor implements ExecChainHandler {
  @Override
  public ClassicHttpResponse execute(
      ClassicHttpRequest classicHttpRequest, 
      ExecChain.Scope scope, 
      ExecChain execChain) throws IOException, HttpException {
    try {
      classicHttpRequest.setHeader("x-request-id", UUID.randomUUID().toString());
      classicHttpRequest.setHeader("x-api-key", "secret-key");

      ClassicHttpResponse response = execChain.proceed(classicHttpRequest, scope);
      log.debug("Got {} response from server.", response.getCode());

      return response;
    } catch (IOException | HttpException ex) {
      String msg = "Failed to execute request.";
      log.error(msg, ex);
      throw new RequestProcessingException(msg, ex);
    }
  }
}

```
The provided code defines a custom HTTP execution interceptor named `CustomHttpExecutionInterceptor`, implementing the `ExecChainHandler` interface. This interceptor intercepts the execution of HTTP requests. 

Within the `execute()` method, the interceptor first sets custom headers (`x-request-id` and `x-api-key`) to the incoming HTTP request.

Next, the interceptor proceeds with the execution of the request by calling `execChain.proceed(classicHttpRequest, scope)`, which delegates the request execution to the next handler in the execution chain.

Upon receiving the response from the server, the interceptor logs the status code of the response. This logging statement provides visibility into the response received from the server.

If an `IOException` or `HttpException` occurs during the execution of the request or response handling, the interceptor catches these exceptions. It logs an error message indicating the failure and wraps the exception in a `RequestProcessingException`, which is then thrown to indicate the failure of the request execution.

Now let us build a HTTP client using this execution interceptor:
``` java
HttpExecutionInterceptor interceptor = new CustomHttpExecutionInterceptor();
HttpClientBuilder builder = HttpClients.custom();
CloseableHttpClient client = null;
client = builder.addExecInterceptorFirst("customExecInterceptor", interceptor).build();

```
In this code snippet, we first create an instance of `CustomHttpExecutionInterceptor` to intercept request execution. Then we build the client using `HttpClientBuilder`.

Then we call `addResponseInterceptorFirst()` method passing the interceptor object as an argument. This method adds the `interceptor` as the first response interceptor in the chain of interceptors. It also has a method `addResponseInterceptorLast()` to add the interceptor at the end and methods `addResponseInterceptorBefore()` and `addResponseInterceptorAfter()` to add the interceptor before and after an existing interceptor respectively.

By adding the interceptor first, it ensures that the interceptor get chance to perform its logic ahead of other interceptors. Finally, it calls the `build()` method to create the `CloseableHttpClient` instance with the configured response interceptor.

## Conclusion
In this part of article series, we explored the configuration aspects of Apache HttpClient, focusing on connection management, caching, and interceptor setup.

Firstly, we learned connection management, discussing the customization of connection pools, timeouts, and keep-alive settings to optimize HTTP request handling and resource utilization.

Next, we examined how to configure caching in Apache HttpClient, enabling the caching of HTTP responses to improve performance and reduce network overhead.

Finally, we explored interceptor configuration, including the implementation of custom request and response interceptors to modify HTTP requests and responses at various stages of execution, providing flexibility for logging, header manipulation, and centralized exception handling.