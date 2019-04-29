# Metrics

After reading this document you will understand the following
- How we gather metrics
- Why we don't use the Prometheus Pushgateway
- How we label and name metrics per service types
- How we name and label metrics for clients

## Metrics Gathering

Purpose of this document is to outline how metrics are extracted from our applications / components and sent to prometheus.
I have broken this down into two main cases
- Standard
- Non Standard

### Standard Case

The usual cases is when we have persistent / long running applications running inside kubernetes. This includes typically the following
- Express JS Web Applications
- GraphQL applications
- NodeJS based cron jobs (e.g. consuming SQS Queue)

Each of these apps will expose a metrics endpoint which will be periodically scraped by prometheus.
The way this is done inside kubernetes is through the use of annotations typically done on deployment or pod resources

| Annotation Name | Description  |
| ------------- |-------------|
|  prometheus.io/scrape |  set to "true" to inscruct this application to be scraped by prometheus |
| prometheus.io/port    |  the port of the http server to scrape, typically 3001 for us. It's generally best to put this on a separate express app so its not exposed to the internet |
| prometheus.io/path    |  the path to where to locate metrics on the http server, its generally **/metrics** |


#### Labels

When applications are scraped inside kubernetes by prometheus the following labels are added on

| Label Name | Description  |
| ------------- |-------------|
|  app |  the name of the application (obtained from the standard deployment lables) its typically the system-component |
| chart    |  the name of the helm chart |
| component   |  the name of the component (obtained from the standard deployment lables) |
| system | the name of the microservice e.g. smx-core, pmsx (obtained from the standard deployment lables)|
|kubernetes_namespace| the kubernetes namespace where the application is deployed|
|kubernetes_pod_name| the identifier of the actual kubernetes pod where app is running|
|pod_template_hash| the hash of the pods configuration defintion|

In addition to these, you may see other labels too, in particular if you deploy to pciprod you'll notice the network policy labels also show up 

### Non Standard Cases

There are two main non standard cases
- AWS Lambda
- AWS KCL Multilang Daemon

#### AWS Lambda
These are transient jobs / batch jobs which can exist intermittenly, hence its not possible to have a reliable metrics endpoint exposed to be scraped by prometheus

