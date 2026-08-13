# Storage buckets

`insta services add storage <name>` gives the branch a **private, S3-compatible bucket**. There is
no vendor SDK and no InstaCloud storage client — point any S3 library at the injected credentials
and it works. This page is the part that isn't obvious: how bytes actually get in and out, and the
four things that bite first.

## What you get

Adding the service injects these into every compute service on the branch:

| Variable | Example |
| --- | --- |
| `BUCKET_NAME` | `insta-b5f7e21d-…-f1de1ba6` |
| `AWS_ACCESS_KEY_ID` | `tid_…` |
| `AWS_SECRET_ACCESS_KEY` | `tsec_…` |
| `AWS_ENDPOINT_URL_S3` | `https://t3.storage.dev` |
| `AWS_REGION` | `auto` |

With several storage services, each one's names are suffixed — `BUCKET_NAME_ASSETS`,
`AWS_ACCESS_KEY_ID_ASSETS`, … — and the **oldest** service of the type also gets the plain names, so
single-service projects work unchanged. Never bake these into an image; they arrive as env.

## Writing and reading objects

Nothing InstaCloud-specific. Two rules cover it: **set the endpoint explicitly**, and leave the
region as `auto`.

```js
// Node — @aws-sdk/client-s3
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'

const s3 = new S3Client({
  endpoint: process.env.AWS_ENDPOINT_URL_S3,   // required — without it the SDK talks to real AWS
  region: process.env.AWS_REGION,              // 'auto'
})                                             // key/secret come from AWS_* automatically

await s3.send(new PutObjectCommand({
  Bucket: process.env.BUCKET_NAME,
  Key: 'avatars/u1.png',
  Body: bytes,
  ContentType: 'image/png',
}))
```

```python
# Python — boto3
import boto3, os
s3 = boto3.client('s3', endpoint_url=os.environ['AWS_ENDPOINT_URL_S3'], region_name=os.environ['AWS_REGION'])
s3.put_object(Bucket=os.environ['BUCKET_NAME'], Key='avatars/u1.png', Body=data, ContentType='image/png')
```

The same credentials drive any S3 tool, which is the quickest way to seed or inspect a bucket:

```bash
aws s3 ls "s3://$BUCKET_NAME" --recursive --endpoint-url "$AWS_ENDPOINT_URL_S3"
aws s3 cp ./dist "s3://$BUCKET_NAME/dist" --recursive --endpoint-url "$AWS_ENDPOINT_URL_S3"
rclone copy ./dist insta:$BUCKET_NAME/dist       # with the same key/secret/endpoint configured
```

## The four traps

1. **Forgetting the endpoint.** An S3 client with credentials but no `endpoint` silently talks to
   real AWS and fails on a bucket that isn't yours. This is the single most common mistake.
2. **Assuming a bucket is shared across branches.** It is not. Each branch gets its **own** bucket
   (CoW-forked from the parent at `insta branch create`) with its **own** scoped key, so a leaked
   branch credential cannot reach production data. Read `BUCKET_NAME` from env per branch — never
   hardcode the name you saw once.
3. **Expecting to undelete.** There is no object versioning and no recycle bin. `insta storage
   delete` and any S3 `DeleteObject` are permanent, and deleting a key that was never there still
   reports success — so success is not proof the file existed.
4. **Expecting search.** S3 filters by **key prefix** only; there is no substring match, in the CLI,
   the console, or the API. Design keys so the prefix is the thing you will want to filter on
   (`avatars/2026/…`, not `2026-avatars-…`).

## Public vs private

Buckets are private by default: reads need the credentials or a presigned URL. Flip a bucket to
anonymous public-read with

```bash
insta services set-access storage <name> public   # or private
```

Public is a whole-bucket switch, not per-object. Prefer keeping it private and handing out presigned
URLs from your backend when only some files should be reachable.

## Browsing from outside your app

Once the object endpoints are in place, a bucket's contents are reachable without wiring up an S3
client:

```bash
insta storage list                      # keys, size, last modified (--prefix to filter, --cursor to page)
insta storage get <key> -o ./file       # via a short-lived presigned URL, bytes come straight from the provider
insta storage delete <key>              # immediate and irreversible
```

The console's storage service detail lists the same objects with per-row download and delete, and
agents have `insta_storage_list` / `insta_storage_download_url` / `insta_storage_delete` over MCP.
All of them are governed: `storage.read` for listing and download, `storage.delete` for removal (see
[governance.md](governance.md)).

**Uploading is not yet in the CLI, the console, or MCP** — put files in from your app or an S3 tool,
as above.
