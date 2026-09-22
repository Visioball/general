# Game HTTP API specification

**Author:** Daan Eggen  
**Version:** 0.2  
**Status:** Proposed prototype contract; not implemented or deployed.  
**Date:** 19/09/2026

## Overview

- This standalone specification defines four operations:
  `POST /games`, `GET /games/{id}`, `POST /audio` and `POST /image`.
- A game contains an ID, title, optional image URL, server creation timestamp
  and JSON definition. It represents a saved configuration, not a played session.
- The service base URL is deployment-specific. Examples use
  `<baseurl>` as a placeholder for the deployment URL, including its scheme.
  Substitute it before sending requests. Object delivery may use a different
  base URL; always use the actual URL returned by an upload.
- Hosted deployments use HTTPS; local development may use HTTP.
- No application authentication is defined. This contract is intended for a
  controlled demo with non-sensitive sample data. Reachable clients can create
  games, upload files and retrieve known IDs; IDs are not authorization credentials.
- Listing, updating, deleting and executing games are unsupported.

## Request and response conventions

- `POST /games` requires `Content-Type: application/json`; an optional `charset=utf-8`
  parameter is accepted. Encode JSON as UTF-8. Compressed requests are unsupported on all upload and game-create operations.
- File uploads use a raw binary body and the media type specified below, not
  JSON, base64 or multipart form data.
- Successful bodies use `Content-Type: application/json`. Error bodies use
  `Content-Type: application/problem+json`.
- All API responses include `Cache-Control: no-store`.
- Clients should accept both response media types. No other representation is
  offered, and clients must not require another response media type.
- The maximum game-create body is 65,536 bytes, including whitespace. Larger bodies
  return 413. Maximum nesting is 20 objects/arrays, counting the outer request
  object as level 1; scalar values do not increase depth.
- JSON must be syntactically valid, with no duplicate object keys at any depth.
  Duplicate keys and malformed JSON return 400.
- No query parameters are defined. GET has no request body.

## POST /audio and POST /image

- Each request uploads exactly one object file as raw bytes. No filename or
  metadata fields are accepted. The server generates a UUIDv4 object key and
  derives its extension from the validated media type, ignoring client filenames.
- The limits and formats below are proposed prototype defaults. Size is measured
  in body bytes; multipart overhead is irrelevant because multipart is unsupported.

| Endpoint      | Accepted Content-Type                  | Maximum body size         |
| ------------- | -------------------------------------- | ------------------------- |
| `POST /audio` | `audio/mpeg` (MP3), `audio/wav` (WAV)  | 20 MiB = 20,971,520 bytes |
| `POST /image` | `image/png` (PNG), `image/jpeg` (JPEG) | 5 MiB = 5,242,880 bytes   |

- Require one of the exact media types listed for that endpoint. Missing or
  unsupported types return 415; for example, SVG, JSON and multipart are rejected.
- Check size, actual file signature and parseability. Empty, corrupt or mismatched
  files return 422. Parsing must use bounded resources; deeper decoder limits
  must be agreed before implementation. No resizing or transcoding is performed.
- Persist the object before returning `201 Created`. The URL must immediately
  serve the uploaded bytes and remain usable across application restarts.
- Return `Location` equal to the returned absolute `url`. Success bodies use the
  schema below. API responses use `Cache-Control: no-store`; this does not specify
  the cache policy of the separately delivered object bytes.
- Every successful upload creates a new immutable object and URL. Identical files
  are not deduplicated. An upload never overwrites an existing object.
- URLs are stable HTTPS URLs, not expiring upload or download links. Anyone with
  a URL can read its bytes in this prototype. The storage/CDN serves files using
  their validated media type; no additional application GET-file route is defined.
- An upload does not create a game. Files remain stored if a later upload or game
  creation fails. The client retains successful URLs for retry. Deletion and
  automatic orphan cleanup are outside this API; agree on manual demo cleanup.
- Rejected uploads must not leave a published object. Storage failures return
  500/503 without claiming success; a lost response may still follow a completed
  upload, so an automatic retry can create a duplicate object.

| Response field | Type    | Rules                                                                            |
| -------------- | ------- | -------------------------------------------------------------------------------- |
| `url`          | string  | Absolute HTTPS read URL, no credentials or whitespace, maximum 2,048 characters. |
| `contentType`  | string  | Validated media type from the endpoint's allowlist.                              |
| `sizeBytes`    | integer | Stored file length, greater than zero and within the endpoint limit.             |

Example image upload (the bracketed body represents binary bytes):

```http
POST <baseurl>/image HTTP/1.1
Content-Type: image/png
Accept: application/json, application/problem+json

[PNG file bytes]
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-store
Location: <baseurl>/image/6ba7b810-9dad-41d1-80b4-00c04fd430c8.png

{
  "url": "<baseurl>/image/6ba7b810-9dad-41d1-80b4-00c04fd430c8.png",
  "contentType": "image/png",
  "sizeBytes": 24576
}
```

Example audio upload:

```http
POST <baseurl>/audio HTTP/1.1
Content-Type: audio/mpeg
Accept: application/json, application/problem+json

[MP3 file bytes]
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-store
Location: <baseurl>/audio/6ba7b811-9dad-41d1-80b4-00c04fd430c8.mp3

{
  "url": "<baseurl>/audio/6ba7b811-9dad-41d1-80b4-00c04fd430c8.mp3",
  "contentType": "audio/mpeg",
  "sizeBytes": 153600
}
```

The client then places these URLs in the `POST /games` example below. The base URL
and object paths are illustrative; clients must use each returned URL verbatim.

## Create payload

The `POST /games` body must be an object with the fields below. Unknown outer fields,
including client-supplied `id` or `createdAt`, return 422.

| Field        | Type           | Required | Rules                                                                                                                                                                 |
| ------------ | -------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `title`      | string         | Yes      | 1–120 Unicode code points and at least one non-whitespace character. Stored exactly as supplied; no trimming.                                                         |
| `imageUrl`   | string or null | No       | Omitted or null means no image. Otherwise an absolute HTTPS URL with a hostname, no credentials, no whitespace and at most 2,048 characters. Empty string is invalid. |
| `definition` | object         | Yes      | Any JSON object, including `{}`. Nested JSON values are allowed within body and depth limits. An array, string or null at this field is invalid.                      |

- Use the `url` returned by `POST /image` as `imageUrl`. Audio upload URLs go
  inside `definition`; this document uses `definition.audioUrls` as an example
  convention. There is no top-level `audioUrl` or `audioUrls` game field.
- Game creation validates `imageUrl` syntax only. It does not fetch the URL or
  check ownership, existence or media type. External HTTPS image URLs remain
  accepted. Audio URLs inside the opaque definition are not separately validated.
- Upload validation is separate from game validation: uploaded bytes are checked
  when the file is received, not when its URL is submitted in a game.
- The server does not interpret `definition` or validate game rules. Consumers
  must validate it before applying settings to hardware.
- Returned JSON preserves values, not byte formatting or object-key order.

## Game response

Every successful game POST or GET returns the same complete game shape.

| Field        | Type           | Rules                                                                                                                             |
| ------------ | -------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `id`         | string         | Server-generated UUIDv4, lowercase canonical hyphenated form.                                                                     |
| `title`      | string         | Validated title from the create request.                                                                                          |
| `imageUrl`   | string or null | Supplied URL, or null when omitted/null in the request. Always present in responses.                                              |
| `createdAt`  | string         | Server creation time in UTC, exactly `YYYY-MM-DDTHH:mm:ss.SSSZ`, e.g. `2026-09-19T12:00:00.000Z`. Remains unchanged on retrieval. |
| `definition` | object         | Stored JSON definition.                                                                                                           |

IDs follow [UUIDv4 in RFC 9562](https://www.rfc-editor.org/rfc/rfc9562.html#section-5.4).
Creation uses secure randomness and enforces uniqueness; a collision must not
replace an existing game. Timestamp values are creation metadata, not scheduling
or precise ordering guarantees.

## POST /games

- Validate the complete request before storing anything.
- Assign the UUID and creation time on the server. Persist the full game before
  returning success, so it is immediately retrievable and survives app restarts.
- Return `201 Created`, the complete game body and a `Location` header containing
  the relative resource path `/games/{id}`.
- Each accepted request creates a new game, including repeated identical requests.
  No deduplication or idempotency-key mechanism is defined. If the response is
  lost, the client may not know whether creation succeeded; retrying may create
  a duplicate. There is no search operation to resolve that uncertainty.

Example request (`definition` keys are illustrative, not a game-rule schema):

```http
POST <baseurl>/games HTTP/1.1
Content-Type: application/json
Accept: application/json, application/problem+json

{
  "title": "Follow the sound",
  "imageUrl": "<baseurl>/image/6ba7b810-9dad-41d1-80b4-00c04fd430c8.png",
  "definition": {
    "mode": "followSound",
    "durationSeconds": 60,
    "audioUrls": ["<baseurl>/audio/6ba7b811-9dad-41d1-80b4-00c04fd430c8.mp3"],
    "settings": { "sound": "bell" }
  }
}
```

Example response:

```http
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-store
Location: /games/550e8400-e29b-41d4-a716-446655440000

{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "Follow the sound",
  "imageUrl": "<baseurl>/image/6ba7b810-9dad-41d1-80b4-00c04fd430c8.png",
  "createdAt": "2026-09-19T12:00:00.000Z",
  "definition": {
    "mode": "followSound",
    "durationSeconds": 60,
    "audioUrls": ["<baseurl>/audio/6ba7b811-9dad-41d1-80b4-00c04fd430c8.mp3"],
    "settings": { "sound": "bell" }
  }
}
```

Minimal valid request:

```json
{
  "title": "Demo game",
  "definition": {}
}
```

Its response includes `imageUrl: null` as well as the server-generated fields.

## GET /games/{id}

- Accept a UUIDv4 in canonical hyphenated form, with hexadecimal letters in either
  case. Normalize case for lookup; always return lowercase IDs.
- The path ID must match
  `^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-4[0-9a-fA-F]{3}-[89aAbB][0-9a-fA-F]{3}-[0-9a-fA-F]{12}$`.
- A malformed ID returns 400. A syntactically valid but unknown ID returns 404.
- Return `200 OK` and the complete game when found. Retrieval does not change
  the record or its timestamp. The body matches the POST response for that ID.

Example request:

```http
GET <baseurl>/games/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1
Accept: application/json, application/problem+json
```

Example response for the minimal game:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "Demo game",
  "imageUrl": null,
  "createdAt": "2026-09-19T12:00:00.000Z",
  "definition": {}
}
```

## Errors

Errors follow [RFC 9457 problem details](https://www.rfc-editor.org/rfc/rfc9457.html).
The body contains `type` (always `about:blank`), `title` (HTTP status phrase),
`status` (integer matching the HTTP status) and `detail` (a readable explanation).
Validation errors also include `errors`, an array of objects with `field` (JSON
Pointer for game JSON, with the empty string referring to the whole body;
use the empty string for a binary upload error) and `message`.
Do not return stack traces, storage internals or complete submitted definitions.
Clients should branch on status and field, not exact message wording.

| Status                     | Applies to          | Meaning                                                                                                              |
| -------------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------- |
| 400 Bad Request            | POST, GET           | Malformed JSON, duplicate JSON keys, or invalid path ID.                                                             |
| 404 Not Found              | GET                 | Valid ID does not exist.                                                                                             |
| 413 Content Too Large      | All POST operations | Body exceeds the limit for that endpoint.                                                                            |
| 415 Unsupported Media Type | All POST operations | Missing/unsupported Content-Type or unsupported Content-Encoding.                                                    |
| 422 Unprocessable Content  | All POST operations | Game JSON violates field/depth rules, or uploaded bytes are empty, corrupt or inconsistent with the declared format. |
| 500 Internal Server Error  | All operations      | Unexpected server failure.                                                                                           |
| 503 Service Unavailable    | All operations      | Temporary service/storage unavailability. No success is claimed.                                                     |

For a game POST with multiple problems, check media type/encoding, size, JSON syntax,
then field/depth validation in that order. A rejected validation request creates
no record. A failed persistence operation must not leave a partially saved game.
Connection loss can still leave the client uncertain about a completed write.
For uploads, check media type/encoding, size, then file validity in that order.

Example validation error:

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json
Cache-Control: no-store

{
  "type": "about:blank",
  "title": "Unprocessable Content",
  "status": 422,
  "detail": "The request contains invalid fields.",
  "errors": [
    { "field": "/title", "message": "Must contain a non-whitespace character." }
  ]
}
```

Example missing-game error:

```json
{
  "type": "about:blank",
  "title": "Not Found",
  "status": 404,
  "detail": "No game exists with this ID."
}
```

## Acceptance checks

These are planned checks for the implementation, not recorded test results.

- Upload each supported file type: receive 201, matching `Location`/`url`, correct
  `contentType` and byte length; download the URL and compare bytes and media type.
- Upload an image and audio, submit their returned URLs in a game, then GET the
  game and verify that both references are preserved exactly.
- Check upload sizes at each limit and one byte over, empty/corrupt files,
  mismatched media types and unsupported formats. Confirm the defined errors
  and absence of published objects for rejected requests.
- Repeat an upload: receive different URLs. Fail game creation after an upload:
  confirm the successful file URL still works and can be reused in a corrected POST.
- Restart the application after uploads and confirm their URLs remain usable.
- Create with all fields: receive 201, a UUIDv4, UTC timestamp, matching data
  and `Location`; immediately GET that path and compare the complete record.
- Create without an image and with a null image: both responses contain null.
- Retrieve the same ID in uppercase: receive the same lowercase ID and record.
- Submit two identical valid requests: receive two distinct IDs.
- Reject missing/blank/overlong title, invalid image URL, non-object definition,
  unknown outer fields, and client-supplied metadata with 422 and no new record.
- Check title lengths 120 and 121, body sizes 65,536 and 65,537 bytes, and nesting
  levels 20 and 21. Valid boundary inputs succeed; the next value fails as defined.
- Check malformed JSON and duplicate keys (400), unsupported content type (415),
  malformed ID (400), and a valid unknown UUIDv4 (404).
- Simulate a failed write: no 201 and no partially stored record. Restart the
  application after a successful create and confirm the game remains retrievable.
- Check success and error media types, `Cache-Control`, POST `Location` and error
  fields. A missing external image must not cause the game's GET to fail.

## Compatibility and deployment

- Paths are unversioned for this single prototype. Coordinate any breaking
  contract change with consumers and increment this document's version before
  implementation. A future public API may need a versioning policy.
- No hosting provider, framework, database or object-storage service is prescribed.
- Browser clients on a different origin may require deployment-specific CORS
  and OPTIONS handling. These are transport configuration, not additional game
  operations, and must be agreed when the client origin is known.
- HTTP operation and status semantics are based on
  [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html).
