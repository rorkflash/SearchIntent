# Search Intent

**Search Intent** is an open-source microservice that converts natural-language search queries into structured API requests.

It sits between your search box and your backend API.

```text
User query
  ↓
Search Intent
  ↓
SearchObject
  ↓
ApiRequest
  ↓
Target Data API
```

Example:

```text
"Show me Harry Potter places in London"
```

becomes:

```json
{
  "intent": "search_places",
  "objects": ["place", "scene", "movie"],
  "entities": {
    "movie_title": ["Harry Potter"],
    "city": ["London"]
  },
  "resolved": {
    "movie_ids": [12],
    "city_ids": [44]
  },
  "api_request": {
    "method": "POST",
    "url": "https://api.example.com/search",
    "body": {
      "scope": ["place", "scene", "movie"],
      "filters": {
        "movie_ids": [12],
        "city_ids": [44]
      },
      "pagination": {
        "limit": 20,
        "offset": 0
      }
    }
  }
}
```

Search Intent is **not a search engine**.  
It is the missing layer between a natural-language search box and a strict backend API contract.

---

## Why Search Intent?

Most applications have the same problem:

```text
Users write messy natural language.
Backend APIs need strict JSON.
```

Search Intent solves this by converting open text into a predictable, validated, project-specific API request.

It can be used for:

- movie and filming-location search
- e-commerce search
- real estate search
- restaurant search
- travel search
- document search
- internal admin search
- marketplace search
- directory search

The core engine is project-agnostic.  
Each deployed service instance is configured for **one project/domain**.

---

## Core Principle

```text
One Search Intent instance = one project/domain/API contract
```

The open-source code is reusable, but a running service loads one active configuration:

```text
config/
  project.json
  extractor.json
  search-map.json
  intent-map.json
  search-object-schema.json
  api-map.json
  resolver-map.json
  cache.json
  auth-inbound.json
```

For example:

```text
MovieTrip Search Intent service
  → MovieTrip config
  → MovieTrip API

Shop Search Intent service
  → Shop config
  → Shop API

Real Estate Search Intent service
  → Real Estate config
  → Real Estate API
```

---

## What It Does

Search Intent converts:

```text
natural-language query
```

into:

```text
SearchIntentCommand
```

A `SearchIntentCommand` contains:

```json
{
  "query": "Harry Potter places in London",
  "locale": "en",
  "intent": "search_places",
  "objects": ["place", "scene", "movie"],
  "entities": {},
  "resolved": {},
  "filters": {},
  "api_request": {}
}
```

---

## High-Level Architecture

```text
Gateway / Backend
   ↓ JWT
Search Intent Service
   ↓
Inbound Auth Middleware
   ↓
Query Normalizer
   ↓
Extractor
   - GLiNER / GLiNER2-compatible extractor
   - optional regex extractor
   - optional plugin extractor
   ↓
Intent Detector
   ↓
SearchObject Builder
   ↓
Resolver Layer
   - static resolver
   - HTTP resolver
   - plugin resolver
   ↓
Cache Manager
   ↓
API Mapper
   ↓
ApiRequest
   ↓
optional Target API Executor
```

---

## Main Concepts

### 1. Search Map

Defines what business objects can be searched.

Example objects:

```json
{
  "movie": {
    "aliases": ["movie", "film"],
    "entities": ["movie_title", "genre", "person", "year"],
    "searchable_fields": ["title", "original_title", "description"]
  },
  "place": {
    "aliases": ["place", "location", "filming location"],
    "entities": ["place", "city", "country", "landmark"],
    "searchable_fields": ["name", "address", "description"]
  }
}
```

### 2. Extractor Config

Defines how natural-language entities and intent information are extracted.

The first supported extractor is intended to be GLiNER/GLiNER2-compatible, using JSON-defined labels and schemas.

