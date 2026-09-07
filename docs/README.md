# 아키텍처

```text
                    ANDROID
                       │
                       │
               Capture Session
                       │
                       ▼
             Local Raw Session
                       │
                       │ multipart
                       ▼
              HTTP/HTTPS Upload
                       │
                       ▼
                    NGINX
                       │
                       ▼
                   FASTAPI
                       │
            integrity validation
                       │
                       ▼
                /srv/incoming
                       │
                 atomic publish
                       │
                       ▼
                /srv/sessions
                  RAW / IMMUTABLE
                       │
                       ▼
              CPU PREPROCESSOR
                       │
        ┌──────────────┼───────────────┐
        │              │               │
    timestamp       camera→EE       quality
     alignment       transform       filtering
        │              │               │
        └──────────────┼───────────────┘
                       │
                episode normalize
                       │
                delta action
                       │
                       ▼
            /srv/processed/v1
                       │
                     READY
                       │
                       ▼
                 GPU SERVER
                       │
                   DataLoader
                       │
                  Policy Model
```