#### AWS KCL Multilang Daemon
The [AWS KCL Multilang Daemon](https://github.com/awslabs/amazon-kinesis-client#amazon-kcl-support-for-other-languages) is a long running process which runs on kubernetes. 
It is designed to allow one to consume kinesis streams using communication between a parent Java process and potentially multiple child processes running in a different language (for us its NodeJS) over stdout and stdin.
Because the fact that multiple child NodeJS processes will be spun up potentially, each of which will be consuming a shard of the kinesis stream they would all need to bind to the same port to expose a HTTP server to enable the scraping of metrics. Obviously the latter is not possible as only a single process can bind to a port. You could argue that if you only had at most a single child process for each running KCL that would solve the problem, but then you would have another problem to ensure that you have enough consumers to consume all the shards in the stream otherwise you risk lost events.

Now in either case we still would like to get metrics from these running processes into prometheus. The way this is generally done is that you have an external application typically the [Prometheus Pushgateway](https://github.com/prometheus/pushgateway) which these transient / non standard processes push to. This pushgateway is then ultimately scraped by prometheus itself just as described above in the standard case. There is one major draw back to the pushgateway is that it does not perform any aggregation, if a metric with exactly the same labels comes in before the pushgateway has been scraped it will overwrite the previous value. We are almost always certain to have many hundereds if not thousands of metric values collected before they are scraped by prometheus. Thus we will be using [prom aggregation gatway](https://github.com/weaveworks/prom-aggregation-gateway) instead.

For AWS Lambda we would push metrics at the end of each invocation, while in the case of the KCL we could periodically push to ammortise the overhead as its a long running process.

## Naming Metrics

Ideally speaking metric names for generic components should not contain the system name e.g. *pmsx*, rather they should be broken down by logical component / service e.g. http, lambda, kcl. One should then filter to get the metrics for their system by using the system label. By following this pattern we are able to build generic dashboards using PromQL and Grafana which can be heavily reused.

There are exceptions of course, which should be put under a custom namespace, for more information please see the [official docs](https://prometheus.io/docs/practices/naming/)

## Mandatory Labels

I would like all metrics to have at least the following labels
- system
- component
- app
- version

With these combinations of labels you should be easily be able to see from grafana dashboards how positively / negatively software changes has impacted performance.

I will futher break the cases down

### AWS Lambda

Lambda can be triggered from multiple different event sources e.g. SQS, S3 Events and Kinesis.
In the case where we are dealing with kinesis, the *shardId* should be a mandatory label

### AWS KCL Multilang Daemon

All metrics obtained from the KCL must include the *shardId*

### HTTP Server

Must include the following
- user (if not present set to `UNKNOWN`)
- route (always use the raw route e.g. `/users/:userId` instead of the actual path requested e.g. `/users/fekofef` otherwise you risk having a metric per entity which will eventually lead to a metrics explosion)

### GQL Server
Must include the following
- user (if not present set to `UNKNOWN`)
- operationName (must be enforced to be mandatory by the server, otherwise its useless [see here for more info](https://graphql.org/learn/queries/#operation-name) )

## Metric Types

There are four metric types offerred by Prometheus
- Gauge
- Counter
- Summary
- Histogram

You can find more information about these from the [offical docs](https://prometheus.io/docs/concepts/metric_types/)


## Metrics that should be collected by service

### HTTP

Prefix *http_* to the Name column to obtain the name of the metric 


| Name  | Type  | Description  |
|---|---|---|
| request_duration  | histogram  | histogram containing breakdown of duration for all http requests in milliseconds   |
| status  |  counter | counter which has additional label of code which represents the number of times this status code was returned  |
| errors  | counter  | counter which has additional label name, indicating the number of times this error has occurred e.g. SQL Errors (typically in JS / Java you'd get the name of the class) |
| request_body_size  | histogram  |  histogram representing the request body size in bytes |
| response_body_size  |  histogram  | histogram representing the response body size in bytes  |

### GQL

Prefix *gql_* to the Name column to obtain the name of the metric 


| Name  | Type  | Description  |
|---|---|---|
| request_duration  | histogram  | histogram containing breakdown of duration for all gql requests in milliseconds   |
| status  |  counter | counter which has additional label of code which represents the number of times this status code was returned  |
| errors  | counter  | counter which has additional label name, indicating the number of times this error has occurred e.g. SQL Errors |
| request_body_size  | histogram  |  histogram representing the request body size in bytes |
| response_body_size  |  histogram  | histogram representing the response body size in bytes  |


### Lambda

Prefix *lambda_* to the Name column to obtain the name of the metric 


| Name  | Type  | Description  |
|---|---|---|
| duration  | histogram  | histogram containing breakdown of duration for all lambda requests in milliseconds   |
|events_processed| counter | physical number of events that have been processed|
| invocations  | counter  | number of times lambda was invoked   |
| status  |  counter | counter which has additional label of code which represents the number of times this status was returned  |
| errors  | counter  | counter which has additional label name, indicating the number of times this error has occurred e.g. AxiosError (typically in JS / Java you'd get the name of the class) |

In addtion the following are captured if the lambda is sourced from a kinesis stream

| Name  | Type  | Description  |
|---|---|---|
| processing_latency  | histogram  | how long has this message being sitting in the stream (captured with the shardId)  |

Similar for the case where lambda is sourced from SQS

| Name  | Type  | Description  |
|---|---|---|
| processing_latency  | histogram  | indicates how long messages have been sitting in the queue before being processed |
| recieves | gauge | Has a label called count, indicating the number of times a message was received. The reason why I have chosen a guage is that you'll have to decrement the count for the previous time the message was recieved assuming that the current recieve count was greater than 1 and then only then would you increment the current count. Without doing this you would be accumulating events rather than reporting on the actual state |

### AWS KCL Multilang Daemon

Prefix *kcl_* to the Name column to obtain the name of the metric 

| Name  | Type  | Description  |
|---|---|---|
| duration  | histogram  | how long does it take in milliseconds to process the current batch   |
|events_processed| counter | physical number of events that have been processed|
| processing_latency  | histogram  | how long have messages being sitting in the stream (captured with the shardId) before being processed  |
| errors  | counter  | counter which has additional label name, indicating the number of times this error has occurred e.g. AxiosError (typically in JS / Java you'd get the name of the class) |

## Metrics collected by clients

These metrics will allow one to see if there are any network connectivity issues e.g. cannot reach service or if there is unexpected large network latencies

There are two main cases
- HTTP
- GQL

### HTTP

Prefix *client_http_* to the Name column to obtain the name of the metric 

You must include the route and service labels, similar to how its done for server side collection of metrics


| Name  | Type  | Description  |
|---|---|---|
| request_duration  | histogram  | histogram containing breakdown of duration for all http requests in milliseconds   |
| status  |  counter | counter which has additional label of code which represents the number of times this status code was returned  |
| errors  | counter  | counter which has additional label name, indicating the number of times this error has occurred e.g. CONNECTION_TIMEOUT (typically in JS / Java you'd get the name of the class) |

### GQL

Prefix *client_gql_* to the Name column to obtain the name of the metric 

You must include the operationName and service labels, similar to how its done for server side collection of metrics

| Name  | Type  | Description  |
|---|---|---|
| request_duration  | histogram  | histogram containing breakdown of duration for all gql requests in milliseconds   |
| status  |  counter | counter which has additional label of code which represents the number of times this status code was returned  |
| errors  | counter  | counter which has additional label name, indicating the number of times this error has occurred e.g. CONNECTION_TIMEOUT|