```json
{
  "extractor": {
    "provider": "gliner2",
    "model": "fastino/gliner2-base-v1",
    "mode": "combined_schema",
    "thresholds": {
      "entity": 0.35,
      "classification": 0.45,
      "relation": 0.35
    }
  }
}
```

The core code should not hardcode labels.  
Labels come from JSON config:

```json
{
  "entities": {
    "movie_title": {
      "label": "movie title",
      "description": "Movie, film, franchise, or TV series title",
      "examples": ["Harry Potter", "The Godfather", "Interstellar"]
    },
    "city": {
      "label": "city",
      "description": "City or town name",
      "examples": ["London", "New York", "Paris"]
    }
  }
}
```

### 3. Intent Map

Defines possible search intents.

```json
{
  "intents": {
    "search_places": {
      "description": "User wants real-world places, filming locations, landmarks, streets, restaurants, or buildings connected to the domain.",
      "keywords": ["place", "places", "location", "locations", "filmed", "where was"],
      "default_objects": ["place", "scene", "movie"]
    },
    "search_scenes": {
      "description": "User wants specific scenes, moments, or sequences.",
      "keywords": ["scene", "scenes", "moment", "sequence"],
      "default_objects": ["scene", "movie", "place"]
    },
    "global_search": {
      "description": "General search across all searchable objects.",
      "keywords": [],
      "default_objects": ["movie", "scene", "place", "city", "country"]
    }
  }
}
```

### 4. Search Object Schema

Defines the normalized internal object that Search Intent builds before creating the final API request.

```json
{
  "schema": {
    "type": "object",
    "required": ["query", "locale", "intent", "objects"],
    "properties": {
      "query": { "type": "string" },
      "locale": { "type": "string" },
      "intent": { "type": "string" },
      "objects": {
        "type": "array",
        "items": { "type": "string" }
      },
      "entities": { "type": "object" },
      "resolved": { "type": "object" },
      "filters": { "type": "object" },
      "sort": { "type": "object" },
      "pagination": { "type": "object" }
    }
  },
  "defaults": {
    "sort": {
      "by": "relevance",
      "direction": "desc"
    },
    "pagination": {
      "limit": 20,
      "offset": 0
    }
  }
}
```

### 5. Resolver Map

Maps extracted text entities to real IDs in the target project.

Example:

```json
{
  "resolvers": {
    "movie_title": {
      "type": "http",
      "method": "GET",
      "url_from_env": "TARGET_MOVIE_RESOLVER_URL",
      "query": {
        "q": "{{ value }}"
      },
      "result_path": "$.items[*].id",
      "target": "movie_ids",
      "cache_ttl_seconds": 2592000
    },
    "city": {
      "type": "http",
      "method": "GET",
      "url_from_env": "TARGET_CITY_RESOLVER_URL",
      "query": {
        "q": "{{ value }}"
      },
      "result_path": "$.items[*].id",
      "target": "city_ids",
      "cache_ttl_seconds": 2592000
    }
  }
}
```

### 6. API Map

Maps the internal `SearchObject` to the target API request.

```json
{
  "api": {
    "base_url_from_env": "TARGET_API_BASE_URL",
    "mode": "generate_only",
    "auth": {
      "type": "bearer_token",
      "token_from_env": "TARGET_API_TOKEN"
    }
  },
  "endpoints": {
    "search": {
      "method": "POST",
      "path": "/api/search",
      "headers": {
        "Content-Type": "application/json"
      },
      "body": {
        "scope": "{{ search_object.objects }}",
        "filters": "{{ search_object.filters }}",
        "sort": "{{ search_object.sort }}",
        "pagination": "{{ search_object.pagination }}"
      },
      "options": {
        "remove_null_values": true,
        "remove_empty_arrays": true,
        "remove_empty_objects": true
      }
    }
  },
  "intent_endpoint_map": {
    "global_search": "search",
    "search_places": "search",
    "search_scenes": "search",
    "search_movies": "search"
  }
}
```

