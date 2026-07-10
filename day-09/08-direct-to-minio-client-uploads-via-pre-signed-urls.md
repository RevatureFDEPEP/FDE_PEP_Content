# Direct-to-MinIO Client Uploads via Pre-Signed URLs

> *Day 9: Question Authoring Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
Day 8's backend exposes an endpoint that returns a pre-signed PUT URL for MinIO with signed `ContentType` and `ContentLength`. Today the frontend uses that URL to upload directly from the browser — no bytes flow through `question-management-service`. We build a drag-drop dropzone with progress (which requires `XMLHttpRequest` because `fetch` doesn't expose upload progress), wire it into the form so a successful upload stores the resulting `imageKey` in the form state, and address the inevitable MinIO CORS gotcha.

## The Two-Step Flow

```
Browser                          api-gateway / question-mgmt-svc          MinIO
  |                                       |                                  |
  |  1. POST /api/uploads/presign         |                                  |
  |     { filename, contentType, size }   |                                  |
  | ------------------------------------> |                                  |
  |                                       |                                  |
  |     { url, fields, key }              |                                  |
  | <------------------------------------ |                                  |
  |                                       |                                  |
  |  2. PUT <url>                         |                                  |
  |     Content-Type: <signed>            |                                  |
  |     Content-Length: <signed>          |                                  |
  |     body: <file bytes>                |                                  |
  | ----------------------------------------------------------------------> |
  |                                                                          |
  |     200 OK                                                               |
  | <---------------------------------------------------------------------- |
  |                                       |                                  |
  |  3. Form submit includes `imageKey`   |                                  |
  | ------------------------------------> |                                  |
```

Step 2 is the one that needs progress reporting and that triggers CORS.

## Step 1: Requesting the Pre-Signed URL

```ts
// lib/api-uploads.ts
import { apiPost } from './api';

export type PresignRequest = {
  filename: string;
  contentType: string;
  contentLength: number;
};

export type PresignResponse = {
  url: string;        // signed PUT URL
  key: string;        // object key under the bucket, e.g. "questions/01HX.../image.png"
  contentType: string;
  expiresInSeconds: number;
};

export function presignQuestionImage(req: PresignRequest) {
  return apiPost<PresignRequest, PresignResponse>('/api/uploads/question-image/presign', req);
}
```

## Step 2: PUT with Progress via XMLHttpRequest

`fetch` does not expose request upload progress. Use `XMLHttpRequest` and wrap it in a Promise:

```ts
// lib/upload.ts
export type UploadProgress = { loaded: number; total: number; percent: number };

export function putWithProgress(
  url: string,
  file: File,
  contentType: string,
  onProgress: (p: UploadProgress) => void,
  signal?: AbortSignal,
): Promise<void> {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open('PUT', url);
    xhr.setRequestHeader('Content-Type', contentType); // must match presigned ContentType exactly

    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) {
        onProgress({
          loaded: e.loaded,
          total: e.total,
          percent: Math.round((e.loaded / e.total) * 100),
        });
      }
    };

    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) resolve();
      else reject(new Error(`Upload failed: ${xhr.status} ${xhr.statusText}`));
    };
    xhr.onerror = () => reject(new Error('Network error during upload'));
    xhr.onabort = () => reject(new DOMException('Upload aborted', 'AbortError'));

    if (signal) {
      signal.addEventListener('abort', () => xhr.abort());
    }

    xhr.send(file);
  });
}
```

Key details:

- `Content-Type` header **must equal** what the backend signed. Mismatch → 403 SignatureDoesNotMatch.
- `Content-Length` is set automatically by the browser from the `File` blob — don't try to set it manually (browsers refuse).
- `xhr.upload.onprogress` exposes byte-level progress; `xhr.onprogress` is for download progress.

## The Dropzone Component

