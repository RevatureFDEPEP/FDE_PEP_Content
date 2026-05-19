# MinIO Containerization — S3 Emulation for Local Object Storage

> *Day 2: Local Dev Stack with Docker Compose — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
v3.0 of the curriculum replaces AWS S3 in the local stack with **MinIO**, an S3-compatible object store you can run as a container. The application code uses `boto3` exactly as it would against real S3 — only the endpoint URL changes. Today you stand it up, create the bucket the substrate expects, and prove a `boto3` client can list it; Week 2's image-upload work for question authoring builds directly on this.

## What MinIO Is

MinIO is an open-source object storage server that implements the S3 API on a local filesystem (or distributed across nodes — irrelevant for our single-node dev use case). For our purposes:

- It speaks the S3 REST API at port `9000`.
- It serves a web console at port `9001`.
- It is configured with a root access key and secret, just like an S3 IAM user.
- Data lives at `/data` inside the container; like Postgres and Mongo, persist it with a named volume.

The S3-compatibility is the entire point: your application code does not know it is talking to MinIO. When the substrate eventually points at real S3 in deployed environments (Week 3), the only change is the `endpoint_url` argument and the credential source.

## Compose Definition

```yaml
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"     # S3 API
      - "9001:9001"     # Web console

  minio-init:
    image: minio/mc:latest
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      mc alias set local http://minio:9000 minioadmin minioadmin;
      mc mb --ignore-existing local/question-images;
      mc anonymous set download local/question-images;
      "

volumes:
  minio_data:
```

Two services, not one:

1. **`minio`** — the storage server itself. Note the explicit `command:` overriding the image's default; MinIO needs to know which directory to serve and where to put the console.
2. **`minio-init`** — a one-shot companion that uses MinIO's `mc` CLI to create the `question-images` bucket the substrate expects. It runs once, exits, and stays exited. This is a standard pattern for "create bucket on stack-up."

## Connection — boto3 Endpoint Override

The `boto3` client gets pointed at MinIO via `endpoint_url`:

```python
import boto3

s3 = boto3.client(
    "s3",
    endpoint_url="http://minio:9000",           # Compose DNS, not s3.amazonaws.com
    aws_access_key_id="minioadmin",
    aws_secret_access_key="minioadmin",
    region_name="us-east-1",                    # any value; MinIO ignores it
    config=boto3.session.Config(s3={"addressing_style": "path"}),
)

s3.list_buckets()
s3.put_object(Bucket="question-images", Key="hello.txt", Body=b"hi")
```

The `addressing_style="path"` is the most commonly missed config: MinIO uses path-style addressing (`http://minio:9000/question-images/key`), whereas real S3 defaults to virtual-host style (`http://question-images.s3.amazonaws.com/key`). With no host DNS for `question-images.minio` on the Compose network, virtual-host style fails. Force path style and it works against both MinIO and S3.

From the host (browser or `curl`), `localhost:9000` is the API and `localhost:9001` is the console — log into the console with `minioadmin` / `minioadmin` to visually inspect buckets and objects.

## Example / Worked Scenario

Stand the stack up and confirm the bucket exists:

```bash
docker compose up -d minio minio-init
docker compose logs minio-init
```

Expected log lines: `Bucket created successfully` (or "already exists" on subsequent runs because of `--ignore-existing`). Now exercise the bucket with `mc` directly:

```bash
docker compose run --rm minio-init mc ls local/
docker compose run --rm minio-init mc cp /etc/hostname local/question-images/smoke.txt
docker compose run --rm minio-init mc ls local/question-images/
```

And from the host using a Python script:

```python
# scratch.py
import boto3
from botocore.config import Config

s3 = boto3.client(
    "s3",
    endpoint_url="http://localhost:9000",
    aws_access_key_id="minioadmin",
    aws_secret_access_key="minioadmin",
    region_name="us-east-1",
    config=Config(s3={"addressing_style": "path"}),
)
print([o["Key"] for o in s3.list_objects_v2(Bucket="question-images").get("Contents", [])])
```

Open `http://localhost:9001` in a browser, log in, and you should see `question-images` containing `smoke.txt`.

## Common Pitfalls

- **Default (virtual-host) addressing style.** `boto3` against MinIO without `addressing_style="path"` fails with confusing DNS errors. Always set it for MinIO clients.
- **Confusing the two ports.** `9000` is the API (what `boto3` talks to); `9001` is the console (what your browser talks to). They are not interchangeable.
- **Forgetting the bucket creation step.** A fresh MinIO container has zero buckets. The `minio-init` companion exists specifically to create them. If you replicate this Compose file without the init service, you will hit `NoSuchBucket` errors.
- **Treating MinIO credentials as production-grade.** `minioadmin/minioadmin` is fine for local; real S3 in Week 3 uses IAM roles via the AWS SDK's default credential chain. Don't ship hard-coded keys.

## Key Takeaways

- MinIO is an S3-compatible object store you run as a container; application code uses `boto3` with `endpoint_url` overridden.
- Run two services: the MinIO server, and a one-shot `mc` companion that creates buckets on first start.
- Always set `addressing_style="path"` in `boto3` config when talking to MinIO.
- Port `9000` = S3 API; port `9001` = web console.

---
*Prerequisites: day-2-local-orchestration-with-docker-compose.md, day-2-postgresql-containerization.md (named volume pattern)*
