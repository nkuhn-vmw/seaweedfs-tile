# SeaweedFS Tanzu Tile

Tanzu Operations Manager tile for [SeaweedFS](https://github.com/seaweedfs/seaweedfs) distributed object storage. Deploys a fully managed S3-compatible storage service on the Tanzu platform.

## Features

- **S3-Compatible Object Storage**: SeaweedFS cluster with master, volume, filer, and S3 gateway
- **Service Broker**: Open Service Broker API for Cloud Foundry marketplace integration
- **Per-Binding IAM Credentials**: Each `cf bind-service` creates isolated IAM users with dedicated access keys via SeaweedFS's embedded IAM API
- **Shared Plan**: Dedicated S3 bucket per service instance on the shared cluster
- **On-Demand Dedicated Plans**: Provision isolated SeaweedFS clusters via BOSH (single-node or HA)
- **Gorouter Integration**: S3 and broker endpoints registered with Cloud Foundry router for TLS termination
- **Smoke Tests**: Automated errand validates end-to-end S3 connectivity (put, get, list, delete)
- **Health Checks**: Monit-integrated health checks for all SeaweedFS components

## Prerequisites

- Tanzu Operations Manager 3.0+
- Tanzu Application Service (TAS) deployed
- Network connectivity between the tile VMs and TAS components (NATS, gorouter)

## Building the Tile

The tile is built from the [bosh-seaweedfs](https://github.com/nkuhn-vmw/bosh-seaweedfs) BOSH release.

```bash
./scripts/build-tile.sh
```

Output: `product/seaweedfs-<version>.pivotal`

### Build Dependencies

The build script expects:
- `tile` CLI (tile-generator)
- The bosh-seaweedfs release repo at `../bosh-seaweedfs` (or set `BOSH_RELEASE_DIR`)
- BPM release at `resources/bpm-1.1.21.tgz`
- Routing release at `resources/routing-0.283.0.tgz`

## Installation

1. Upload `seaweedfs-<version>.pivotal` to Tanzu Operations Manager
2. Click "+" to add the tile to the installation
3. Configure tile settings (see below)
4. Apply Changes

## Tile Configuration

### Cluster Settings

| Setting | Description | Default |
|---------|-------------|---------|
| Master Instances | Number of master servers | 1 |
| Volume Instances | Number of volume servers | 3 |
| Replication Type | Data replication strategy (000, 001, 010, 100) | 000 |

### Networking

| Setting | Description | Default |
|---------|-------------|---------|
| S3 Route Name | Hostname for S3 endpoint (e.g., `seaweedfs-s3`) | seaweedfs-s3 |
| Broker Route Name | Hostname for broker endpoint | seaweedfs-broker |
| Master Console Route | Hostname for master UI | seaweedfs-master |
| Filer Console Route | Hostname for filer UI | seaweedfs-filer |

Routes are registered as `<route-name>.<system-domain>` via gorouter.

### Service Plans

The tile pre-configures a **shared** plan. On-demand dedicated plans can be added via the Service Plan Configuration form:

| Field | Description |
|-------|-------------|
| Plan Name | Display name in the CF marketplace |
| Deployment Type | `single_node` or `ha` |
| VM Type | BOSH VM type for the dedicated cluster |
| Disk Type | Persistent disk type |
| Storage Quota (GB) | Maximum storage per instance |

## Usage

After installation and applying changes:

```bash
# View the service in the marketplace
cf marketplace -e seaweedfs

# Create a shared bucket
cf create-service seaweedfs shared my-storage

# Bind to an app
cf bind-service my-app my-storage

# Restage to pick up credentials
cf restage my-app
```

### Binding Credentials

Each binding creates a dedicated IAM user with isolated access keys:

```json
{
  "credentials": {
    "endpoint": "seaweedfs-s3.sys.example.com",
    "endpoint_url": "https://seaweedfs-s3.sys.example.com",
    "bucket": "cf-abc123-def456",
    "access_key": "UNIQUE_PER_BINDING",
    "secret_key": "UNIQUE_PER_BINDING",
    "region": "us-east-1",
    "use_ssl": true,
    "uri": "s3://ACCESS:SECRET@seaweedfs-s3.sys.example.com/cf-abc123-def456"
  }
}
```

Credentials are scoped to the binding's bucket via IAM policy and automatically cleaned up on unbind.

## Architecture

```
                    +-----------+
                    | Gorouter  |  (TLS termination)
                    +-----+-----+
                          |
              +-----------+-----------+
              |                       |
        +-----+-----+         +------+------+
        | S3 Gateway |         |   Broker    |
        |  (port 8333)|        | (port 8080) |
        +-----+-----+         +------+------+
              |                       |
        +-----+-----+                |
        |   Filer    |         +------+------+
        | (port 8888)|         | BOSH Director|
        +-----+-----+         | (on-demand)  |
              |                +-------------+
        +-----+-----+
        |  Volume(s) |
        | (port 8080)|
        +-----+-----+
              |
        +-----+-----+
        |  Master    |
        | (port 9333)|
        +-----------+
```

### Credential Flow (Shared Plan)

1. `cf create-service` -- Broker creates an S3 bucket via the admin S3 client
2. `cf bind-service` -- Broker calls SeaweedFS IAM API (internal endpoint) to:
   - Create an IAM user (`CreateUser`)
   - Generate access keys (`CreateAccessKey`)
   - Attach a bucket-scoped policy (`PutUserPolicy`)
3. `cf unbind-service` -- Broker cleans up IAM user, keys, and policy
4. `cf delete-service` -- Broker deletes the S3 bucket and all objects

## Errands

| Errand | Description |
|--------|-------------|
| register-broker | Registers the service broker with Cloud Foundry |
| deregister-broker | Removes the service broker from Cloud Foundry |
| smoke-tests | Validates end-to-end: create service, bind, push app, test S3 ops, cleanup |

## Related Repositories

- [bosh-seaweedfs](https://github.com/nkuhn-vmw/bosh-seaweedfs) - BOSH release containing all jobs and the service broker source code

## License

Apache 2.0
