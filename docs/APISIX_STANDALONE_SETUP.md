# APISIX Standalone Configuration with File-Based Routes

## Overview

This implementation configures APISIX in **standalone mode** to load routes from static YAML files without requiring the Admin API or etcd dependency.

## Configuration Changes

### 1. Docker Compose Setup
**File:** `example/docker-compose.yml`

- **Config Mount:** Uses `config-standalone.yaml` instead of `config.yaml`
  - Path: `./apisix_conf/config-standalone.yaml:/usr/local/apisix/conf/config.yaml:ro`
  
- **Routes Mount:** `routes.yaml` is mounted as APISIX configuration file
  - Path: `./apisix_conf/routes.yaml:/usr/local/apisix/conf/apisix.yaml:ro`

- **Environment:** `APISIX_STAND_ALONE=true`
  - Enables standalone mode for APISIX

- **Removed etcd Dependency:** No longer requires etcd service for APISIX

### 2. APISIX Configuration
**File:** `apisix_conf/config-standalone.yaml`

```yaml
apisix:
  node_listen: 9080
  enable_ipv6: false

deployment:
  role: data_plane
  role_data_plane:
    config_provider: yaml
```

**Key Settings:**
- `config_provider: yaml` - Loads routes from YAML files instead of etcd
- `role: data_plane` - APISIX runs as data plane with yaml provider

### 3. Routes File
**File:** `apisix_conf/routes.yaml`

Routes are defined in standard APISIX format with required `#END` marker:

```yaml
routes:
  - id: 1
    uri: /web1/*
    upstream:
      type: roundrobin
      nodes:
        - host: web1
          port: 80
          weight: 1

  - id: 2
    uri: /web2/*
    upstream:
      type: roundrobin
      nodes:
        - host: web2
          port: 80
          weight: 1

#END
```

**Important:** The `#END` marker is required at the end of the file.

## Testing

### Start Services
```bash
cd example
docker compose up -d
```

### Test Routes
```bash
# Route to web1
curl http://localhost:9080/web1/
# Response: hello web1

# Route to web2
curl http://localhost:9080/web2/
# Response: hello web2
```

### Direct Service Access (Bypasses APISIX)
```bash
# Direct web1 access
curl http://localhost:9081

# Direct web2 access
curl http://localhost:9082
```

## Port Mapping

| Service | Internal | External | Purpose |
|---------|----------|----------|---------|
| APISIX Gateway | 9080 | 9080 | Main gateway endpoint |
| APISIX Control | 9092 | 9092 | Control API (disabled in standalone) |
| Prometheus | 9091 | 9091 | Metrics export |
| web1 | 80 | 9081 | Direct upstream access |
| web2 | 80 | 9082 | Direct upstream access |

## Advantages

✅ **No Admin API Required** - Routes managed via files, not API calls
✅ **No etcd Dependency** - Simpler infrastructure, less services to manage
✅ **Version Control** - Routes tracked in git
✅ **Hot Reload** - APISIX automatically reloads `apisix.yaml` on changes
✅ **Simple Setup** - No API key management needed

## Modifying Routes

1. Edit `apisix_conf/routes.yaml`
2. Add or modify route definitions
3. APISIX automatically detects changes and reloads
4. Test with curl: `curl http://localhost:9080/<path>`

Example - Add new route:
```yaml
routes:
  - id: 3
    uri: /new-service/*
    upstream:
      type: roundrobin
      nodes:
        - host: new-service
          port: 8080
          weight: 1
```

## Environment Variables

- `APISIX_STAND_ALONE=true` - Enable standalone mode
- `APISIX_IMAGE_TAG=3.16.0-debian` - APISIX version (configurable)

## Upstream Services

Services accessed via routes must be on the same docker network (`apisix`):

- `web1` - nginx on port 80
- `web2` - nginx on port 80

Routes reference services by hostname (container name) and internal port.
