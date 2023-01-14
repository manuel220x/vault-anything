# Description
Migrating a single instance configured with auto-unseal using Azure Key Vault and Raft storage, tls enabled and enabled metrics to be consumed by promehteus.


# Source
- OnPrem Server
- Running vault directly on the host
- TLS Enabled
- Certbot to Manage certificates
- AKV Configured with credentials set in the config file


# Destination
- Docker
- Custom Image on private registry



# Docker Image

### Configuration File

```hcl
ui = true

listener "tcp" {
  address       = "0.0.0.0:8300"
  cluster_address = "0.0.0.0:8301"
  tls_cert_file = "/vault/certs/fullchain.pem"
  tls_key_file  = "/vault/certs/privkey.pem"
  telemetry {
    unauthenticated_metrics_access = true
  }
}

storage "raft" {
  path = "/vault/raft"
  node_id = "vault_main"
}

seal "azurekeyvault" {
}

api_addr = "http://127.0.0.1:8300"
cluster_addr = "http://127.0.0.1:8301"

log_level = "Error"

telemetry {
  disable_hostname = true
  prometheus_retention_time = "12h"
}
```

### Dockerfile (WIP)

```dockerfile
FROM vault:1.12.2

COPY vault.hcl /vault/config/vault.hcl

VOLUME /vault/certs
VOLUME /vault/raft

CMD ["server", "-config=/vault/config/vault.hcl"]
```

### Docker Compose File

```yml
version: '3'

services:
  vault:
    image: registry.mamado.com/vault/vault:latest
    ports:
      - "8200:8300"
    volumes:
      - /vault/certs:/vault/certs
      - /vault/raft:/vault/raft
    environment:
      - AZURE_TENANT_ID
      - AZURE_CLIENT_ID
      - AZURE_CLIENT_SECRET
      - VAULT_AZUREKEYVAULT_VAULT_NAME
      - VAULT_AZUREKEYVAULT_KEY_NAME
    cap_add:
      - IPC_LOCK
    entrypoint: vault server -config=/vault/config/vault.hcl
```


# Migration

### 1. Initialize vault:

```bash
export VAULT_SKIP_VERIFY=1
export VAULT_ADDR='https://localhost:8200'
