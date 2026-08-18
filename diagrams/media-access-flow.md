# Media Access Flow

```mermaid
sequenceDiagram
    actor User
    participant App as PHP application
    participant Catalogue as Local S3-synchronized catalogue
    participant PG as PostgreSQL
    participant CDN as Media CDN

    User->>App: request protected episode
    App->>Catalogue: resolve published content metadata
    App->>PG: load account and entitlement
    alt active subscription or valid purchase
        App-->>User: temporary signed media URL
        User->>CDN: request HLS master playlist
        CDN-->>User: master and rendition playlists
        loop adaptive playback
            User->>CDN: request media segment
            CDN-->>User: cached/origin segment
        end
    else no entitlement
        App-->>User: access denied
    end
```

The application makes the access decision once and then leaves the media transfer path. Catalogue publication, entitlement, and CDN delivery remain distinct failure and observability boundaries.

[Back to README](../README.md)
