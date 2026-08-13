# Design Sprint Transcript (front half, timeboxed)
**Date:** 2026-08-13
**Start Time:** 17:04:21 · **End Time:** 17:22:16
**Problem:** File Storage & Sync (Dropbox)
**Front-half readiness: 3/5**
**Front half complete inside 17:00: no** — all three phases delivered, but the API landed at 17:55, 55s past the buzzer.

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 8:00 | 9:13 | +1:13 | 1 | 3/5 |
| Core entities | 12:00 | 11:58 | −0:02 | 1 | 3/5 |
| API design | 17:00 | 17:55 | +0:55 | 1 | 4/5 |

**Budget allocation:** 9:13 requirements / 2:45 entities / 5:57 API. Requirements borrowed 1:13; entities gave it back by finishing in under three minutes; the API absorbed the remaining debt and ran 55s past the hard stop. The overrun is entirely inherited from requirements — every later phase ran at or under its own allowance.

**First-pass completeness:** One message per phase, no clarifying questions, no back-fill into an earlier phase. Single-shot throughout. The gaps are omissions from the first pass, not things that needed a second prompt.

**Plausibility check:** Not performed. The 100kB average file size is the load-bearing assumption of the whole design — it produces the 200 MB/s bandwidth figure — and it was never tested. For a Dropbox-shaped product it is low by roughly an order of magnitude; at 1MB the number becomes 2 GB/s, which changes the upload path materially. One line ("100kB feels low for a file-sync product — call it 1MB and we're at 2 GB/s") was all that was required. Caps Requirements at 3.

## What he produced (verbatim, as it stood at 17:00)

### Requirements
```
FRs ->
1. Users can upload files of arbitrary sizes from multiple devices.
2. Users can share files with others
3. Changes made to a file on one device propagate to other devices.

Out of scope ->
1. Tracking history of edits to document , versioning.
2. Analytics on documents.

NFRs ->
1. Highly available for file uploads (99.9~8.6 hrs downtime/year)
2. Eventual consistency acceptable for file uploads.
3. Latency of file upload as low was possible according to file size.
4. Support file upload for large files , over irregular network.
5. Changes to file propagated within 1 min.
6. Assuming 100M DAU , each user uploading 2 files/day -> 2k file uploads/s.
   Read:write ratio for files of 10:1 so 20k file reads/s.
   Assuming an avg size of 100kB for a file -> 200 MB/s of file data.
```

### Core entities
```
1. File Bytes (stored in object storage)
2. FileMetadata (fileBytesS3Url, id, userId, updatedAt, createdAt,
   status: PENDING_UPLOAD | UPLOADED)
3. FileShare (id, fileId, sharedByUserId, sharedWithUserId)
4. User (id, name, email)
```

### API design
```
NOTE: Identity extracted from JWT auth header

POST /files
Request -> {fileSize, filename, ..}
Response -> File(id, status: PENDING, fileS3Url: presigned s3 Url which can be used
  by client to upload media bytes to s3, s3UploadSessionId: for resumable multi part
  s3 uploads)
Idempotency key header to deduplicate client side retries.

PUT /files/:id/completeUpload
Response -> 2xx status indicating file upload completion to update status in BE.
  (Belts and suspenders)

PUT /files/:id/share/:userId
Response: 2xx status indicating file shared with other user.

GET /files/:id
Response: File(id, fileS3Url, status, updatedAt)

GET /files?changedSince={}
Response: File(id, fileS3Url, updatedAt)[]
```

## What was still missing at 17:00

**Requirements:**
- No latency percentile anywhere. NFR 3 ("as low as possible according to file size") is a wish, not a target.
- No storage growth figure (200 MB/s → ~17 TB/day was one multiplication away).
- Traffic model asserted without a sanity check; 100kB avg file size is the weak link.
- No consistency posture for *sharing* (only for uploads); no availability target for the read/sync path.

**Core entities:**
- **No chunk/block entity** — the object the "arbitrary size over irregular network" FR exists for. He designed resumable multipart upload in the API but never modelled the chunk.
- No `Device` entity, despite multi-device sync being FR 3 and devices needing independent sync positions.
- No sync cursor / version / change-log entity to make `changedSince` reliable.
- No keys or uniqueness constraints (`FileShare` needs unique `(fileId, sharedWithUserId)`).
- No permission level on `FileShare` (view vs edit).

**API design:**
- **No `nextCursor` on `GET /files?changedSince=`** — load-bearing, since that endpoint is the sync path and will page.
- No DELETE on file or on share; no unshare.
- `PUT /files/:id/share/:userId` carries no permission level and returns nothing useful.
- No error/status semantics (409 on conflicting write, 410 on expired presigned URL, 413 on size).
- `changedSince` as a wall-clock timestamp has a clock-skew/missed-update problem a version cursor avoids.
- No idempotency on the share endpoint (PUT is naturally idempotent here, but it was never stated).

## Where the time went
Requirements ran 1:13 long on volume — six NFRs written as prose, including one that carries no number at all. Entities were fast and disciplined (2:45). The API was the phase that deserved the borrowed time and got the leftovers, finishing 55s past the hard stop. Nothing was lost to round-trips or clarifying questions; the whole overrun is front-loaded.

## Feedback given
Strongest front half of the drill so far, and the first with a real out-of-scope list. The API in particular reads like a designed contract — presigned URLs rather than proxied bytes, a named upload-session id for resumability, idempotency on the create, and a completion call whose purpose he articulated. What's still costing him is the sanity check (never performed, on the one assumption that mattered) and the missing chunk entity, which left the API describing a mechanism the data model doesn't contain.