Supported API modes:

```text
generate_only
execute
generate_and_execute
```

---

## Authentication

Search Intent supports two different auth directions.

```text
Gateway → Search Intent
  inbound auth

Search Intent → Target Data API
  outbound API auth
```

### Inbound JWT Auth

Inbound auth validates requests coming from a gateway/backend.

Example `auth-inbound.json`:

```json
{
  "inbound_auth": {
    "enabled": true,
    "type": "jwt",
    "plugin": {
      "module": "plugins.auth.jwt_auth",
      "class": "JwtInboundAuthPlugin"
    },
    "token_source": {
      "type": "header",
      "name": "Authorization",
      "scheme": "Bearer"
    },
    "jwt": {
      "algorithm": "HS256",
      "secret_from_env": "SEARCH_INTENT_JWT_SECRET",
      "issuer_from_env": "SEARCH_INTENT_JWT_ISSUER",
      "audience_from_env": "SEARCH_INTENT_JWT_AUDIENCE",
      "required_claims": ["sub", "exp"]
    },
    "authorization": {
      "enabled": true,
      "required_permissions": ["search:intent:generate"]
    }
  }
}
```

For production, prefer asymmetric JWT validation with JWKS:

```json
{
  "jwt": {
    "algorithm": "RS256",
    "jwks_url_from_env": "SEARCH_INTENT_JWKS_URL",
    "issuer_from_env": "SEARCH_INTENT_JWT_ISSUER",
    "audience_from_env": "SEARCH_INTENT_JWT_AUDIENCE",
    "required_claims": ["sub", "iss", "aud", "exp"]
  }
}
```

### Outbound Target API Auth

Outbound auth is configured in `api-map.json`:

```json
{
  "api": {
    "auth": {
      "type": "bearer_token",
      "token_from_env": "TARGET_API_TOKEN"
    }
  }
}
```

Supported auth types should include:

```text
none
api_key
bearer_token
basic
custom_plugin
```

---

## Secrets and Environment Variables

Sensitive values must not be stored in JSON config.

Use `.env`, Docker secrets, Kubernetes secrets, or CI/CD environment variables.

The repository should include `.env.example`, but never a real `.env`.

Example:

```env
# App
SEARCH_INTENT_ENV=development
SEARCH_INTENT_HOST=0.0.0.0
SEARCH_INTENT_PORT=8080
SEARCH_INTENT_CONFIG_DIR=./config
SEARCH_INTENT_PLUGIN_DIR=./plugins

# Inbound JWT: Gateway → Search Intent
SEARCH_INTENT_JWT_SECRET=change_me_for_local_dev_only
SEARCH_INTENT_JWT_ISSUER=movietrip-gateway
SEARCH_INTENT_JWT_AUDIENCE=search-intent
SEARCH_INTENT_JWKS_URL=https://auth.example.com/.well-known/jwks.json

# Target API
TARGET_API_BASE_URL=https://api.example.com
TARGET_API_TOKEN=change_me
TARGET_API_KEY=change_me

# Cache
REDIS_URL=redis://localhost:6379/0

# Models
MODEL_CACHE_DIR=./models
GLINER2_MODEL=fastino/gliner2-base-v1
```

Recommended `.gitignore`:

```gitignore
.env
.env.*
!.env.example

models/
__pycache__/
.pytest_cache/
```

---

## Caching

Search Intent should cache the generated `SearchObject`, not only the final API response.

Why?

Because this part can be expensive:

```text
normalize
extract entities
classify intent
resolve entities
build SearchObject
build ApiRequest
```

Suggested cache layers:

```text
entities
resolved_entity
search_object
api_request
api_response
```

Example `cache.json`:

```json
{
  "cache": {
    "enabled": true,
    "backend": "redis",
    "namespace": "search-intent",
    "project_version": {
      "strategy": "static",
      "value": "v1"
    },
    "security_scope": {
      "include_tenant": true,
      "include_org": true,
      "include_user": false,
      "include_permissions_hash": true
    },
    "keys": {
      "search_object": {
        "pattern": "{namespace}:{project_version}:{locale}:{query_hash}:search_object",
        "ttl_seconds": 86400
      },
      "entities": {
        "pattern": "{namespace}:{project_version}:{locale}:{query_hash}:entities",
        "ttl_seconds": 604800
      },
      "resolved_entity": {
        "pattern": "{namespace}:{project_version}:{entity_key}:{value_hash}:resolved",
        "ttl_seconds": 2592000
      },
      "api_request": {
        "pattern": "{namespace}:{project_version}:{locale}:{query_hash}:api_request",
        "ttl_seconds": 86400
      },
      "api_response": {
        "pattern": "{namespace}:{project_version}:{locale}:{query_hash}:api_response",
        "ttl_seconds": 300
      }
    }
  }
}
```

---

## Plugins

JSON config should cover most use cases.

Python plugins are for advanced cases:

- custom JWT validation
- custom gateway auth
- custom API signing
- complex payload generation
- custom entity resolution
- custom extractor post-processing
- custom cache key logic
- response post-processing

Root-level plugins:

```text
plugins/
  auth/
    jwt_auth.py
    gateway_header_auth.py

  api/
    api_plugin.py
    signed_api.py

  resolver/
    resolver_plugin.py

  extractor/
    extractor_plugin.py

  cache/
    cache_plugin.py
```

Base plugin interfaces live inside the core package:

```text
search_intent/
  plugins/
    base_auth.py
    base_api.py
    base_resolver.py
    base_extractor.py
    base_cache.py
```

### Plugin Config Example

```json
{
  "plugin": {
    "enabled": true,
    "module": "plugins.api.api_plugin",
    "class": "ProjectApiPlugin"
  }
}
```

Plugins are trusted local code.  
Do not load arbitrary plugin paths from user requests.

---

## Suggested Project Structure

```text
search-intent/
  README.md
  LICENSE
  pyproject.toml
  Dockerfile
  docker-compose.yml
  .env.example

  search_intent/
    main.py

    api/
      routes_parse.py
      routes_generate.py
      routes_search.py
      routes_health.py

    core/
      pipeline.py
      normalizer.py
      intent_detector.py
      search_object_builder.py

    extractors/
      base.py
      gliner2_extractor.py
      regex_extractor.py

    mappers/
      api_mapper.py
      template_engine.py
      request_builder.py
      executor.py

    resolvers/
      base.py
      http_resolver.py
      static_resolver.py

    auth/
      base.py
      middleware.py
      jwt.py

    cache/
      manager.py
      redis_cache.py
      memory_cache.py

    config/
      loader.py
      validator.py

    plugins/
      base_auth.py
      base_api.py
      base_resolver.py
      base_extractor.py
      base_cache.py

  config/
    project.json
    extractor.json
    search-map.json
    intent-map.json
    search-object-schema.json
    api-map.json
    resolver-map.json
    cache.json
    auth-inbound.json

  plugins/
    auth/
      jwt_auth.py

    api/
      api_plugin.py

    resolver/
      resolver_plugin.py

    extractor/
      extractor_plugin.py

  examples/
    movietrip/
      config/
        project.json
        extractor.json
        search-map.json
        intent-map.json
        api-map.json
        resolver-map.json
        cache.json
        auth-inbound.json

    ecommerce/
      config/
        project.json
        extractor.json
        search-map.json
        intent-map.json
        api-map.json
```

---

## HTTP API

### `GET /health`

Health check.

Response:

```json
{
  "status": "ok"
}
```

### `POST /v1/parse`

Parses a query into intent and entities.

Request:

```json
{
  "query": "Harry Potter places in London",
  "locale": "en"
}
```

Response:

