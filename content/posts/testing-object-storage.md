+++
title = 'Testing code that writes to object storage, whichever one you use'
date = 2026-10-21T05:00:00+02:00
summary = "S3, Cloudflare R2, DigitalOcean Spaces, Backblaze B2 and Google Cloud Storage all speak the S3 API. So one MinIO container can stand in for any of them in a tomato run, and the service's code doesn't change, only four environment variables."
tags = ['s3', 'object storage', 'black-box testing']
+++

Most object stores speak the S3 API: AWS S3 itself, Cloudflare R2,
DigitalOcean Spaces, Backblaze B2, Wasabi, and Google Cloud Storage through its
interoperability endpoint. A service written against that API runs on any of
them. Only its configuration changes: the endpoint, the region, the
credentials, and whether buckets are addressed by path or by host name.

That makes object storage easy to test. In a tomato run, one MinIO container
stands in for whichever store you use in production, and tomato's `s3`
resource reads the bucket to check what the service wrote:

{{< storage-flow >}}

## The service

The service builds its client from the environment and nothing else. Here is
the client for a small Go service, `invoices`, that stores each invoice it
issues as JSON:

```go
func newStore() *s3.Client {
	return s3.New(s3.Options{
		Region:       os.Getenv("STORAGE_REGION"),
		BaseEndpoint: aws.String(os.Getenv("STORAGE_ENDPOINT")),
		UsePathStyle: os.Getenv("STORAGE_PATH_STYLE") == "true",
		Credentials: credentials.NewStaticCredentialsProvider(
			os.Getenv("STORAGE_ACCESS_KEY"), os.Getenv("STORAGE_SECRET_KEY"), ""),
		// Only send checksums the operation requires: not every
		// S3-compatible provider accepts the SDK's default CRC32 headers.
		RequestChecksumCalculation: aws.RequestChecksumCalculationWhenRequired,
	})
}
```

The handler calls `PutObject` with the key `invoices/<id>.json`. It never
asks which provider it is talking to.

## The suite

`tomato.yml` starts MinIO, hands the service MinIO's address in the same
variables, and points the `s3` resource at the same container:

```yaml
version: 2

containers:
  minio:
    image: cgr.dev/chainguard/minio:latest
    command: ["server", "/data"]
    ports: ["9000/tcp"]
    env:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    wait_for:
      type: port
      target: "9000"

app:
  command: go run .
  port: 8097
  ready:
    type: http
    path: /health
  env:
    PORT: "8097"
    STORAGE_ENDPOINT: "http://{{.minio.host}}:{{.minio.port.9000}}"
    STORAGE_REGION: us-east-1
    STORAGE_PATH_STYLE: "true"
    STORAGE_ACCESS_KEY: minioadmin
    STORAGE_SECRET_KEY: minioadmin
    STORAGE_BUCKET: billing

resources:
  invoices-storage:
    type: s3
    container: minio
    options:
      buckets: [billing]        # created before the first scenario
      reset_buckets: [billing]  # emptied before every scenario

  invoices-client:
    type: http-client
    base_url: "http://localhost:8097"
```

The scenarios check the service's effect, the object, not only its HTTP
response. In the `s3` resource's steps a path is `bucket/key`:

```gherkin
Feature: Invoices are stored in object storage

  Scenario: An issued invoice is written to the bucket
    When "invoices-client" sends "POST" to "/invoices" with json:
      """
      {"id": "inv-42", "amount": 1999}
      """
    Then "invoices-client" response status is "201"
    And "invoices-storage" object "billing/invoices/inv-42.json" should exist
    And "invoices-storage" object "billing/invoices/inv-42.json" content type should be "application/json"
    And "invoices-storage" object "billing/invoices/inv-42.json" content should match:
      """
      {"id": "inv-42", "amount": 1999, "status": "issued"}
      """

  Scenario: Each scenario starts with an empty bucket
    Then "invoices-storage" bucket "billing" should be empty

  Scenario: A bad request writes nothing
    When "invoices-client" sends "POST" to "/invoices" with json:
      """
      {"amount": 1999}
      """
    Then "invoices-client" response status is "400"
    And "invoices-storage" bucket "billing" should be empty
```

```text
$ tomato run
3 scenarios (3 passed)
9 steps (9 passed)
```

## In production

The same four variables, set per environment. These are the values for the
common providers:

| Provider | `STORAGE_ENDPOINT` | `STORAGE_REGION` | `STORAGE_PATH_STYLE` | Keys |
|---|---|---|---|---|
| AWS S3 | `https://s3.<region>.amazonaws.com` | `<region>` | `false` | IAM access key |
| Cloudflare R2 | `https://<account>.r2.cloudflarestorage.com` | `auto` | `true` | R2 API token |
| DigitalOcean Spaces | `https://<region>.digitaloceanspaces.com` | `us-east-1` | `true` | Spaces key |
| Backblaze B2 | `https://s3.<region>.backblazeb2.com` | `<region>` | `true` | application key |
| Wasabi | `https://s3.<region>.wasabisys.com` | `<region>` | `true` | access key |
| Google Cloud Storage | `https://storage.googleapis.com` | `auto` | `true` | HMAC key |

## When it isn't the S3 API

The approach holds as long as the service talks the S3 API. A service that
uses Google's own client library (`cloud.google.com/go/storage` and its
counterparts in other languages) talks GCS's JSON API instead, and MinIO does
not answer it. There are two ways to test it with tomato:

- **Switch the service to GCS's interoperability endpoint** with an HMAC key,
  as in the last row of the table. Then everything on this page applies.
- **Keep the native client** and check the service's effect through its own
  API, for example a `GET /invoices/inv-42`, rather than through the bucket.
  tomato has no resource for the JSON API today.

The same goes for Azure Blob Storage, which has an API of its own.

## Against the real bucket

The `s3` resource can also point at a hosted provider: set `options.endpoint`,
`access_key`, `secret_key` and `region` in place of `container`. Keep MinIO
for the suite itself, and if you do this for a smoke run, mind three things:

- **Always set `reset_buckets`.** Without it, tomato empties *every* bucket
  the key can list before each scenario. That is the right default for a
  throwaway container and the wrong one for an account with production data
  in it. Better still, use a key that can only reach a test bucket.
- **The key needs `ListBuckets`.** The resource lists buckets to check that
  the store is ready, so a key scoped to a single bucket fails that check.
- **Google Cloud Storage has no multi-object delete**, which tomato uses to
  empty a bucket, so a reset fails there. Run against GCS through MinIO.
