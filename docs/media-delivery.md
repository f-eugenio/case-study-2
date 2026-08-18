# Media Delivery

## Objective

The platform separates two questions:

1. may this account watch the requested content?
2. how should the video bytes reach the player?

The PHP backend owns authorization. An HLS-capable CDN owns delivery. EC2 does not proxy playlists or segments.

See the [media access flow](../diagrams/media-access-flow.md).

## Catalogue read path

Browse pages load a hierarchical JSON projection of topics, modules, episodes, and authors. The files are published to S3 and synchronized into the active release during deployment.

This keeps common catalogue navigation inexpensive:

- no repeated multi-table joins for every browse page;
- simple filesystem reads close to the PHP process;
- one published projection can serve many anonymous requests.

Search follows a different path in PostgreSQL because it needs full-text parsing, fuzzy matching, deterministic relevance ordering, and bounded pagination.

## Entitlement path

The authenticated episode request resolves content metadata and checks PostgreSQL for either:

- an active subscription; or
- a valid one-time purchase covering the requested content.

Only after that decision does the backend mint temporary CDN access. The signing material and entitlement logic remain server-side.

Temporary access limits the useful lifetime of a copied URL, but it is one control rather than a complete content-protection system. Security also depends on correct CDN origin policy, TLS, path scope, expiry, and avoiding sensitive URL logging.

## Delivery path

Once authorized:

1. the browser initializes the video player with the temporary master-playlist URL;
2. the CDN returns the HLS master playlist;
3. the player selects an appropriate rendition;
4. playlists and segments continue directly between browser and CDN;
5. the PHP application is no longer on the media transfer path.

The player consumes adaptive HLS renditions, including modern codec outputs. Managed transcoding and object publication run upstream; this service owns authorization and playback delivery.

## Why this boundary matters

### Performance

Adaptive renditions let the player react to network and device conditions. CDN edge delivery reduces origin distance and does not occupy PHP workers with long-lived segment responses.

### Cost

Proxying video through EC2 would scale application bandwidth, connection count, and worker pressure with every segment. Direct CDN delivery moves that work to infrastructure designed for object caching and media transfer.

### Failure isolation

Application and delivery failures are partially decoupled. Once a temporary URL has been issued, a normal application release does not sit between the player and each segment. A CDN or origin failure still prevents playback and needs separate observability.

## Publication consistency

The platform has two catalogue projections:

- S3/JSON for browse reads;
- PostgreSQL for search and user-specific state.

They are not one transaction. Deployment synchronizes the JSON projection with exact timestamps and removes objects no longer present in S3. Operational controls should include schema validation, a publication version, and a last-known-good manifest to prevent a malformed batch from affecting the full browse surface.

## Search workload

PostgreSQL search spans episodes, modules, topics, and authors. Weighted fields, full-text parsing, fuzzy similarity, stable ordering, and bounded windows support a useful catalogue search without a separate OpenSearch domain.

The trade-off is shared capacity: search consumes the same relational resources as entitlement and billing. Query latency and database CPU should determine when a dedicated search projection becomes justified.

## Operational signals

The useful indicators are split by boundary:

| Boundary | Signals |
|---|---|
| Authorization | denial reason, entitlement lookup latency, signed-URL issuance failures |
| Catalogue | publication version, sync duration, missing/malformed object count |
| CDN | startup time, playlist errors, segment error rate, cache ratio, egress |
| Player | rendition changes, buffering, terminal playback failures |

These signals let operations distinguish an account-policy problem from a catalogue publication problem or a delivery problem.

[Back to README](../README.md)
