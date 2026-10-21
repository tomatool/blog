+++
title = 'Testing a Kafka service that signs in to MSK with IAM'
date = 2026-10-21T06:15:00+02:00
summary = "A service that produces to Amazon MSK with IAM has no Kafka password: it trades a web identity token for a role session at STS and signs the handshake with it. Tests usually switch all of that off. tomato runs it: its aws resource serves STS and its kafka preset speaks AWS_MSK_IAM, so the service's own auth code runs, and fails, the way it does on AWS."
tags = ['kafka', 'msk', 'aws iam', 'black-box testing']
+++

A service on EKS that produces to Amazon MSK with IAM authentication has no
Kafka username or password. Its pod is given a role and a web identity token
(IRSA), the AWS SDK trades the token at STS for a short-lived session, the
Kafka client signs its handshake with that session, and MSK lets it in if the
role's IAM policy allows it.

Tests usually turn all of that off and connect to a plaintext broker, which
leaves the code that most often breaks in a new environment untested: the
SASL mechanism, the credential chain, the role. tomato keeps it on. Its `aws`
resource serves STS and hands the service a token and a role, and its
`kafka` preset adds a listener that speaks MSK's `AWS_MSK_IAM` handshake. Step
through a run:

{{< msk-iam-flow >}}

## The service

`orders` takes an order over HTTP and produces it to the `orders` topic. The
Kafka client is configured from the environment only, and takes its
credentials from the AWS SDK's default chain; nothing in it knows about
tomato. These are the parts that matter, in four languages:

{{< code-tabs >}}
{{< tab "Go" >}}
```go
awsCfg, err := config.LoadDefaultConfig(ctx) // web identity, via AWS_ROLE_ARN + token file
if err != nil {
	log.Fatal(err)
}
opts := []kgo.Opt{
	kgo.SeedBrokers(strings.Split(os.Getenv("KAFKA_BROKERS"), ",")...),
	kgo.RecordDeliveryTimeout(10 * time.Second), // fail the request instead of retrying forever
	kgo.SASL(aws.ManagedStreamingIAM(func(ctx context.Context) (aws.Auth, error) {
		c, err := awsCfg.Credentials.Retrieve(ctx)
		return aws.Auth{AccessKey: c.AccessKeyID, SecretKey: c.SecretAccessKey, SessionToken: c.SessionToken}, err
	})),
}
if os.Getenv("KAFKA_TLS") == "true" {
	opts = append(opts, kgo.DialTLSConfig(&tls.Config{}))
}
kafka, err := kgo.NewClient(opts...)
```