```json
{
  "query": "Harry Potter places in London",
  "locale": "en",
  "intent": "search_places",
  "objects": ["place", "scene", "movie"],
  "entities": {
    "movie_title": ["Harry Potter"],
    "city": ["London"]
  },
  "confidence": {
    "intent": 0.91,
    "entities": 0.87
  }
}
```

### `POST /v1/generate`

Generates the target API request.

Request:

```json
{
  "query": "Harry Potter places in London",
  "locale": "en"
}
```

Response:

```json
{
  "query": "Harry Potter places in London",
  "locale": "en",
  "intent": "search_places",
  "search_object": {
    "objects": ["place", "scene", "movie"],
    "entities": {
      "movie_title": ["Harry Potter"],
      "city": ["London"]
    },
    "resolved": {
      "movie_ids": [12],
      "city_ids": [44]
    },
    "filters": {
      "movie_ids": [12],
      "city_ids": [44]
    },
    "pagination": {
      "limit": 20,
      "offset": 0
    }
  },
  "api_request": {
    "method": "POST",
    "url": "https://api.example.com/api/search",
    "headers": {
      "Content-Type": "application/json"
    },
    "body": {
      "scope": ["place", "scene", "movie"],
      "filters": {
        "movie_ids": [12],
        "city_ids": [44]
      },
      "pagination": {
        "limit": 20,
        "offset": 0
      }
    }
  },
  "cache": {
    "hit": false
  }
}
```

### `POST /v1/search`

Optional endpoint.

Generates the API request and executes it against the configured target API when API mode allows execution.

Request:

```json
{
  "query": "Harry Potter places in London",
  "locale": "en"
}
```

Response:

```json
{
  "search_object": {},
  "api_request": {},
  "api_response": {}
}
```

---

## Local Development

Install dependencies:

```bash
pip install -e .
```

Run:

```bash
uvicorn search_intent.main:app --host 0.0.0.0 --port 8080 --reload
```

With Docker:

```bash
docker compose up --build
```

Test:

```bash
curl -X POST http://localhost:8080/v1/generate \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Harry Potter places in London",
    "locale": "en"
  }'
```

---

## MVP Roadmap

### v0.1.0

- FastAPI service
- single active project config
- JSON config loader
- GLiNER/GLiNER2-compatible extractor interface
- rule-based intent fallback
- SearchObject builder
- API mapper
- static resolver
- HTTP resolver
- Redis cache support
- inbound JWT plugin
- outbound API auth
- root-level plugin system
- MovieTrip example config
- e-commerce example config

### v0.2.0

- combined extraction schema
- structured extraction mode
- relation extraction support
- better confidence scoring
- resolver ranking
- OpenAPI docs
- config validation CLI
- plugin scaffolding CLI

### v0.3.0

- testing dashboard
- query evaluation suite
- multilingual normalization helpers
- analytics hooks
- response post-processing plugins
- advanced cache invalidation
- deployment templates

---

## Design Rules

1. The core engine must stay project-agnostic.
2. A running service instance should load one active project config.
3. Business meaning belongs in JSON config.
4. Secrets belong in environment variables.
5. Advanced logic belongs in trusted local plugins.
6. Search Intent generates API requests; it does not replace your search engine.
7. Entity resolution and API execution must be deterministic and validated.
8. Cache generated search objects for speed.
9. Do not load arbitrary plugins from user input.
10. Keep the target API contract explicit and inspectable.

---

## License

This project is intended to be released as open source.

Recommended license:

```text
Apache-2.0
```

or:

```text
MIT
```

Choose Apache-2.0 if you want clearer patent protection.  
Choose MIT if you want the simplest permissive license.

---

## Status

Early design / prototype stage.

The architecture is being designed around:

```text
natural-language query
  → SearchObject
  → ApiRequest
  → optional API execution
```

The first real-world use case is MovieTrip, but the engine is designed to be reusable for any project with a search API.
