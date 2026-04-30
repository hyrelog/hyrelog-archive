# Local avatar storage with MinIO (S3-compatible)

Profile avatar uploads use S3-compatible storage. For local development we use **MinIO** via Docker. In production you’ll use real S3 (e.g. AWS) and set the same env vars accordingly.

## 1. Start MinIO with Docker

**Recommended: use the HyreLog API stack (one Docker Compose for everything)**

From the **hyrelog-api** repo:

```bash
cd path/to/hyrelog-api
docker compose up -d
```

This starts Postgres instances, **MinIO**, and a `minio-init` job that creates the bucket `hyrelog-avatars` with public read so avatar image URLs work. MinIO is on the same ports as below.

- **S3 API:** http://localhost:9000  
- **MinIO Console:** http://localhost:9001 (login: `minioadmin` / `minioadmin`)

**Alternative (dashboard-only):** if you are not running the API stack and want MinIO by itself, from the **dashboard** repo run:

```bash
docker compose -f docker-compose.minio.yml up -d
```

Use the same `.env` values below. If 9000/9001 are already in use, edit `docker-compose.minio.yml` to map different host ports (e.g. `9002:9000`, `9003:9001`) and set `S3_ENDPOINT` and `S3_PUBLIC_BASE_URL` to `http://localhost:9002`.

## 2. Add env vars to `.env`

Append (or merge) into your `.env`:

```env
# S3-compatible storage (MinIO for local dev; AWS S3 in production)
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=hyrelog-avatars
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_REGION=us-east-1
# URL used for stored object links (browser must be able to load this)
S3_PUBLIC_BASE_URL=http://localhost:9000
```

- **Local:** Keep `S3_ENDPOINT` and `S3_PUBLIC_BASE_URL` as above so the app and the browser both use MinIO.
- **Production:** Omit `S3_ENDPOINT` (or use your S3 endpoint), set `S3_ACCESS_KEY` / `S3_SECRET_KEY` to IAM credentials, and set `S3_PUBLIC_BASE_URL` to your CDN or bucket public URL (e.g. `https://your-cdn.example.com` or the S3 bucket URL).

## 3. Restart the dashboard

Restart the Next.js app so it picks up the new env vars. Profile → Upload photo should then upload to MinIO and the avatar URL will point to `http://localhost:9000/hyrelog-avatars/avatars/...`.

## 4. Optional: check MinIO

- Open http://localhost:9001 and log in with `minioadmin` / `minioadmin`.
- Confirm the bucket `hyrelog-avatars` exists and that objects appear after an upload.

## Production

- Use AWS S3 (or another S3-compatible provider).
- Set `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_REGION`.
- Set `S3_PUBLIC_BASE_URL` to the base URL where objects are publicly readable (e.g. CloudFront or bucket website URL). Do **not** set `S3_ENDPOINT` for AWS.
- Ensure the bucket policy or IAM allows `PutObject` (and optionally `GetObject` if you use signed URLs; for public read you’ll use a CDN or public bucket URL).