[franz-go](https://github.com/twmb/franz-go) v1.22 with the AWS SDK for Go v2. Build it first
(`go build -o orders .`); tomato runs the binary.
Without `RecordDeliveryTimeout` franz-go retries a refused handshake forever,
and a request that should fail hangs instead.
{{< /tab >}}
{{< tab "Java" >}}
```java
var producer = new KafkaProducer<String, String>(Map.of(
        "bootstrap.servers", System.getenv("KAFKA_BROKERS"),
        "security.protocol", "true".equals(System.getenv("KAFKA_TLS")) ? "SASL_SSL" : "SASL_PLAINTEXT",
        "sasl.mechanism", "AWS_MSK_IAM",
        "sasl.jaas.config", "software.amazon.msk.auth.iam.IAMLoginModule required;",
        "sasl.client.callback.handler.class", "software.amazon.msk.auth.iam.IAMClientCallbackHandler",
        "max.block.ms", "10000"),
        new StringSerializer(), new StringSerializer());
```

`kafka-clients` 4.4 with [aws-msk-iam-auth](https://github.com/aws/aws-msk-iam-auth)
2.3.9. Its `-all` jar includes the STS module the SDK needs for web identity;
with the slim jar, add `software.amazon.awssdk:sts` yourself.
{{< /tab >}}
{{< tab "Python" >}}
```python
tls = os.environ.get("KAFKA_TLS") == "true"

if not tls:
    # kafka-python only allows AWS_MSK_IAM over SASL_SSL; tomato's listener is SASL_PLAINTEXT.
    class MskIamPlaintext(SaslMechanismAwsMskIam):
        def __init__(self, **config):
            super().__init__(**{**config, "security_protocol": "SASL_SSL"})

    register_sasl_mechanism("AWS_MSK_IAM", MskIamPlaintext, overwrite=True)

producer = KafkaProducer(
    bootstrap_servers=os.environ["KAFKA_BROKERS"].split(","),
    security_protocol="SASL_SSL" if tls else "SASL_PLAINTEXT",
    sasl_mechanism="AWS_MSK_IAM",  # credentials: botocore's default chain
    key_serializer=str.encode,
    value_serializer=lambda v: json.dumps(v, separators=(",", ":")).encode(),
)
```

[kafka-python](https://github.com/dpkp/kafka-python) 3.0 with botocore. It
refuses `AWS_MSK_IAM` without TLS, so the five lines in `if not tls` lift that
check for the test run only. The `aws-msk-iam-sasl-signer` libraries use
`OAUTHBEARER` instead, which tomato's listener does not speak.
{{< /tab >}}
{{< tab "Node.js" "nodejs" >}}
```js
const kafka = new Kafka({
  brokers: process.env.KAFKA_BROKERS.split(","),
  ssl: process.env.KAFKA_TLS === "true",
  // AWS_MSK_IAM, credentials from the AWS SDK's default chain (fromNodeProviderChain)
  sasl: createMechanism({ region: process.env.AWS_REGION }),
});
const producer = kafka.producer();
await producer.connect();
```

kafkajs 2.2 with
[@jm18457/kafkajs-msk-iam-authentication-mechanism](https://www.npmjs.com/package/@jm18457/kafkajs-msk-iam-authentication-mechanism)
4.0, which implements `AWS_MSK_IAM` on top of the AWS SDK for JavaScript v3.
{{< /tab >}}
{{< /code-tabs >}}

The one switch is `KAFKA_TLS`. MSK's IAM port is `SASL_SSL`; tomato's is
`SASL_PLAINTEXT`, the same handshake without TLS.

## The suite

`tomato.yml` is the same for every language except `app.command`:

```yaml
version: 2

containers:
  kafka:
    preset: kafka
    auth: aws_msk_iam          # adds the AWS_MSK_IAM listener on 9098

app:
  command: ./orders
  port: 8097
  ready:
    type: http
    path: /health
    timeout: 120s
  env:
    PORT: "8097"
    KAFKA_BROKERS: "{{.kafka.host}}:{{.kafka.port.9098}}"

resources:
  orders-identity:             # STS, a token file and a role, in the app's environment
    type: aws
    options:
      region: eu-central-1
      role_arn: arn:aws:iam::000000000000:role/orders

  orders-events:               # reads the topic on the plain listener
    type: kafka
    container: kafka
    options:
      topics: [orders]

  orders-client:
    type: http-client
    base_url: "http://localhost:8097"
```

`./orders` is the Go binary. For Java the command is `java -cp lib/* Orders.java`,
for Python `uv run orders.py`, and for Node.js `node orders.mjs`.

The service is never handed a Kafka credential or an AWS key. The `aws`
resource sets `AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE` and
`AWS_ENDPOINT_URL_STS`, and removes any AWS keys the service would otherwise
inherit from your shell or CI, so it can only get in the way it does on EKS.

```gherkin
Feature: placing an order publishes it to Kafka

  Scenario: a placed order is published
    Given "orders-events" consumes from "orders"
    When "orders-client" sends "POST" to "/orders" with json:
      """
      {"id": "ord-1", "total": 1999}
      """
    Then "orders-client" response status is "201"
    And "orders-events" receives from "orders" within "10s":
      """
      {"id":"ord-1","total":1999,"status":"placed"}
      """
    And "orders-events" last message has key "ord-1"
    And "orders-identity" role "arn:aws:iam::000000000000:role/orders" was assumed

  Scenario: a bad order publishes nothing
    Given "orders-events" consumes from "orders"
    When "orders-client" sends "POST" to "/orders" with json:
      """
      {"total": 1999}
      """
    Then "orders-client" response status is "400"
    And "orders-events" topic "orders" is empty
```

```text
$ tomato run
2 scenarios (2 passed)
10 steps (10 passed)
```

All four services pass it unchanged. The last step of the first scenario is
the one a plaintext test cannot have: the service got in as the role it was
meant to.

## In production

The same variables, set by EKS and your deployment instead of tomato:

| Variable | Under tomato | On EKS |
|---|---|---|
| `KAFKA_BROKERS` | `localhost:<port>`, the IAM listener | MSK's IAM bootstrap brokers, port 9098 |
| `KAFKA_TLS` | unset: `SASL_PLAINTEXT` | `true`: `SASL_SSL` |
| `AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE` | set by the `aws` resource | set by EKS from the service account's role annotation |
| `AWS_ENDPOINT_URL_STS` | tomato's STS on 127.0.0.1 | unset: the SDK's regional STS |
| `AWS_REGION` | `region` of the `aws` resource | the cluster's region |

## When the service has the wrong identity

The failure this setup catches is the one that only shows up on AWS: the
service authenticates as someone other than the role with the Kafka policy.
The usual cause is a pod that was not given the role, so the SDK's credential
chain keeps looking and finds the node's identity instead. Reproduce it by
leaving the role out of the service's environment, and giving the chain
something to fall back to:

```yaml
app:
  env:
    AWS_ROLE_ARN: ""                 # the service account lost its role annotation
    AWS_WEB_IDENTITY_TOKEN_FILE: ""

resources:
  orders-identity:
    type: aws
    options:
      region: eu-central-1
      role_arn: arn:aws:iam::000000000000:role/orders
      ambient_identity: true         # a long-term identity where the chain looks next
```

Every SDK falls back to it, and the listener refuses it with MSK's own
answer:

```text
SASL_AUTHENTICATION_FAILED: SASL Authentication failed.: [<request id>]: Access denied
```

How that reaches the scenario depends on the client. Go and Java keep running
and answer the order with a 502. Python and Node.js connect when they start,
so the service never becomes ready and tomato stops with `app not ready`.
Either way the run fails, as the deployment would.

## What tomato checks, and what it doesn't

The listener checks **who** the client is: it reads the access key id out of
the signed `kafka-cluster:Connect` payload and lets in only the sessions the
`aws` resource issued for the configured role. Both sides derive that key from
the role ARN, so they agree without talking to each other. That covers the
mechanism, the credential chain, the STS call and the role.

It does not verify the signature, and it does not evaluate IAM policies, so a
policy that allows `Connect` but not `WriteData` on the topic passes here and
fails on MSK. It has no TLS. Those belong to the cluster's own configuration,
and are worth a check in a real AWS environment before the first deploy.
