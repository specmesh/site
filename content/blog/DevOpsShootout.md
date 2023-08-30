---
title: "Kafka DevOps tooling shootout - JulieOps, Jikou & SpecMesh OS, September 2023"
date: 2023-08-25T12:00:00+00:00
featureImage: images/blog/devops-mesh-thing-banner.jpg
tags: ["Educational", "Blog"]
---
_-- Neil Avery (Ex-Confluent)_

## Why are we here

There are a couple of different approaches to provisioning Kafka resources (topics, schemas, ACLs etc). We can use conventional DevOps tools like CloudFormation, Terraform, or Ansible. These tools exist in every organization and are generally used for provisioning server infrastructure and firewalls. All of them are built upon foundational Infrastructure as Code principles. However, they can also configure Kafka application resources.


## DevOps v GitOps

A subset of DevOps tooling is GitOps. The difference between GitOps and other tools is that GitOps integrates with Git-related workflows and developer cycles that use Git as the source of truth, whereas DevOps processes manage their own `source of truth` and statefulness.  State management is needed to manage the current known enforced state into pre-declarative state - this caters for configuration drift and generally needs something like a ‘DryRun’ to show the drift and explain the plan.

That being said, GitOps tooling is naturally regarded as more developer-centric – developers use Git on a day-to-day basis don’t they?
When building Apache Kafka applications, we also have a division between server resources and infrastructure and application or data-centric resources. For example, DevOps teams own and provision clusters of Kafka brokers, whereas developers own the configuration and lifecycle of Kafka's data resources, such as topics, schemas, and ACLs.

## Well known tools

Popular Kafka GitOps tools include:
- JulieOps - Pourbon - ExConfluent - unmaintained
- KafkaGitOps - https://github.com/devshawn/kafka-gitops - Kafka consultant: unmaintained
- Jikkou - https://github.com/streamthoughts/jikkou - by Florian Hussonnois : active
- SpecMesh - specmesh.io - active

In this post, I'm going to compare JulieOps (due to its broad adoption), Jikkou (because it is actively maintained and created by Florian), and SpecMesh—because, after all, this is the tool I'm shamelessly promoting.

## GitOps tools characteristics

### Jikkou
**Jikkou** (jikkō / 実行) is an open-source tool designed to provide an easier way to manage the configurations on your Data Stream platform.

Florian describes “_Jikkou is a CLI and library for Kafka resources, it uses a Resources file to capture the state, which when combined with the CLI, GitOps workflow of your application can be used to provision Topics and other resources accordingly._”

Features Jikkou provides:
- Topic Creation and Configuration: Jikkou allows you to easily create new topics, update their configurations and then delete them if needed. It also provides the capability to export some topic configurations to eventually apply them on a different Kafka cluster.
- Client Quota Limits: Jikkou allows you to configure Quota limits for authenticated principals and clients.
- Authorization and Access Control List: Jikkou allows you to configure access control lists (ACLs) effortlessly, ensuring that only authorized users or applications can read from or write to specific topics.


“_Note: Jikkou adopts the same resource model as Kubernetes to describe entities. Considering that an increasing number of developers are becoming well-familiar with this model, Jikkou attempts to provide developers with an intuitive approach to managing Kafka entities._”

An example resource follows:


```yaml
# kafka-topics.yaml
---
apiVersion: "kafka.jikkou.io/v1beta2"
kind: KafkaTopic
metadata:
name: 'topic-using-jikkou'
spec:
partitions: 6
replicas: 3
configs:
min.insync.replicas: 2
cleanup.policy: 'delete'
```

Run it: `> jikkou apply -f kafka-topics.yaml`

```json lines
Output:
TASK [ADD] Add topic 'topic-using-jikkou' (partitions=6, replicas=3, configs=[cleanup.policy=delete,min.insync.replicas=2]) - CHANGED
{
"changed" : true,
"end" : 1684181768701,
"resource" : {
"name" : "topic-using-jikkou",
"partitions" : {
"after" : 6
},
"replicas" : {
"after" : 3
},
"configs" : {
"cleanup.policy" : {
"after" : "delete",
"operation" : "ADD"
},
"min.insync.replicas" : {
"after" : "2",
"operation" : "ADD"
}
},
"operation" : "ADD"
},
"failed" : false,
"status" : "CHANGED"
}
EXECUTION in 1s 627ms
ok : 0, created : 1, altered : 0, deleted : 0 failed : 0
```

The resource file could be altered and a new property added, and the cli re-run the apply the change.

To Delete a topic the user would execute:

`>jikkou delete --file kafka-topics.yaml`

Note -- A templating approach is supported where common configuration values are captured in a common file.

```yaml
# file: ./kafka-topics.tpl
apiVersion: 'kafka.jikkou.io/v1beta2'
kind: 'KafkaTopicList'
items:
{% for country in values.countryCodes %}
- metadata:
name: "{{ values.topicPrefix}}-iot-events-{{ country }}"
spec:
partitions: {{ values.topicConfigs.partitions }}
replicas: {{ values.topicConfigs.replicas }}
configMapRefs:
- TopicConfig
{% endfor %}
---
apiVersion: "core.jikkou.io/v1beta2"
kind: "ConfigMap"
metadata:
name: TopicConfig
template:
values:
default_min_insync_replicas: "{{ values.topicConfigs.replicas | default(3, true) | int | add(-1) }}"
data:
retention.ms: 3600000
max.message.bytes: 20971520
min.insync.replicas: '{% raw %}{{ values.default_min_insync_replicas }}{% endraw %}'
```


