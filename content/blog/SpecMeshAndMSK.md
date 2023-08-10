---
title: "SpecMesh OS - Governance with AWS MSK, August 2023"
date: 2023-08-05T12:00:00+00:00
featureImage: images/blog/msk-cloud-mesh-1.jpeg
tags: ["Educational", "Blog"]
---
_-- Neil Avery (Ex-Confluent)_

## Working with MSK
_A walkthrough showing how SpecMesh hides topic provisioning and messy Kafka ACL configuration to create an effortless experience_


Confluent Cloud and AWS MSK are the two main players in the world of Cloud-based Kafka. While MSK is introducing features like tiered storage, connectors, and making it more AWS-native (Lambdas), the IAM-based governance supported by MSK is often considered clunky and cumbersome. Confluent's RBaC also has its limitations. However, I'm going to show you how SpecMesh changes all of this. It is designed to work on any Apache Kafka runtime that supports the [SimpleAclAuthorizer](https://github.com/a0x8o/kafka/blob/master/core/src/main/scala/kafka/security/auth/SimpleAclAuthorizer.scala).

**Technical TLDR;**

Run a cluster; associate user/secrets/principles in the Secrets Manager with the cluster; use these user/secret pairs in the client connections. SpecMesh will create the ACLs to ensure Access control works and you have self-service governance!

## A really short Kafka security primer

The importance of governance:
- Control access to your company data
- Rich control mechanisms for Read/Write/Configure access of topic data
- Self-governance mechanisms are needed - scale the organisation, don't create central team congestion
- Should be simple; most ACL/Governance is anything but simple 

<br>

Kafka Security is multi-faceted:
- Authorization -- who? - aka the `principle` or `user`
- Access -- what? resource and how are ‘they’ trying to access it)
- Wire encryption
- Encryption at rest

<br>


## Whats the problem?

