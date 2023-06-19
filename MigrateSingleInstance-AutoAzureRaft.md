# Description
Migrating a single node vault instance from a server A to Server B, both running linux. The instance running with the below configurations:
- auto-unseal using Azure Key Vault
- Raft storage
- tls 
- metrics  enabled to be consumed by promehteus

The goal of document this is just to keep record of some of the key artifacts that I created while I went through this migration process (just for learning purposes). I consider the main challenges were:
1. Come up with a good Dockerfile
2. Understanding the `raft restore` operation in vault
3. Understanding the Azure Key Vault auto-unseal process

### Come up with a good Dockerfile

This is always a challenge and a decision very specific to each project, in this case, I wanted to stick as much as possible to the official docker image, but also didn't want to have complex docker-compose file in case I migrate to any other orchestrator soon, as I am already considering this. In addition to this, since I am using TLS with Let's Encrypt certificates I wanted to have the flexibility of being able to script the frecuent renewal of the certificates. 

The solution was to create a simple Dockerfile on a private registry which would contain the main hcl file and read most of the values from environment variables. 


### Understanding auto-unseal and raft restore operations

This was the most time consuming task. At least for me with no experience working with these 2 things before, the documentation was not very clear. In my mind, I just wasn't able to connect these 2 operations, the disconnet was just trying to answer this question: 

> If the master key is on Azure and the backup/snapshot is using that key, how am I suppose to start vault in the new instance with that key if the data folder is empty? 

The answer was simple. Originally I thought vault was suppose to be initialized because it will point to a specific master key that already exist, then I would just need to find a command to restore the data, but how I would login if the new server has no access to the data folder? This is because we are not restoring an instance, we are migrating it, so the process is as follows:

- Start the new server
- Initialize it as if it was a brand new one, which would spill out a brand new root token and recovery keys (which in fact, both are  just temporary ones)
- Use that new root token to login and run the restore operation
- Your original root token and recovery keys take will take over and you can discard the temporary ones. 

That's it, the restore command seem to perform a good amount of operations so, its recommended to watch the logs and check the result carefully. 

### Source
- Dedicated VPS
- Running vault directly on the host
- Certbot to Manage certificates
- Azure authentication set in config file


### Destination
- VPS with Docker
- Custom Image on private registry
- CertBot to Manage certificates on the host   (soon the migration to docker)
- Azure authentication set in env Variables




# Docker Image

### Vault Configuration File

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
```


`vault status`

Sample Output:

```
Key                      Value
---                      -----
Recovery Seal Type       azurekeyvault
Initialized              false
Sealed                   true
Total Recovery Shares    0
Threshold                0
Unseal Progress          0/0
Unseal Nonce             n/a
Version                  1.12.2
Build Date               2022-11-23T12:53:46Z
Storage Type             raft
HA Enabled               true
```


`vault operator init`

Sample Output:
```
Recovery Key 1: rnQnl9ayEIDGmdM7qIb+zbgjAVLVKbD7deOLaEtZW1Zk
Recovery Key 2: yox8dsXY/IaLtHl/fBUudyQn1O3GfU49LYT1/F8/zdVv
Recovery Key 3: mv0+Nn94jv+U4SsWUjRKadxCBHuHc/ueL4r8kcr5A32R
Recovery Key 4: Btcql8vgrepfFeMVS61uMhmGQaRaOIiUgFIfq1TNteoO
Recovery Key 5: FPJpBIHvg74AEIXv0voHKdheTaitaCD1Q/ln6FGWUf3b

Initial Root Token: hvs.oIlIP4W7KcOjEDVDFz1Ymm5L

Success! Vault is initialized

Recovery key initialized with 5 key shares and a key threshold of 3. Please
securely distribute the key shares printed above.
```

`vault login`

> Enter the Root token from Above

`vault operator raft snapshot restore -force vault_backup_2023_01_09.snap`


After a few seconds the vault server is ready and you can login using any of the previously configured methods