Jikkou is lightweight, actively maintained and looks like a great tool if you want to just get stuff done. It is very flexible, and can also be hooked into a Git Workflow for GitOps.

You can dive into Jikkou in this article.

https://medium.com/@fhussonnois/why-is-managing-kafka-topics-still-such-a-pain-introducing-jikkou-4ee9d5df948


## JulieOps
_An operational manager for Apache Kafka (Automation, GitOps, SelfService)_


- https://github.com/kafka-ops/julie
- https://julieops.readthedocs.io/en/3.x/

JulieOps has captured a lot of attention and I've seen it used on many projects - including some very large projects. One of the reasons for its adoption is that it drives both Confluent Platform and Confluent Cloud seamlessly - whereas Confluents CFK only works with Confluent Cloud.


Unfortunately, JulieOps is no longer actively maintained, but let's take a look. 


Features:
- Support for multiple access control mechanisms:
  - Traditional ACLs
  - Role Based Access Control as provided by Confluent
- Automatically set access control rules for:
  - Kafka Consumers
  - Kafka Producers
  - Kafka Connect
  - Kafka Streams applications ( microservices )
  - KSQL applications
  - Schema Registry instances
  - Confluent Control Center
  - KSQL server instances
- Manage topic naming with a topic name convention
  - Including the definition of projects, teams, datatypes and for sure the topic name
  - Some of the topics are flexible and defined by user requirements
- Allow for creation, deletion and update of:
  - topics, following the topic naming convention
  - Topic configuration, variables like retention, segment size, etc
  - Acls, or RBAC rules
  - Service Accounts (Experimental feature only available for now in Confluent Cloud)
- Manage your cluster schemas.
  - Support for Confluent Schema Registry

To configure JulieOps you use a ‘topology’ file.

```yaml
context: "context"
source: "source"
projects:
- name: "foo"
  consumers:
  - principal: "User:app0"
  - principal: "User:app1"
  streams:
  - principal: "User:App0"
    topics:
      read:
      - "topicA"
      - "topicB"
      write:
      - "topicC"
      - "topicD"
  connectors:
  - principal: "User:Connect1"
    topics:
      read:
      - "topicA"
      - "topicB"
  - principal: "User:Connect2"
    topics:
      write:
      - "topicC"
      - "topicD"
  topics:
  - name: "foo" # topicName: context.source.foo.foo
    config:
      replication.factor: "2"
      num.partitions: "3"
  - name: "bar" # topicName: context.source.foo.bar
    config:
      replication.factor: "2"
      num.partitions: "3"
- name: "bar"
  topics:
  - name: "bar" # topicName: context.source.bar.bar
    config:
      replication.factor: "2"
      num.partitions: "3"
```

Then run it as follows (using the –topology flag)

```text
% docker run purbon/kafka-topology-builder:latest julie-ops-cli.sh  --help
Parsing failed cause of Missing required options: topology, brokers, clientConfig
usage: cli
    --brokers <arg>                  The Apache Kafka server(s) to connect
                                     to.
    --clientConfig <arg>             The client configuration file.
    --dryRun                         Print the execution plan without
                                     altering anything.
    --help                           Prints usage information.
    --overridingClientConfig <arg>   The overriding AdminClient
                                     configuration file.
    --plans <arg>                    File describing the predefined plans
    --quiet                          Print minimum status update
    --topology <arg>                 Topology config file.
    --validate                       Only run configured validations in
                                     your topology
```

It features the ability to dryRun and also makes sense to tie it in with a GitWorkflow.

_“Julie manages a .cluster-state file, similarly as terraform manages a .tfstate file, which is read and written at the working directory when the Julie command is launched. For this to work properly, we must have the topology.state.topics.cluster.enabled and topology.state.cluster.enabled parameters set to false, otherwise Julie will not check the .cluster-state file and the changes will not be saved“_

### BUT...

One thing that strikes me as a concern - is that it manages its own state. To me - this is not GitOps - Git should be the master of the state and not a `.cluster-state` file on a local directory. This state file means the execution cannot be distributed to serverless processes unless the state file is also made available via S3? - whereas with GitOps - that state is captured in Git. The output of the execution is the CDC and is available via Workflow logs.

JulieOps is widely adopted, however has a few features that go beyond Kafka topic resource modelling (which IMO is the limit and we start venturing into DevOps tools). This includes
- Roles - should be built into the Git Workflow - not into the provision tool config.
- Connect Clusters - these are DevOps responsibilities  - config is debatable
- KSQLDB - client side processing

There is a blurred line between your topics and those processes (clients) that interact with them. Connectors and KSQLDB are both client-side processing runtimes. JulieOps lets you provision your KSQLDB queries using:


```yaml
context: "context"
company: "company"
env: "env"
source: "source"
projects:
  - name: "projectA"
    ksql:
      artefacts:
          streams:
              - path: "ksql-streams/riderlocations.sql"
                name: "riderLocations"
          tables:
              - path: "ksql-tables/users.sql"
                name: "users"
      access_control:
          - principal: "User:ksql0"
            topics:
              read:
                  - "topicA"
              write:
                  - "topicC"
    topics:
      - name: "foo"
      - name: "bar"
        dataType: "avro"
```

I'm not entirely convinced that this is the right approach - IMO KSQL-DB applications should have their own developer CI-CD pipeline that tests, validates and provisions the SQL statements. While the topics should be controlled via provisioning tools - SQL queries are part of the application processing logic. We start to branch out from the ‘data model’/‘data product’ space into the client processing space. If we need to publish the results of the query as a data product, then the published result should be the data-stream, to which, access is controlled/granted.

IMO there should be clean separation between the data-model (topic source modelling by developers), data resources (topic name and config), data infrastructure (connect clusters, brokers) and security (roles). Many of these can be included in different build pipelines and accessed by others.

I guess in truth, it's essential to clearly define product scope. What is the goal of the technology and how cleanly can we put it in a `box`. For example, I have worked in some companies that use disparate languages (Ruby, Golang, Kotlin) and technologies (snowflake, elastic) with Apache Kafka. In that situation, uniform provisioning of data resources/models/topics is essential - it also means that client-side processing is excluded. The onus falls upon the team to own, test and run their KSQL, Kafka Streams etc. As soon as a tool gets to the client side the audience changes - it becomes more ‘developer’ and less ‘data-centric`. Developers require CICD test pipelines to validate, prevent regression and to evolve their microservices, and monoliths in their language of choice. It's a lot to handle!

I did a quick Q&A with one of my colleagues:

_“JulieOps appears to have been born out of developer frustration, but it has since evolved into a Swiss Army knife over time. There is no distinct boundary between developer functions and operational scope, as it covers aspects like connectors, ACLs, and more.”_

And another chat Sion from OSO DevOps (who has used it in anger)

_“State management should be something transparent to the developer, in a mechanism which is universally understood. Git is perfect for this and understood by developers and operations teams - using PullRequests, Workflows and Git for state management - it doesn't make sense for GitOps tools to include ‘own’ state management.”_  – Sion, CTO at OSO.sh 


## SpecMesh
_Enterprise Apache Kafka using AsyncAPI specs to build data mesh with GitOps_

_“SpecMesh is an opinionated modelling layer over Apache Kafka resources that combines GitOps, AsyncAPI (modelling), a parser, testing, provisioning tools as well as chargeback support. Utilizing this methodology and toolset enables organizations to adopt Kafka at scale while incorporating simplification guardrails to prevent many typical mistakes.”_

Being GitOps oriented means, that like Jikkou and JulieOps, SpecMesh can be run from the Command line or as part of a GitOps workflow (that promotes it through various environments). Note, unlike JulieOps, SpecMesh focuses on the data model - roles and clusters are a separate concern (clusters configured and run by cluster managers, roles by security tools and accessed by the pipeline)

A sample SpecMesh Async API spec


```yaml
asyncapi: '2.5.0'
id: 'urn:acme:lifestyle:onboarding'
info:
  title: ACME Lifestyle Onboarding
  version: '1.0.0'
  description: |
    The ACME lifestyle onboarding app that allows stuff - see this url for more detail.. etc

channels:
  _public/user_signed_up:
    bindings:
      kafka:
        partitions: 3
        replicas: 1
        configs:
          cleanup.policy: delete
          retention.ms: 999000

    publish:
      message:
        payload:
          $ref: "/schema/simple.schema_demo._public.user_signed_up.avsc"
```

Command line:
>% docker run --rm --network confluent -v "$(pwd)/resources:/app" ghcr.io/specmesh/specmesh-build-cli provision -bs kafka:9092 -sr http://schema-registry:8081 -spec /app/simple_schema_demo-api.yaml -schemaPath /app
>

Executing the `provision` command creates topics, publish schemas, and creates ACLs based upon `public, private, protected` keywords. This ACL methodology and the use of AsyncAPIs is something seen at many customer sites.

One thing that SpecMesh supports, and probably the most challenging question of any enterprise installation, is the enablement of chargeback. I've seen countless Slack conversations on Confluent PS, various internal docs, and presentations that hook into JMX metrics, etc. It's a complete mess. SpecMesh's enforcement of principles like `ids` and domain owners, consumer groups, etc., enables a strategy for extracting metrics and enables the implementation of a chargeback solution.


## Finally
All three tools have their place; however, like any organization, you need to settle on a consolidated tool stack and adopt a simple, clearly defined strategy. For example, treat security, clustering, and data modelling as separate concerns.

I would summarise as follows:
- Jikkou: lightweight, actively maintained and great for CLI work and GitOps integration.
- JulieOps: Broad adoption but should be looking to migrate to other tools due to the project being mothballed
- SpecMesh: For those on a Data Mesh journey who want to enable AsyncAPI specs and GitOPs to enable uniformity.









