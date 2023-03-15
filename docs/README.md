# DevOps Demonstration Notes and Plan

## Demonstration Notes

This repository includes Kubernetes [manifests](../manifests) and
GitHub Actions [workflows](../.github/workflows) which illustrate a
potential DevOps deployment of a MongoDB database and the Mongo Express
application in an Azure Kubernetes Service cluster, using storage volumes
mounted from Azure File Shares for database persistence across
deployments.

The Mongo Express application and MongoDB database are both configured
to run in sets of three replicas for High Availability (HA), with the
database pods contained in a stateful set and bound to persistent
volumes to ensure the data is retained across deployments.

The application deployment workflow starts with a canary deployment
and review step, followed by full deployment or a rollback to the
previous state.

For more details on the configuration of this demonstration, see the
[Development Steps](#development-steps) section below.

## Operational Planning

Any production system should be monitored for failures and unexpected
issues.  A typical planning process begins with establishing an
understanding as to the required performance of the system, often
described as Service Level Agreements (SLAs), which define a set
of metrics by which the services will be measured (Service Level
Indicators, or SLIs) and the thresholds above or below which those
metrics must remain (Service Level Objectives, or SLOs).

An entire ecosystem exists to help define and meet these objectives.
In brief, system adminstrators and DevOps personnel should be alerted
whenever an SLO is breached, and, importantly, when it is in some
danger of doing so, so there is time to investigate and remediate
the problems.

Burn rate alerts are especially valuable for determining whether a
specific SLO is at risk of breaching its available error budget for
a given window of time, and providing a warning signal beforehand.

#### Logging

Applications should be designed to perform standard and error
logging, which may be directed to a log forwarding agent such
as [fluent-bit](https://github.com/fluent/fluent-bit), which can
then deliver them to search and indexing tools such as Splunk
and Azure Monitor Logs.  Apache Kafka may be useful for streaming
large volumes of logs and other data between services.

Most cloud providers will offer a version of these tools, such as
Amazon CloudWatch Logs, where logs can be examined and alerts may
also be generated if defined thresholds are exceeded, for instance,
if an excess of error log messages are detected.

The OpenTelemetry project defines a common set of APIs, tools,
and terminology for capturing logs as well as metrics and traces
from cloud deployments.  Applications can make use of their
libraries and semantic definitions to help format their log
fields according to industry best practices.

#### Server Metrics

Agents running on each server should collect operating system
and application metrics and forward these to an aggregation
service such as DataDog or Azure Monitor Metrics.  Datadog
provides its own agent, which is an expanded version of the
open-source statsd daemon.  The aggregation platform should
offer monitoring and analysis of the collected metrics,
including the ability to calculate minimums, maximums,
averages, and p95 and p99 percentiles, which can be useful
in detecting the worst-case behaviour of the system.

A number of metrics may be collected at the level of the
operating system, for CPU, I/O, memory, and disk usage.
These may include:

- CPU load: a good general indicator of system health,
  and high load may prevent processes from completing
  transactions or connections before they time out.
- Memory usage: excessive usage will result in poor
  performance, and may reveal memory leaks in processes;
  under extreme pressue, the OS will stop processes that
  are consuming too much memory.
- Disk usage: high disk usage can lead to processes
  failing due to lack of available space; high disk I/O
  latency can result in read and write timeouts.
- Network I/O usage: high latency or congested bandwidth
  may both lead to packet loss or retransmission, and
  ultimately to connection failures.

At the application level, typical metrics include:

- Request volume: if the total number of requests received
  exceeds what the application can support, 
- Request size: overly large request data may result in
  application or data storage errors.
- Request duration: excessively long requests may consume
  resources which should be freed for other requests, or
  reveal problems with the application and its dependencies.
- Error rate: if the ratio of server-side errors to
  overall transactions is too high, SLOs may be breached
  and client issues will become apparent.

Application liveness and health checks may also be performed
by the container automation system, as well as by load balancers
and other system components.

Specific types of applications may also have dedicated agents
and monitoring solutions.  Relational databases, for instance,
may be monitored by tools like SolarWinds VividCortex, which
samples SQL query traffic on the database and can provide
insight into whether certain queries are active or not, and
how long they take to execute and how many rows they return.

#### Client Metrics

Several companies such as NewRelic specialize in providing client-side
monitoring through the use of JavaScript libraries included in the
application's Web pages.  Agent code runs in the browser or mobile
application and collects telemetry data to return to monitoring
service, such as page load times, page rendering speeds, and errors
seen by the client.

These are aggregated by the service and allow for monitoring and
alerting if they demonstrate issues as seen by real users, instead of
solely relying on server-side metrics.

#### Alerting

A paging and notification system should be chosen, such as PagerDuty,
or equivalent services from other providers like ServiceNow.  A hierarchy
of on-call personnel may be defined with first-tier and second-tier
responders.  SLO monitors should signal the paging system when an
specified threshold has been exceeded.

#### Tracing

Identifying where an error occured in a complex system with multiple
interdependent services can be made less challenging if tracing data
is available to help identify the call stack between services and
the time and relative duration of each call.

Tracing data emitted by tools such as OpenTelemetry libraries may
be collected and analyzed; LightStep is a major provider of custom
tools for this purpose, for example.

## Development Steps

The Kubernetes manifests and GitHub Actions workflows in this demonstration
assume several initial steps have been taken:

- A "demo" environment is created in the repository's settings and
  is configured with required reviewers, so the application deployment
  workflow will pause for validation of the canary deployment.
  See the screenshot below for an example.
- A set of Azure File Shares has been created, one for each of the
  three persistent volumes.
- An Azure Kubernetes Service cluster is configured.
- An Azure Docker container registry is configured.
- The relevant names, credentials, and settings for the Azure account
  and cluster and registry are stored in the appropriate GitHub
  Actions secrets.
- GitHub Actions secrets are also configured for the application's
  login and password, and its cookie-signing secrets.

<img width="599" alt="Screen Shot 2023-03-14 at 12 37 57 AM" src="https://user-images.githubusercontent.com/28857117/225280536-2691fb66-9499-4952-942e-19817e322e4b.png">

The MongoDB replicas run on three pods in a Kubernetes stateful
set, each with a `/data/db` directory mounted on a persistent
volume backed by an Azure File Share.  The stateful set ensures
that pods will be replaced one at a time, and will maintain the
same volume mount and identifier as the previous pod.  This
permits the application and other database replicas to connect to
the new pod at the same address as the previous one.  The use of
persistent volumes ensures that data will not be lost of a pod
or the entire database set is stopped.

#### Pull Requests

A [series](https://github.com/chrisd8088/mongo-express/pulls?q=is%3Apr) of
pull requests (PRs) were used to construct this demonstration.

Each PR's descriptions and commits contain detailed explanations of
the purpose and functionality of the workflows and manifests.

The corresponding [runs](https://github.com/chrisd8088/mongo-express/actions)
of the new workflows includes dispatched and automatically executed
runs, with an example of an application canary deployment
[run](https://github.com/chrisd8088/mongo-express/actions/runs/4424169471)
where no approval was given to proceed to a full deployment.

A dispatch workflow and manifest for the persistent volumes were
created in a [first PR](https://github.com/chrisd8088/mongo-express/pull/1),
with a successful deployment
[run](https://github.com/chrisd8088/mongo-express/actions/runs/4422690889)
by the dispatch trigger.

<img width="815" alt="Screen Shot 2023-03-15 at 1 29 43 AM" src="https://user-images.githubusercontent.com/28857117/225285283-013c49eb-2484-4825-b238-ef915ca19f45.png">

A second dispatch workflow and manifest for the MongoDB stateful set were
created in [another PR](https://github.com/chrisd8088/mongo-express/pull/2),
with a successful deployment
[run](https://github.com/chrisd8088/mongo-express/actions/runs/4423369118)
by the dispatch trigger.

<img width="821" alt="Screen Shot 2023-03-15 at 1 30 31 AM" src="https://user-images.githubusercontent.com/28857117/225285383-e3053496-e43e-41c6-85cf-1942749ef2a1.png">

A third dispatch workflow and manifest for the application load balancer
service were created in a
[third PR](https://github.com/chrisd8088/mongo-express/pull/3),
with a successful deployment
[run](https://github.com/chrisd8088/mongo-express/actions/runs/4423512462)
by the dispatch trigger.

<img width="664" alt="Screen Shot 2023-03-15 at 3 43 27 AM" src="https://user-images.githubusercontent.com/28857117/225285670-abe3e01a-10f1-4d74-ad9a-abbe8b4d9959.png">

Dispatch events were used for these workflows because regular application
development PRs should not result in the Continuous Deployment pipeline
rebuilding the corresponding services.  In particular, the database
should not be redeployed when a small application change is made; however,
its deployment pipeline is idempotent and can be run repeatedly without
loss of data or connectivity.

The CI/CD pipeline for the Mongo Express application was created in
a [fourth PR](https://github.com/chrisd8088/mongo-express/pull/4),
which adds a workflow triggered on pushes to the default "demo"
Git branch, and a manifest for the application.  This workflow performs
the CI tests provided by the upstream project, then builds a Docker image
and pushes it to the container registry, and then deploys the image to
a canary pod added to the normal set of pods.  The "demo" environment's
required reviewers are then sent a GitHub notification to confirm the
canary deployment is acceptable, and if so, the full deployment is
completed.

<img width="811" alt="Screen Shot 2023-03-15 at 1 29 20 AM" src="https://user-images.githubusercontent.com/28857117/225285999-bb9d71e3-c7ac-4a10-8a06-f9f30e67881b.png">

After confirming the canary is acceptable, a successful
[run](https://github.com/chrisd8088/mongo-express/actions/runs/4424077729)
of the workflow looks like this:

<img width="1410" alt="Screen Shot 2023-03-15 at 12 58 03 AM" src="https://user-images.githubusercontent.com/28857117/225285915-d9329e3d-7df3-4f35-86d9-02bb7a91cc09.png">

However, if the canary were to be deemed unacceptable, as was done
in this example deployment
[run](https://github.com/chrisd8088/mongo-express/actions/runs/4424169471),
the workflow looks like this instead:

<img width="1407" alt="Screen Shot 2023-03-15 at 1 54 55 AM" src="https://user-images.githubusercontent.com/28857117/225286427-eb02096c-94c4-41eb-97e7-61106d873067.png">

Finally, a [fifth PR](https://github.com/chrisd8088/mongo-express/pull/5)
was merged with a small change to the application itself,
to demonstrate how a PR merge triggers the CI/CD workflow, while opening
the PR only runs the existing CI workflow.

While the canary deployment was in place, on average only one request out
of every five was served the new UI view, representing a 20% canary target.

The change made was to suppress the display of the internal MongoDB databases,
so the application's home page changed from:

<img width="340" alt="Screen Shot 2023-03-15 at 1 24 57 AM" src="https://user-images.githubusercontent.com/28857117/225287169-61580c56-163a-4a10-b587-d056240bd78e.png">

to the following:

<img width="408" alt="Screen Shot 2023-03-15 at 1 24 48 AM" src="https://user-images.githubusercontent.com/28857117/225287202-19088220-4240-4b40-b296-eec6cb76d49f.png">

#### Simplifications

Several simplifying assumptions were made for this demonstration, including:

- Only one system node pool is used, whereas for production differently sized
  Kubernetes node pools would be expected for the database and application pods.
- No ingress controller was created, as the application does not have multiple
  URL routes.
- No HTTPS/TLS configuration was created.

#### Disaster Recovery

While this demonstration includes High Availability deployments using
multiple replicas of each database and application pod, with load
balanced over them, Disaster Recovery (DR) is not part of this demonstration.

As a thought experiment, several options are available, with the caveat
that providing synchronous database transactions in a widely distributed
system is extremely challenging (although Google has several systems which
achieve this).  More conventionally, a geo-replicated copy of the database
would be made available for failover purposes, with the expectation that
some data loss would result because the secondary copy will only be
updated asynchronously.

The Azure File Shares used for the demonstration could be configured on
zone- and/or geo-redundant storage instead of merely locally-redundant
storage.  In particular, with geo-redundant storage, asynchronous replication
to the remote regions would ensure the data could survive a regional outage.

The container registry would also need to be geo-replicated, and the
Kubernetes cluster duplicated in the remote region.  Each cluster
should run across multiple zones within each region.

Application and database deployments would target both regions,
ensuring all pods were running the same versions of the software.

The database would be configured in a read-only mode in the remote
region, since it would be receiving replicated data at the filesystem
level.  It might be necessary to run the database against persistent
volumes mounted a read-only mode to ensure, or to not run the database
at all if it is not able to avoid writing to the filesystem.

A global traffic manager would route traffic to the primary region
so long as it was healthy.  In the event of regional outage,
the traffic manager (possibly implemented using a service such as
Akamai's Global Traffic Management) would route requests to the
secondary region.  The database there would be put into read-write
mode, possibly by remounting the volumes in that mode.
