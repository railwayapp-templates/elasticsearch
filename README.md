## Elasticsearch

This template deploys Elasticsearch on Railway with support for both single-node and multi-node cluster configurations.

### Features

- **Single-node or Multi-node**: Deploy as a standalone instance or a 3-node cluster
- **Cloud-optimized**: Disables mmap (avoids `vm.max_map_count` kernel requirement)
- **Health checks**: Anonymous user with monitor permissions for health checks
- **Volume support**: Automatically handles volume permissions for non-root user

### Quick Start (Single Node)

Deploy this template as-is for a single-node Elasticsearch instance. No additional configuration required.

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CLUSTER_NAME` | `elasticsearch-cluster` | Name of the Elasticsearch cluster |
| `NODE_NAME` | `node-1` | Unique name for this node |
| `DISCOVERY_TYPE` | `single-node` | Set to `multi-node` for cluster mode |
| `DISCOVERY_SEED_HOSTS` | (empty) | Comma-separated list of other nodes (e.g., `es-node-2:9300,es-node-3:9300`) |
| `INITIAL_MASTER_NODES` | (empty) | Comma-separated list of initial master-eligible node names |
| `NODE_ROLES` | `master,data,ingest` | Roles for this node |
| `HTTP_PORT` | `9200` | HTTP API port |
| `TRANSPORT_PORT` | `9300` | Inter-node communication port |
| `TRANSPORT_SSL_ENABLED` | `false` | Enable TLS for inter-node communication |
| `ELASTIC_PASSWORD` | (auto-generated) | Password for the `elastic` superuser |

---

## 3-Node Cluster Setup on Railway

To deploy a 3-node Elasticsearch cluster on Railway:

### Step 1: Create Three Services

Deploy this repository as three separate Railway services in the same project:
- `es-node-1`
- `es-node-2`
- `es-node-3`

### Step 2: Enable Private Networking

Ensure all three services are in the same Railway project. Railway's private networking allows services to communicate using their service names as hostnames.

### Step 3: Configure Environment Variables

Set these environment variables for each service:

#### All Nodes (Shared Variables)

```
CLUSTER_NAME=my-elasticsearch-cluster
DISCOVERY_TYPE=multi-node
INITIAL_MASTER_NODES=node-1,node-2,node-3
ELASTIC_PASSWORD=your-secure-password
```

#### es-node-1

```
NODE_NAME=node-1
DISCOVERY_SEED_HOSTS=es-node-2:9300,es-node-3:9300
```

#### es-node-2

```
NODE_NAME=node-2
DISCOVERY_SEED_HOSTS=es-node-1:9300,es-node-3:9300
```

#### es-node-3

```
NODE_NAME=node-3
DISCOVERY_SEED_HOSTS=es-node-1:9300,es-node-2:9300
```

### Step 4: Add Volumes

Each service needs its own persistent volume mounted at `/esdata`.

### Step 5: Deploy

Deploy all three services. The nodes will automatically discover each other and form a cluster.

### Verifying the Cluster

Once all nodes are running, check cluster health:

```bash
curl -u elastic:your-secure-password http://your-es-node-1-url:9200/_cluster/health?pretty
```

Expected response:
```json
{
  "cluster_name": "my-elasticsearch-cluster",
  "status": "green",
  "number_of_nodes": 3,
  "number_of_data_nodes": 3
  ...
}
```

---

## Production Considerations

### Transport Layer Security (TLS)

For production clusters, enable TLS for inter-node communication:

1. Generate certificates using `elasticsearch-certutil`
2. Mount certificates in each container
3. Set `TRANSPORT_SSL_ENABLED=true`
4. Configure certificate paths in elasticsearch.yml

### Memory Settings

Set appropriate JVM heap size via the `ES_JAVA_OPTS` environment variable:

```
ES_JAVA_OPTS=-Xms2g -Xmx2g
```

Recommendation: Set heap to 50% of available memory, but no more than 31GB.

### Node Roles

For larger clusters, consider dedicated node roles:
- **Master-only nodes**: `NODE_ROLES=master`
- **Data-only nodes**: `NODE_ROLES=data`
- **Coordinating-only nodes**: `NODE_ROLES=` (empty)

---

## Technical Details

### Why mmap is Disabled

Elasticsearch normally requires `vm.max_map_count` to be set to `262144` (default is `65530`). This setting cannot be changed in most cloud-hosted environments like Railway. Disabling mmap (`node.store.allow_mmap: false`) avoids this requirement at the cost of some performance.

### Volume Permissions

The entrypoint script runs `chown` on the `/esdata` directory because:
- Docker volumes are mounted as root
- Elasticsearch runs as non-root user (UID 1000)
- The script ensures Elasticsearch can write to the data directory

### Anonymous Health Checks

An anonymous user with the `anonymous_role` is configured to allow unauthenticated health checks:
- Grants `monitor` cluster privilege
- Enables external health check endpoints without credentials

---

## Version Upgrades

Before upgrading, please carefully read the upgrade manual: https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster
