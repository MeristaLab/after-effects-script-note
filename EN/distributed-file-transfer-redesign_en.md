# Redesigning File Transfer Protocol in a Distributed System

## Background

While developing a distributed rendering system for After Effects, I noticed that the file transfer implementation uses different approaches for GET (download) and POST (upload).

```
GET  → Range: bytes=N-
POST → { chunk_index: 2, total_chunks: 10 }
```

Both were adopted as common practices, but I realized I had never questioned why these two methods differ. That prompted me to investigate.

## Why GET and POST Use Different Approaches

The Range header for GET is standardized in HTTP/1.1 (RFC 7233). It allows clients to request a specific byte range from the server as part of the protocol specification.

POST, on the other hand, has no equivalent standard for specifying the range of data being sent. As a result, each implementation independently developed chunked upload — splitting files and sending them sequentially. The asymmetry comes from the HTTP specification itself.

## The Problem with Large Files

Chunked upload is simple to implement and works well for smaller files. It is a widely used approach.

However, this system handles video files that can reach several GB or even tens of GB. At that scale, managing retries and resume logic on interruption requires significant custom implementation. Chunked upload is not optimal for this use case. Given that GET Range was designed with large files in mind, I investigated whether a better approach exists for POST as well.

## TUS Protocol

TUS protocol emerged as a candidate for large file uploads. It is an open protocol that standardizes resumable uploads and has proven implementations in production.

However, adopting TUS introduced several problems.

Python server implementations of TUS exist, but none are official reference implementations — they are community-maintained, raising concerns about long-term stability. The official reference implementation is tusd, written in Go. While stable, it would introduce a second language into an otherwise Python-only system.

Additionally, this system uses a distributed architecture with Gateway, Controller, Storage, and Node services. Applying TUS to internal communication between these services would add significant management overhead. Mapping tusd's upload IDs to the system's own job_ids would also require additional logic.

Adding TUS later on top of the existing chunked upload approach was considered, but the integration cost at that point would not be trivial.

## Conclusion: Unifying with Content-Range

The `Content-Range` header exists for POST requests in HTTP as well.

```
GET  → Range: bytes=N-
POST → Content-Range: bytes N-M/Total
```

This mirrors the GET Range approach and stays within HTTP standards. It requires no external dependencies and works across the distributed internal services without modification.

Implementing Content-Range from the start keeps debugging in our own hands. The implementation requires more effort than using a TUS library, but being an HTTP standard makes it more versatile than TUS and leaves the client-side implementation open to more options. This is the approach moving forward.