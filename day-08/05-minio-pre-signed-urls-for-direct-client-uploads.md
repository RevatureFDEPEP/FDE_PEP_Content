# MinIO (S3-Compatible) Pre-Signed URLs for Direct Client Uploads

> *Day 8: Question Authoring Slice (Backend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview

Questions can have image attachments — a diagram, a screenshot of code, a chart. The naive flow ("frontend uploads to backend, backend uploads to MinIO") wastes bandwidth, doubles latency, and balloons backend memory on large files. The grown-up flow is a **pre-signed URL**: the backend issues a short-lived signed `PUT` URL, the frontend uploads *directly* to MinIO, and the backend only stores the resulting object key on the question document.

Today you wire that up. MinIO is already containerized (Day 2) and reachable inside the stack; this topic turns it into a usable attachment store from `question-management-service`.

## What a pre-signed URL actually is

A pre-signed URL is a regular S3 URL with extra query parameters that *prove* the request is authorized — an HMAC of the request method, path, headers, and an expiration timestamp, signed with the service's secret key. MinIO (and S3) accept the URL from anyone, because the signature itself is the credential. No client SDK, no IAM, no proxy through the backend.

Three properties matter:

1. **Method-bound.** A pre-signed PUT URL only works for PUT, not for GET or DELETE.
2. **Time-bound.** The URL is valid until its `X-Amz-Expires` window elapses (you set it).
3. **Object-bound.** The signature covers the bucket and key; a URL for `images/abc.png` can't be used to upload to `images/def.png`.

The frontend never sees the MinIO credentials. The backend never sees the file bytes. Both wins.

## Boto3 against MinIO

MinIO speaks S3, but a few client knobs are non-default:

```python
# services/question-management-service/app/storage/minio_client.py
import boto3
from botocore.client import Config
from app.config import settings

def make_s3_client():
    return boto3.client(
        "s3",
        endpoint_url=settings.MINIO_ENDPOINT_URL,       # e.g., "http://minio:9000"
        aws_access_key_id=settings.MINIO_ACCESS_KEY,
        aws_secret_access_key=settings.MINIO_SECRET_KEY,
        region_name="us-east-1",                         # MinIO ignores it, boto3 requires it
        config=Config(
            signature_version="s3v4",
            s3={"addressing_style": "path"},             # CRITICAL: MinIO needs path-style
        ),
    )
```

Two settings the AWS SDK gets wrong by default against MinIO:

- **`endpoint_url=`** — without it, boto3 talks to real S3 (and your secrets leak nowhere useful).
- **`addressing_style: "path"`** — by default, boto3 uses *virtual-hosted-style* (`https://bucket.s3.amazonaws.com/key`). MinIO needs path-style (`http://minio:9000/bucket/key`). Setting this wrong yields a confusing "could not resolve host" error.

`signature_version="s3v4"` is the modern default but worth being explicit.

## Issuing a pre-signed PUT URL

```python
# services/question-management-service/app/storage/uploads.py
import uuid
from datetime import datetime, timezone
from app.storage.minio_client import make_s3_client
from app.config import settings

ALLOWED_TYPES = {"image/png", "image/jpeg", "image/webp"}
MAX_IMAGE_BYTES = 5 * 1024 * 1024  # 5 MiB

def issue_image_upload_url(content_type: str, content_length: int) -> dict:
    if content_type not in ALLOWED_TYPES:
        raise ValueError(f"unsupported content type: {content_type}")
    if content_length <= 0 or content_length > MAX_IMAGE_BYTES:
        raise ValueError(f"content_length must be in 1..{MAX_IMAGE_BYTES}")

    today = datetime.now(timezone.utc).strftime("%Y/%m/%d")
    ext = {"image/png": "png", "image/jpeg": "jpg", "image/webp": "webp"}[content_type]
    key = f"questions/{today}/{uuid.uuid4().hex}.{ext}"

    s3 = make_s3_client()
    url = s3.generate_presigned_url(
        ClientMethod="put_object",
        Params={
            "Bucket": settings.MINIO_BUCKET,
            "Key": key,
            "ContentType": content_type,
            "ContentLength": content_length,
        },
        ExpiresIn=300,  # 5 minutes
        HttpMethod="PUT",
    )
    return {"upload_url": url, "key": key, "expires_in": 300}
```

Choices worth defending:

- **Backend chooses the key.** Never let the client name objects directly — they will pick collisions, traversals, or unicode horrors. Random UUID inside a dated prefix is fine.
- **`ContentType` and `ContentLength` in `Params`.** These become part of the signature; the client *must* send matching headers or the upload is rejected. That's how you actually enforce "<= 5 MiB PNG/JPEG/WebP" — the signature carries the constraint.
- **`ExpiresIn=300`.** Long enough for a normal user to upload, short enough that a leaked URL has limited blast radius.

## The FastAPI endpoint

```python
# services/question-management-service/app/routes/uploads.py
from typing import Annotated, Literal
from fastapi import APIRouter, HTTPException, status
from pydantic import BaseModel, Field
from app.storage.uploads import issue_image_upload_url

router = APIRouter(prefix="/uploads", tags=["uploads"])

class ImageUploadRequest(BaseModel):
    content_type: Literal["image/png", "image/jpeg", "image/webp"]
    content_length: int = Field(gt=0, le=5 * 1024 * 1024)

class ImageUploadResponse(BaseModel):
    upload_url: str
    key: str
    expires_in: int

@router.post("/image", response_model=ImageUploadResponse, status_code=status.HTTP_201_CREATED)
async def request_image_upload(req: ImageUploadRequest) -> ImageUploadResponse:
    try:
        result = issue_image_upload_url(req.content_type, req.content_length)
    except ValueError as e:
        raise HTTPException(status.HTTP_422_UNPROCESSABLE_ENTITY, detail=str(e))
    return ImageUploadResponse(**result)
```

The frontend then writes the returned `key` into the question's `image_key` field (Topic 1) when it POSTs `/questions`. The chain is: ask for URL -> upload to MinIO -> save question pointing to `key`.

## Verifying the loop with curl

Before the Day 9 frontend exists, validate the round-trip from the terminal — this is the single best debugging tool for this whole topic.

```bash
# 1. Ask the backend for a pre-signed URL
RESP=$(curl -s -X POST http://localhost/api/uploads/image \
  -H "Content-Type: application/json" \
  -d '{"content_type":"image/png","content_length":1234}')
echo "$RESP" | jq .

UPLOAD_URL=$(echo "$RESP" | jq -r .upload_url)
KEY=$(echo "$RESP" | jq -r .key)

# 2. Upload directly to MinIO using the URL
#    Note: headers MUST match what was signed (Content-Type, Content-Length)
curl -v -X PUT "$UPLOAD_URL" \
  -H "Content-Type: image/png" \
  --data-binary @./sample.png

# 3. Confirm the object exists (via MinIO console or mc)
docker compose exec minio mc ls myminio/rev-eval-ai-pep/$KEY
```

Expected outcomes:

- Step 2 returns `200 OK` from MinIO. If it returns `403 SignatureDoesNotMatch`, your `Content-Type` header doesn't match the signed value (almost always the cause).
- Step 3 shows the object with the size from `content_length`.

Once curl works end-to-end, the frontend's `fetch(upload_url, { method: 'PUT', headers, body })` will work too.

## Bucket setup and CORS

A one-time bootstrap (compose service or trainer-prep step) creates the bucket and sets CORS so browser uploads work:

```bash
mc alias set local http://minio:9000 minioadmin minioadmin
mc mb --ignore-existing local/rev-eval-ai-pep
mc anonymous set download local/rev-eval-ai-pep   # if image URLs need to be readable directly
mc cors set /etc/minio/cors.json local/rev-eval-ai-pep
```

`cors.json` allows `PUT` from the frontend origin:

```json
{
  "CORSRules": [{
    "AllowedOrigins": ["http://localhost"],
    "AllowedMethods": ["PUT", "GET"],
    "AllowedHeaders": ["*"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }]
}
```

Without CORS, the browser blocks the upload before it even hits MinIO and you spend an hour staring at backend logs that show nothing.

## Example / Worked Scenario

The Day 9 form has an "Attach image" button. The flow it implements:

1. User picks `diagram.png` (50 KB, `image/png`).
2. Frontend calls `POST /api/uploads/image` with `{content_type, content_length}`.
3. Backend validates against allow-list, generates a UUID key under `questions/2026/05/19/`, returns `{upload_url, key, expires_in: 300}`.
4. Frontend does `fetch(upload_url, { method: "PUT", headers: { "Content-Type": "image/png" }, body: file })` directly to MinIO. No backend memory used; no proxy.
5. On success, frontend stores `key` and assembles the question payload: `{ type: "multi_select", ..., image_key: "questions/2026/05/19/abc...png" }`.
6. `POST /api/questions` persists the question; `image_key` lives in the document (Topic 3); Day 10 will issue a *read* pre-signed URL on retrieval.

The signed URL is single-use in practice — once the question is saved with that `key`, no further uploads to it happen.

## Common Pitfalls

- **Forgetting path-style addressing.** Without `addressing_style: "path"`, boto3 tries `bucket.minio:9000`, DNS fails, you blame networking. The fix is one line.
- **Letting the client choose the object key.** Users will collide, escape the prefix, or include `../`. Always generate the key on the backend.
- **Not signing `ContentType` / `ContentLength`.** If you omit them from `Params`, the client can upload any size of any type and your "5 MiB PNG only" rule becomes a polite suggestion. Sign the constraints in.
- **Expiry too long.** `ExpiresIn=86400` looks convenient and creates a leak window. Five minutes is enough; ten is generous.
- **Forgetting CORS on the bucket.** Direct browser uploads fail with no useful error in the backend logs because the request never arrived. Browser console shows the preflight failing — read it.
- **Using the wrong endpoint URL inside vs outside the compose network.** `http://minio:9000` works from another container; the browser needs `http://localhost:9000` (or whatever the reverse proxy exposes). The pre-signed URL has the hostname baked in — set `endpoint_url` to the host the *client* will use.

## Key Takeaways

- Pre-signed URLs offload bytes from the backend: the client uploads directly to MinIO, the backend only handles small JSON.
- For MinIO, set `endpoint_url`, `addressing_style="path"`, and `signature_version="s3v4"` — the boto3 defaults assume real S3.
- The backend picks the key, signs the `ContentType` and `ContentLength`, and uses a short `ExpiresIn`. Constraints in the signature are constraints the client can't bypass.
- Verify the round-trip with curl before integrating a frontend — `403 SignatureDoesNotMatch` almost always means the upload's `Content-Type` header doesn't match what was signed.
- Configure bucket CORS once at bootstrap; the browser will silently refuse to upload without it.
- The question document only stores the `image_key`; read-side pre-signed GETs come later in the slice.

---
*Prerequisites: [04-fastapi-request-response-patterns.md](04-fastapi-request-response-patterns.md), Day 2 (MinIO containerization, compose networking).*
