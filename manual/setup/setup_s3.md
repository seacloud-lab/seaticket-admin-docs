# S3 Storage

The standard [SeaTicket deployment](setup_seaticket.md) uses local disk for both attachments and SeaSearch data. No S3 service is required unless you select `s3` for either backend.

!!! warning "Changing storage backends"

    Changing a storage backend does not automatically move attachments or search data between backends. Back up existing data and plan its migration before switching an existing deployment.

## Use S3 as the storage backend

To store both attachments and SeaSearch data in S3, set the following in `/opt/seaticket/.env`:

```dotenv
ATTACHMENT_STORAGE_BACKEND=s3
# Leave SS_STORAGE_TYPE empty to follow ATTACHMENT_STORAGE_BACKEND.
SS_STORAGE_TYPE=
S3_HOST=https://s3-endpoint.example.com
S3_KEY_ID=your-access-key-id
S3_SECRET_KEY=your-secret-access-key
S3_FILE_BUCKET=seaticket-files
S3_WEB_CRAWL_BUCKET=seaticket-connections
S3_SEASEARCH_BUCKET=seaticket-search
```

Replace the example values with your actual endpoint, credentials, and bucket names. Create the buckets needed by the selected backends and grant the credentials read and write access to them. The components using S3 share the following endpoint and credentials:

| Variable | Description | Required when |
| --- | --- | --- |
| `S3_HOST` | S3-compatible storage endpoint. | Attachments or SeaSearch use S3 |
| `S3_KEY_ID` | S3 access key ID. | Attachments or SeaSearch use S3 |
| `S3_SECRET_KEY` | S3 secret access key. | Attachments or SeaSearch use S3 |
| `S3_FILE_BUCKET` | Bucket for uploaded files. | `ATTACHMENT_STORAGE_BACKEND=s3` |
| `S3_WEB_CRAWL_BUCKET` | Bucket for connection assets, such as email attachments. | `ATTACHMENT_STORAGE_BACKEND=s3` |
| `S3_SEASEARCH_BUCKET` | Bucket for SeaSearch data. Compose maps this to `S3_SS_BUCKET` unless that variable is explicitly set. | Resolved `SS_STORAGE_TYPE=s3` |

## Advanced S3 options

These options are normally left at their defaults:

| Variable | Description | Default |
| --- | --- | --- |
| `S3_AWS_REGION` | AWS region used by SeaSearch. | `us-east-1` |
| `S3_USE_V4_SIGNATURE` | Use Signature Version 4 for SeaSearch S3 requests. | `true` |
| `S3_PATH_STYLE_REQUEST` | Use path-style S3 requests in SeaSearch. | `true` |
| `S3_SSE_C_KEY` | Optional SeaSearch SSE-C encryption key. Requires HTTPS and Signature Version 4. | Empty |

These advanced options configure SeaSearch. The attachment backend currently uses Signature Version 4 and path-style requests directly; it does not read these four options.

## Use different storage backends

`SS_STORAGE_TYPE` accepts `disk` or `s3`. If unset or empty, it follows `ATTACHMENT_STORAGE_BACKEND`; if both are unset or empty, it uses `disk`.

Set `SS_STORAGE_TYPE` explicitly to use different backends:

| Attachment backend | SeaSearch backend | Settings | Required buckets |
| --- | --- | --- | --- |
| Disk | S3 | `ATTACHMENT_STORAGE_BACKEND=disk`, `SS_STORAGE_TYPE=s3` | `S3_SEASEARCH_BUCKET` |
| S3 | Disk | `ATTACHMENT_STORAGE_BACKEND=s3`, `SS_STORAGE_TYPE=disk` | `S3_FILE_BUCKET`, `S3_WEB_CRAWL_BUCKET` |

Both combinations require the shared S3 endpoint and credentials. For SeaSearch account and API configuration, see [SeaSearch](setup_seasearch.md).

After configuring storage, continue with [Start SeaTicket](setup_seaticket.md#start-seaticket). For an existing deployment, run `docker compose up -d` from `/opt/seaticket` to recreate affected containers after completing any required data migration.