Kafka governance is painful and complicated and good tools don't really exist. CFLT wrap a set of predefined roles into what they call [RBaC](https://docs.confluent.io/platform/current/security/rbac/index.html) - but it's nothing as comprehensive as [AWS IAM](https://docs.aws.amazon.com/msk/latest/developerguide/iam-access-control.html). Unfortunately, IAM, while being incredibly powerful and technically superior - is also incredibly finicky and challenging to setup and get right. 


**MSK IAM**
```yaml
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:Connect",
                "kafka-cluster:AlterCluster",
                "kafka-cluster:DescribeCluster"
            ],
            "Resource": [
                "arn:aws:kafka:region:Account-ID:cluster/MSKTutorialCluster/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:*Topic*",
                "kafka-cluster:WriteData",
                "kafka-cluster:ReadData"
            ],
            "Resource": [
                "arn:aws:kafka:region:Account-ID:topic/MSKTutorialCluster/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:AlterGroup",
                "kafka-cluster:DescribeGroup"
            ],
            "Resource": [
                "arn:aws:kafka:region:Account-ID:group/MSKTutorialCluster/*"
            ]
        }
    ]
}
```

Confluent RBaC API
https://docs.confluent.io/platform/current/security/rbac/rbac-config-using-rest-api.html
```yaml
confluent iam rbac role-binding create \
--principal User:<my-user-name> \
--role SystemAdmin \
--kafka-cluster-id <kafka-cluster-id>
```



Wouldn't it be nice if governance had something as simple as a shared file system to configure - say
>$ chmod ugo+RWX /usr/myapp/stuff 
 
SpecMesh achieves this via the use of structured topics that are part of the AsyncAPI spec for your app. It is a collection of structured topics that are modelled in the spec. The template goes
```xml
<domain-id><public|private|protected><topic-name>
```   

A spec looks like this:
```yaml
id: 'urn:acme.lifestyle.onboarding'
channels:
  _public.user_signed_up:
    publish:
      message:
        payload:
          $ref: "/schema/simple.schema_demo._public.user_signed_up.avsc"```
```

## Opinions matter

Due to Kafka's lack of opinionated topic structures, org's will choose something that works for them, or often times nothing at all -- flat topic structures are the norm. This also means that its unlikely that there is no obvious relationship between ACLs and topics - that relationship might be stored in code or scripts. SpecMesh forces the use of structure - even without SpecMesh this is something that many orgs already do! Hint: its common to `public`, `private` keywords to help ACL writing and formulation.

# MSK security options for Authentication and Authorization

AWS MSK supported security mechanisms include:
1. mTLS: https://catalog.workshops.aws/msk-labs/en-US/securityencryption/tlsmauth
1. SASL/SCRAM: (authentication) https://catalog.workshops.aws/msk-labs/en-US/securityencryption/saslscram
1. IAM: https://catalog.workshops.aws/msk-labs/en-US/securityencryption/iam

<br>

SpecMesh works with vendor agnostic SASL/SCRAM authentication (cross-vendor, multi-language) _ACLs (access control) - but not IAM (authentication and access control).


## MSK has a few limitations

1. AWS want you to use IAM - however this only works with Java clients using their jars
1. Non-Java clients (RUST, Go) will need to use SASL/SCRAM with MSK - IAM wont work without building your own IAM integration. 
1. IAM isnt that easy to setup and/or automate
1. Broker property: super.users parameter is not supported on MSK (this property bypasses all ACL Checks - see https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/security/authorizer/AclAuthorizer.scala). A workaround is to use multi-authentication (below)

<br>


## MSK Multi-Authentication - SASL/SCRAM + IAM

When multi-authentication is enabled on MSK cluster authorization depends on which of those access control methods a client is using to access MSK cluster.

Let's consider the following example: both IAM and SASL/SCRAM are enabled, and 'client A' is accessing the MSK cluster via IAM authentication, while 'client B' is accessing the cluster via SASL/SCRAM. In this scenario, you can still invoke Apache Kafka ACL APIs and add ACLs for an MSK cluster that uses IAM access control. However, ACLs stored in Apache ZooKeeper will have no effect on authorization for IAM roles. Therefore, access and authorization for 'client A,' which uses IAM auth, will be controlled by the IAM policy alone, as the added ACLs do not affect 'client A' in any way, even though they are added.

On the other hand, when a client is using non-IAM authentication, the added ACLs (including the "allow.everyone.if.no.acl.found" setting) will have an effect. In this case, authorization will be controlled by ACLs. So, when 'client B,' which uses SASL/SCRAM, attempts to perform any operations, it will be validated against the ACLs that were added.

In short, to fill in the gaps in the above table:

Authn & Authz mech | Kafka client authn | Kafka client authz | Kafka ACL behaviour | Property allow.everyone.if.no.acl.found
 --- | --- | --- | --- | --- | --- |
SASL/IAM clients | SASL/IAM | IAM | No effect | No effect |
SASL/SCRAM clients | SASL/SCRAM | ACLs | Applies/Does have an effect | Applies/Does have an effect |

## Follow on using a provided Spec

Ideally, you can follow these steps with your own Spec. Grab some from the [SpecMesh ApacheKafka demo repository](https://github.com/specmesh/getting-started-apachekafka) and create your own repo.

```yaml
asyncapi: '2.5.0'
id: 'urn:acme.simple_range.life_enhancer'
info:
  title: ACME Life Enhancer
  version: '1.0.0'
  description: |
    ACMEs Life enhancer records and predicts how ones life will change due to many events that are experienced - see http://acme.org/life_range for more info
  license:
    name: Apache 2.0
    url: 'https://www.apache.org/licenses/LICENSE-2.0'
servers:
  test:
    url: test.mykafkacluster.org:8092
    protocol: kafka-secure
    description: Test broker

channels:
  _public.user_signed_up:
    bindings:
      kafka:
        envs:
          - staging
          - prod
        partitions: 3
        replicas: 1
        configs:
          cleanup.policy: delete
          retention.ms: 999000

    publish:
      summary: Inform about signup
      operationId: onSignup
      message:
        bindings:
          kafka:
            schemaIdLocation: "payload"
        schemaFormat: "application/vnd.apache.avro+json;version=1.9.0"
        contentType: "application/octet-stream"
        payload:
          $ref: "/schema/acme.simple_range.life_enhancer._public.user_signed_up.avsc"
```
Source: https://github.com/specmesh/getting-started-apachekafka/blob/main/resources/acme_simple_range_life_enhancer-api.yaml



## Requirements
1. Admin access to your AWS console (ability to create MSK cluster, start an instance, configure IAM, roles, start instances, configure secrets etc)
1. Basic understanding of Kafka broker (broker properties), and client eco-system (producer, consumer, client.properties)

<br>

SASL/SCRAM on MSK has the following limitations:
1. `super.users` parameter is not supported
1. MSK only supports SCRAM-SHA-512 authentication
1. An MSK cluster can have up to 1000 users
1. You must use an AWS KMS key with your Secret
1. You cannot use a Secret that uses the default Secrets Manager encryption key with Amazon MSK
1. You can't use an asymmetric KMS key with Secrets Manager
1. You can associate up to 10 secrets with a cluster at a time using the BatchAssociateScramSecret operation
1. The name of secrets associated with an Amazon MSK cluster must have the prefix `AmazonMSK_`
1. Secrets associated with an Amazon MSK cluster must be in the same Amazon Web Services account and AWS region as the cluster

<br>

Source: https://docs.aws.amazon.com/msk/latest/developerguide/msk-password.html#msk-password-limitations

## Steps

### 1. Start your MSK Kafka cluster with SASL/SCRAM AND IAM


- Sign in to the A[WS Management Console and open the Amazon MSK console](https://console.aws.amazon.com/msk/)
- Choose Create cluster
- Enter a cluster name, and leave all other settings unchanged
- From the table under All cluster settings, copy the values of the following settings and save them because you need them later in this tutorial: VPC, Subnets, Security groups associated with VPC
- Choose Create cluster

_Note: Creation will take about 15 minutes._

Later we will change the default env configuration for the SimpleAclAuthorizer:
KAFKA_ALLOW_EVERYONE_IF_NO_ACL_FOUND: "TRUE"


Source: https://docs.aws.amazon.com/msk/latest/developerguide/msk-configuration.html


### 2. Make the cluster public and enable SASL + IAM
   
   a. Navigate to the [AWS MSK console](https://console.aws.amazon.com/msk/)<br>
   b. Choose the MSK cluster you just created in Step 1<br>
   c. Click on the Properties tab<br>
   d. In the Security settings section, choose Edit<br>
   e. Check the checkbox next to SASL/SCRAM authentication and IAM<br>
   f. Click Save changes<br>
   <br>
   You can find more details about updating a cluster’s security configurations [here](https://docs.aws.amazon.com/msk/latest/developerguide/msk-update-security.html).

**Create a Symmetric Key**

a. Now go to the [AWS Key Management Service (AWS KMS) console](https://console.aws.amazon.com/kms)<br>
b. Click Create Key<br>
c. Choose Symmetric and click Next<br>
d. Give the key and Alias and click Next<br>
e. Under Administrative permissions, check the checkbox next to the AWSServiceRoleForKafka and click Next<br>
f. Under Key usage permissions, again check the checkbox next to the AWSServiceRoleForKafka and click Next<br>
g. Click on Create secret<br>
h. Review the details and click Finish<br>


You can find more details about creating a symmetric key [here](https://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html#create-symmetric-cmk).


**Store a new Secret**

a. Go to the [AWS Secrets Manager console](https://console.aws.amazon.com/secretsmanager/) <br>
b. Click Store a new secret<br>
c. Choose `Other` type of secret (e.g. API key) for the secret type<br>
d. Under Key/value pairs click on Plaintext<br>
e. Paste the following in the space below it and replace <your-username> and <your-password> with the username and password you want to set for the cluster<br>
```json
{
"username": "<your-username>",
"password": "<your-password>"
}
```


f. On the next page, give a Secret name that starts with `AmazonMSK_`<br>
g. Under Encryption Key, select the symmetric key you just created in the previous sub-section from the dropdown<br>
h. Go forward to the next steps and finish creating the secret. Once created, record the ARN (Amazon Resource Name) value for your secret
You can find more details about creating a secret using AWS Secrets Manager [here](https://docs.aws.amazon.com/msk/latest/developerguide/msk-password.html).<br>


**Associate secrets with MSK cluster**

a. Navigate back to the [AWS MSK console](https://console.aws.amazon.com/msk/) and click on the cluster you created in Step 1<br>
b. Click on the Properties tab<br>
c. In the Security settings section, under SASL/SCRAM authentication, click on Associate secrets<br>
d. Paste the ARN you recorded in the previous subsection and click Associate secrets<br>

**Create the cluster’s configuration**

a. Go to the [AWS CloudShell console](https://console.aws.amazon.com/cloudshell/)<br>
b. Create a file (eg. msk-config.txt) with the following line
`allow.everyone.if.no.acl.found = false`<br>

c. Run the following AWS CLI command, replacing <config-file-path> with the path to the file where you saved your configuration in the previous step<br>

```bash
aws kafka create-configuration --name "MakePublic" \
--description "Set allow.everyone.if.no.acl.found = false" \
--kafka-versions "2.6.2" \
--server-properties fileb://<config-file-path>/msk-config.txt
```

You can find more information about making your cluster public [here](https://docs.aws.amazon.com/msk/latest/developerguide/public-access.html).

### 3. Create a client machine (using IAM)

If you already have a client machine set up that can interact with your cluster, then you can skip this step.
If not, you can create an EC2 client machine and then add the security group of the client to the inbound rules of the cluster’s security group from the VPC console. You can find more details about how to do that [here](https://docs.aws.amazon.com/msk/latest/developerguide/create-client-machine.html).

### 4.  Install Apache Kafka on Client machine
We install Apache Kafka to test and check ACLs, Topic creation etc. You can find more information about how to do that [here](https://docs.aws.amazon.com/msk/latest/developerguide/create-topic.html).


### 5. Create the domain/user secret (principle) for use with SASL/SCRAM

SpecMesh specs have an ‘id’ - this ‘id’ is used as the principle (user). This way SpecMesh will automatically generate and apply ACLs that govern which ‘id’s can access various topics.


Create the SASL/SCRAM secrets for each id and associate them with the cluster.

https://catalog.workshops.aws/msk-labs/en-US/securityencryption/saslscram/authorization


With SASL/SCRAM the username/secret is stored within the secrets-manger. The username is is also called the `principle`,  it and the secret are provided to clients (via properties files). Each secret is associated with your cluster.  When the client connects to MSK, the broker retrieves the credentials from SecretsManager and applies the associated ACLs


### 6. Create client properties files with security credentials


Configure Kafka (Java) clients with the appropriate SASL/SCRAM credentials as shown on the following pages<br>

https://catalog.workshops.aws/msk-labs/en-US/securityencryption/saslscram/authorization#client-setup

Associate secrets:
https://catalog.workshops.aws/msk-labs/en-US/securityencryption/saslscram/authorization

### 7. Provision Specs


Run the SpecMesh CLI `provision` command against each Spec. This will create Topics, Publish schemas and configure ACLs according to the Spec. The ‘id’ of the spec matches the principle.

1. Log onto the client machine**
1. Checkout/Pull the PR
2. Execute SpecMesh CLI `provision` via docker - as shown here

```bash
 % docker run --rm -v "$(pwd)/resources:/app" ghcr.io/specmesh/specmesh-build-cli provision -bs kafka:9092 -sr http://schema-registry:8081 -spec /app/simple_schema_demo-api.yaml -schemaPath /app
```

_Note: the docker runtime can access the broker and schema registry url_

Use the SpecMesh CLI `provision`  - learn more here: https://github.com/specmesh/specmesh-build/blob/main/cli/README.md


Learn more [here](https://specmesh.io/blog/specmeshprovisioningkafkaresourceandasyncapi/)

### 8. Verify Topics and ACLs exist

From the client machine (it has super-user permissions via IAM, MSK service role):
1. list topics: `$ bin/kafka-topics.sh --list --zookeeper localhost:2181`
1. list ACLs: `kafka-acls.sh --authorizer kafka.security.auth.SimpleAclAuthorizer --authorizer-properties zookeeper.connect=localhost:2181 --list --topic SommeTopic`

See:
- [Listing ACLs](https://www.oreilly.com/library/view/building-data-streaming/9781787283985/f8af63c6-ff0a-4832-905e-e9abf4092895.xhtml)
- [Listing topics](https://www.baeldung.com/ops/kafka-list-topics)


<br>
<br>

## Conclusion

Kafka ACLs, RBaC and IAM are challenging to manage in a large scale deployment. SpecMesh simplifies ACLs using a familiar model of hierarchies. Do you really need RBaC when you have private/public/protected. This model is much simpler.


---



Resources:
 - https://www.baeldung.com/ops/kafka-list-topics
 - MSK setup: https://materialize.com/docs/ingest-data/amazon-msk/ 
 - Acls: https://docs.aiven.io/docs/products/kafka/concepts/acl
 - Quirks: https://medium.com/dev-genius/amazon-msk-tips-quirks-7b1e56d53296
 - Debugging: https://docs.confluent.io/platform/current/kafka/authorization.html#debug-using-authorizer-logs

 - https://specmesh.io/blog/specmeshenterpriseevents-p1/
 - https://specmesh.io/
 - https://github.com/specmesh/specmesh-build
 - https://github.com/specmesh/docs/wiki
 - https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/control-plane-vs.-application-plane.html
 - https://www.linkedin.com/newsletters/6937422474598883328/
 - https://martinfowler.com/articles/data-mesh-principles.html
 - https://www.confluent.io/en-gb/blog/event-streaming-benefits-increase-with-greater-maturity
 - https://www.infoq.com/news/2019/06/bounded-context-eric-evans/
 - https://www.asyncapi.com/
 - https://conference.asyncapi.com/