```tsx
'use client';
import { useCallback, useRef, useState } from 'react';
import { presignQuestionImage } from '@/lib/api-uploads';
import { putWithProgress } from '@/lib/upload';
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils';

type State =
  | { status: 'idle' }
  | { status: 'uploading'; percent: number; abort: AbortController }
  | { status: 'success'; key: string }
  | { status: 'error'; message: string };

const MAX_BYTES = 5 * 1024 * 1024; // 5 MB
const ACCEPTED = ['image/png', 'image/jpeg', 'image/webp'];

export function ImageDropzone({
  value,
  onChange,
}: {
  value?: string;
  onChange: (key: string | undefined) => void;
}) {
  const [state, setState] = useState<State>(
    value ? { status: 'success', key: value } : { status: 'idle' }
  );
  const inputRef = useRef<HTMLInputElement>(null);
  const [dragging, setDragging] = useState(false);

  const handleFile = useCallback(async (file: File) => {
    if (!ACCEPTED.includes(file.type)) {
      setState({ status: 'error', message: 'Only PNG, JPEG, or WebP allowed.' });
      return;
    }
    if (file.size > MAX_BYTES) {
      setState({ status: 'error', message: 'Max 5 MB.' });
      return;
    }

    const abort = new AbortController();
    setState({ status: 'uploading', percent: 0, abort });

    try {
      const presigned = await presignQuestionImage({
        filename: file.name,
        contentType: file.type,
        contentLength: file.size,
      });

      await putWithProgress(
        presigned.url,
        file,
        presigned.contentType,
        (p) => setState((s) => (s.status === 'uploading' ? { ...s, percent: p.percent } : s)),
        abort.signal,
      );

      setState({ status: 'success', key: presigned.key });
      onChange(presigned.key);
    } catch (e) {
      setState({ status: 'error', message: (e as Error).message });
      onChange(undefined);
    }
  }, [onChange]);

  const onDrop = (e: React.DragEvent) => {
    e.preventDefault();
    setDragging(false);
    const file = e.dataTransfer.files[0];
    if (file) void handleFile(file);
  };

  return (
    <div
      onDragOver={(e) => { e.preventDefault(); setDragging(true); }}
      onDragLeave={() => setDragging(false)}
      onDrop={onDrop}
      onClick={() => inputRef.current?.click()}
      className={cn(
        'border-2 border-dashed rounded-lg p-6 text-center cursor-pointer transition-colors',
        dragging ? 'border-primary bg-primary/5' : 'border-muted-foreground/30',
      )}
    >
      <input
        ref={inputRef}
        type="file"
        accept={ACCEPTED.join(',')}
        hidden
        onChange={(e) => {
          const f = e.target.files?.[0];
          if (f) void handleFile(f);
        }}
      />

      {state.status === 'idle' && <p>Drag an image here, or click to choose. PNG/JPEG/WebP, ≤5 MB.</p>}

      {state.status === 'uploading' && (
        <div>
          <p>Uploading… {state.percent}%</p>
          <div className="w-full h-2 bg-muted rounded mt-2 overflow-hidden">
            <div className="h-full bg-primary transition-all" style={{ width: `${state.percent}%` }} />
          </div>
          <Button
            type="button"
            variant="ghost"
            className="mt-2"
            onClick={(e) => { e.stopPropagation(); state.abort.abort(); }}
          >
            Cancel
          </Button>
        </div>
      )}

      {state.status === 'success' && (
        <div>
          <p className="text-green-700">Uploaded. Key: <code>{state.key}</code></p>
          <Button
            type="button"
            variant="ghost"
            className="mt-2"
            onClick={(e) => { e.stopPropagation(); setState({ status: 'idle' }); onChange(undefined); }}
          >
            Replace
          </Button>
        </div>
      )}

      {state.status === 'error' && (
        <p className="text-red-600">{state.message}</p>
      )}
    </div>
  );
}
```

## Wiring into the Form

The dropzone is a controlled field — it writes back via `onChange`:

```tsx
<FormField
  control={form.control}
  name="imageKey"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Image (optional)</FormLabel>
      <FormControl>
        <ImageDropzone value={field.value} onChange={field.onChange} />
      </FormControl>
      <FormMessage />
    </FormItem>
  )}
/>
```

When the form submits, `imageKey` is included in the JSON body alongside the rest. The backend stores the key; download URLs are generated on read.

## The MinIO CORS Gotcha

A direct browser PUT to MinIO will be blocked unless MinIO's bucket has a CORS policy allowing the dev frontend origin. Symptoms in the browser console:

```
Access to XMLHttpRequest at 'http://localhost:9000/questions-bucket/...'
from origin 'http://localhost:3000' has been blocked by CORS policy:
Response to preflight request doesn't pass access control check
```

Fix on the MinIO side (typically baked into the dev compose stack or set via `mc admin config`):

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["http://localhost:3000"],
      "AllowedMethods": ["GET", "PUT", "HEAD"],
      "AllowedHeaders": ["*"],
      "ExposeHeaders": ["ETag"],
      "MaxAgeSeconds": 3000
    }
  ]
}
```

For the PEP cohort, this should already be set on the shared dev MinIO. If you're running MinIO locally and hit this, the fix lives in `docker-compose.yml` or an `mc` init script — not in the Next.js app. Don't try to "fix CORS" by adding `mode: 'no-cors'` to fetch (it silently sends the request but you can't read the response and the upload still fails for signed PUTs).

## Example / Worked Scenario

A trainer authoring "What is the capital of France?" wants to attach `paris.jpg`:

1. They drag the file onto the dropzone — `onDrop` fires, file passes type/size validation.
2. `presignQuestionImage` POSTs to the gateway; backend returns `{ url, key: 'questions/01HX.../paris.jpg', contentType: 'image/jpeg', expiresInSeconds: 300 }`.
3. `putWithProgress` opens an XHR PUT to the signed URL; the dropzone re-renders at 0%, 12%, 47%, ... 100%.
4. XHR returns 200; state becomes `success`; `onChange('questions/01HX.../paris.jpg')` populates the form's `imageKey`.
5. Trainer fills the rest of the form and submits; the JSON payload includes `imageKey`; backend stores it on the question document.

If the trainer cancels mid-upload, `AbortController` aborts the XHR; state returns to `idle`; `onChange(undefined)` clears `imageKey` from the form.

## Common Pitfalls

- **Trying to use `fetch` for progress.** `fetch` cannot report upload progress in any major browser. Use `XMLHttpRequest` or a library that wraps it.
- **`Content-Type` mismatch with the signed URL.** The browser may default `Content-Type` to `application/octet-stream` when sending a `Blob`. Set the header explicitly to the value the backend signed.
- **Missing MinIO CORS configuration.** Causes preflight failures. Fix on MinIO, not on the frontend.
- **Sending the file through the backend "to be safe."** That throws away the whole point of pre-signed URLs — server bandwidth and memory pressure scale with cohort size. Trust the signed URL.
- **Not clearing `imageKey` on upload error.** Leaves a stale (broken) key in the form; submit succeeds with a phantom image. Always pair `onChange(undefined)` with the error branch.
- **Forgetting `e.preventDefault()` on `onDragOver`.** Without it, the browser navigates away to "open" the file.

## Key Takeaways
- Two-step flow: POST presign → PUT to signed URL. Bytes never touch your backend.
- Use `XMLHttpRequest` (not `fetch`) when you need upload progress; wrap it in a Promise + `AbortController`.
- Match the signed `Content-Type` exactly in the PUT request, or MinIO returns 403.
- The dropzone is a controlled RHF field that exposes only the resulting `imageKey` to the form.
- MinIO CORS must permit the frontend origin — that lives in MinIO config, not in your fetch options.

---
*Prerequisites: [07-frontend-to-backend-api-integration-patterns.md](07-frontend-to-backend-api-integration-patterns.md), day-8 MinIO pre-signed URL backend topic.*
