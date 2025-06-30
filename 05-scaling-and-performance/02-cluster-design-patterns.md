# Cluster Design Patterns

## Proven Architectures for Enterprise-Scale Elasticsearch

Designing production Elasticsearch clusters requires balancing performance, availability, cost, and operational complexity. This guide covers battle-tested architectural patterns that have proven successful in enterprise environments, from small-scale deployments to global, multi-datacenter installations handling petabytes of data.

---

## 🏗️ Architecture Fundamentals

### The Building Blocks of Cluster Design

**Core design principles:**

- **Fault tolerance:** No single point of failure
- **Scalability:** Linear performance scaling with capacity
- **Operational simplicity:** Manageable complexity
- **Cost efficiency:** Optimal resource utilization
- **Performance predictability:** Consistent response times

**Essential components:**

```
Elasticsearch Cluster Architecture
├── Load Balancing Layer
│   ├── Application Load Balancers
│   └── Connection pooling
├── Elasticsearch Nodes
│   ├── Master-eligible nodes
│   ├── Coordinating nodes
│   ├── Data nodes (hot/warm/cold)
│   ├── Ingest nodes
│   └── Machine learning nodes
├── Storage Layer
│   ├── Local NVMe (hot data)
│   ├── Network storage (warm data)
│   └── Object storage (cold data)
└── Monitoring & Management
    ├── Cluster monitoring
    ├── Log aggregation
    └── Alerting systems

```

### Sizing Framework

**Capacity planning methodology:**

```python
def calculate_cluster_capacity(requirements):
    """Calculate cluster capacity based on requirements"""

    # Data requirements
    daily_data_gb = requirements['daily_ingest_gb']
    retention_days = requirements['retention_days']
    replication_factor = requirements['replication_factor']
    growth_factor = requirements.get('growth_factor', 1.5)  # 50% buffer

    # Calculate storage needs
    raw_storage_gb = daily_data_gb * retention_days
    total_storage_gb = raw_storage_gb * replication_factor * growth_factor

    # Performance requirements
    peak_index_rate = requirements['peak_docs_per_second']
    peak_query_rate = requirements['peak_queries_per_second']
    latency_target_ms = requirements['query_latency_p95_ms']

    # Node calculations
    storage_per_node_gb = 2000  # Conservative for enterprise SSDs
    nodes_for_storage = math.ceil(total_storage_gb / storage_per_node_gb)

    # Performance-based node calculation
    docs_per_node_per_second = 1000  # Conservative estimate
    nodes_for_indexing = math.ceil(peak_index_rate / docs_per_node_per_second)

    queries_per_node_per_second = 100  # Depends on query complexity
    nodes_for_queries = math.ceil(peak_query_rate / queries_per_node_per_second)

    # Take maximum of storage and performance requirements
    data_nodes_needed = max(nodes_for_storage, nodes_for_indexing, nodes_for_queries)

    # Add other node types
    master_nodes = 3  # Always 3 for quorum
    coordinating_nodes = max(2, data_nodes_needed // 4)  # 25% of data nodes
    ingest_nodes = max(1, peak_index_rate // 10000)  # 1 per 10k docs/sec

    return {
        'data_nodes': data_nodes_needed,
        'master_nodes': master_nodes,
        'coordinating_nodes': coordinating_nodes,
        'ingest_nodes': ingest_nodes,
        'total_nodes': data_nodes_needed + master_nodes + coordinating_nodes + ingest_nodes,
        'storage_per_node_gb': storage_per_node_gb,
        'total_storage_gb': total_storage_gb,
        'estimated_cost_per_month': calculate_estimated_cost(data_nodes_needed + master_nodes + coordinating_nodes + ingest_nodes)
    }

def calculate_estimated_cost(total_nodes, cost_per_node_per_month=1000):
    """Rough cost estimation for planning"""
    return total_nodes * cost_per_node_per_month

```

---

## 🏢 Enterprise Deployment Patterns

### Pattern 1: Single Datacenter, High Availability

**Use case:** Most enterprise deployments with high availability requirements within a single region.

**Architecture:**

```
Single Datacenter Cluster (3 Availability Zones)
├── Zone A
│   ├── Master Node 1 (m5.large)
│   ├── Data Nodes 1-3 (r5.2xlarge)
│   └── Coordinating Node 1 (c5.xlarge)
├── Zone B
│   ├── Master Node 2 (m5.large)
│   ├── Data Nodes 4-6 (r5.2xlarge)
│   └── Coordinating Node 2 (c5.xlarge)
└── Zone C
    ├── Master Node 3 (m5.large)
    ├── Data Nodes 7-9 (r5.2xlarge)
    └── Load Balancer (ALB)

```

**Configuration example:**

```yaml
# Master-eligible nodes
cluster.name: production-cluster
node.name: master-01
node.roles: [master]
discovery.seed_hosts: ["master-01", "master-02", "master-03"]
cluster.initial_master_nodes: ["master-01", "master-02", "master-03"]

# Allocation awareness
cluster.routing.allocation.awareness.attributes: zone
node.attr.zone: zone-a

# Network settings
network.host: _site_
http.port: 9200
transport.port: 9300

# Memory and performance
bootstrap.memory_lock: true
indices.memory.index_buffer_size: 20%

```

**Advantages:**

- Simple network topology
- Low latency between nodes
- Cost-effective for most use cases
- Easy to manage and monitor

**Considerations:**

- Single region risk (natural disasters, regional outages)
- Network latency between zones (1-2ms typically acceptable)
- Bandwidth limits between zones

### Pattern 2: Multi-Datacenter with Cross-Cluster Replication

**Use case:** Global applications requiring data locality and disaster recovery.

**Architecture:**

```
Multi-Datacenter Deployment
├── Primary Datacenter (US-East)
│   ├── Full Elasticsearch Cluster
│   ├── Active writes and reads
│   └── Cross-cluster replication source
├── Secondary Datacenter (EU-West)
│   ├── Full Elasticsearch Cluster
│   ├── Read-only operations
│   └── CCR follower indices
└── Disaster Recovery (US-West)
    ├── Minimal cluster (cold standby)
    └── Snapshot repository

```

**Cross-cluster replication setup:**

```json
{
  "persistent": {
    "cluster": {
      "remote": {
        "primary_cluster": {
          "seeds": ["primary-coord-1:9300", "primary-coord-2:9300"],
          "skip_unavailable": false
        }
      }
    }
  }
}

// Create follower index
PUT /logs-follower/_ccr/follow
{
  "remote_cluster": "primary_cluster",
  "leader_index": "logs-leader",
  "max_read_request_operation_count": 5120,
  "max_outstanding_read_requests": 12,
  "max_read_request_size": "32mb",
  "max_write_request_operation_count": 5120,
  "max_write_request_size": "9mb",
  "max_outstanding_write_requests": 9,
  "max_write_buffer_count": 2147483647,
  "max_write_buffer_size": "512mb",
  "max_retry_delay": "500ms",
  "read_poll_timeout": "1m"
}

```

### Pattern 3: Dedicated Cluster by Use Case

**Use case:** Large organizations with distinct workload requirements.

**Architecture:**

```
Use Case Segregation
├── Search Cluster
│   ├── Optimized for query performance
│   ├── High replica count
│   └── SSD storage, high memory
├── Analytics Cluster
│   ├── Optimized for aggregations
│   ├── Larger shards, more CPU
│   └── Mixed storage tiers
├── Logging Cluster
│   ├── Optimized for ingestion
│   ├── Time-based indices
│   └── Hot-warm-cold architecture
└── Security Cluster (SIEM)
    ├── Compliance requirements
    ├── Enhanced security features
    └── Long-term retention

```

**Cluster specialization example:**

```python
cluster_configs = {
    'search_cluster': {
        'node_types': {
            'coordinating': {'count': 3, 'size': 'c5.2xlarge'},
            'data': {'count': 12, 'size': 'r5.4xlarge'}
        },
        'settings': {
            'number_of_replicas': 2,
            'refresh_interval': '1s',
            'requests_cache_enable': True
        },
        'hardware': 'NVMe SSD, high memory'
    },
    'analytics_cluster': {
        'node_types': {
            'coordinating': {'count': 2, 'size': 'c5.4xlarge'},
            'data': {'count': 8, 'size': 'r5.8xlarge'}
        },
        'settings': {
            'number_of_replicas': 1,
            'refresh_interval': '30s',
            'aggregation_optimization': True
        },
        'hardware': 'High CPU, large memory'
    },
    'logging_cluster': {
        'node_types': {
            'ingest': {'count': 4, 'size': 'c5.2xlarge'},
            'data_hot': {'count': 6, 'size': 'i3.2xlarge'},
            'data_warm': {'count': 4, 'size': 'r5.xlarge'},
            'data_cold': {'count': 2, 'size': 'm5.large'}
        },
        'settings': {
            'number_of_replicas': 0,
            'refresh_interval': '30s',
            'lifecycle_policy': 'logs_policy'
        },
        'hardware': 'Mixed: NVMe → SATA → HDD'
    }
}

```

---

## 🛡️ High Availability and Resilience Patterns

### Master Node Quorum Design

**Best practices for master node deployment:**

```python
def design_master_quorum(total_nodes, availability_zones):
    """Design master node quorum for high availability"""

    # Always use odd number of master-eligible nodes
    if total_nodes < 3:
        master_nodes = 3  # Minimum for production
    elif total_nodes < 10:
        master_nodes = 3  # Small cluster
    elif total_nodes < 20:
        master_nodes = 5  # Medium cluster
    else:
        master_nodes = 7  # Large cluster (rarely need more)

    # Distribute across availability zones
    if availability_zones >= 3:
        # Ideal: spread evenly across zones
        nodes_per_zone = master_nodes // availability_zones
        remainder = master_nodes % availability_zones

        distribution = {}
        for i in range(availability_zones):
            zone_name = f"zone-{chr(ord('a') + i)}"
            distribution[zone_name] = nodes_per_zone + (1 if i < remainder else 0)

    else:
        # Suboptimal: fewer than 3 zones
        distribution = {"warning": "Fewer than 3 zones reduces availability"}

    quorum_size = (master_nodes // 2) + 1

    return {
        'master_nodes': master_nodes,
        'quorum_size': quorum_size,
        'zone_distribution': distribution,
        'minimum_nodes_for_operation': quorum_size
    }

```

### Data Replication Strategies

**Replica placement for maximum availability:**

```yaml
# Force replica allocation across zones
cluster.routing.allocation.awareness.attributes: zone
cluster.routing.allocation.awareness.force.zone.values: zone-a,zone-b,zone-c

# Shard allocation constraints
cluster.routing.allocation.same_shard.host: false
cluster.routing.allocation.total_shards_per_node: 3

```

**Advanced allocation filtering:**

```json
{
  "settings": {
    "index.routing.allocation.require.zone": "zone-a,zone-b,zone-c",
    "index.routing.allocation.exclude.node_type": "coordinating",
    "index.routing.allocation.include.storage_type": "ssd"
  }
}

```

### Circuit Breakers and Resource Protection

**Memory protection configuration:**

```yaml
# Circuit breaker settings
indices.breaker.total.limit: 70%
indices.breaker.fielddata.limit: 40%
indices.breaker.request.limit: 60%

# JVM settings
-Xms32g
-Xmx32g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200

# Thread pool settings
thread_pool.write.size: 8
thread_pool.write.queue_size: 200
thread_pool.search.size: 13
thread_pool.search.queue_size: 1000

```

---

## 🌐 Network Design and Load Balancing

### Load Balancer Configuration

**Application Load Balancer setup:**

```python
def design_load_balancer_config(coordinating_nodes, data_nodes):
    """Design load balancer configuration for Elasticsearch"""

    config = {
        'target_groups': {
            'search_queries': {
                'targets': coordinating_nodes + data_nodes,
                'port': 9200,
                'protocol': 'HTTP',
                'health_check': {
                    'path': '/_cluster/health',
                    'interval_seconds': 30,
                    'timeout_seconds': 5,
                    'healthy_threshold': 2,
                    'unhealthy_threshold': 3
                }
            },
            'bulk_indexing': {
                'targets': data_nodes,  # Direct to data nodes
                'port': 9200,
                'protocol': 'HTTP',
                'health_check': {
                    'path': '/_cluster/health?level=shards',
                    'interval_seconds': 10,
                    'timeout_seconds': 3
                }
            }
        },
        'listener_rules': {
            'search_traffic': {
                'condition': 'path-pattern: /*/_search, /*/_msearch',
                'target_group': 'search_queries',
                'algorithm': 'round_robin'
            },
            'bulk_traffic': {
                'condition': 'path-pattern: /*/_bulk, /*/_doc',
                'target_group': 'bulk_indexing',
                'algorithm': 'least_connections'
            }
        },
        'connection_settings': {
            'idle_timeout_seconds': 60,
            'connection_draining_timeout_seconds': 300,
            'sticky_sessions': False
        }
    }

    return config

```

### Client Connection Patterns

**Connection pooling and retry logic:**

```python
from elasticsearch import Elasticsearch
from elasticsearch.connection import create_ssl_context
import ssl

class ElasticsearchClientManager:
    def __init__(self, hosts, max_retries=3, timeout=30):
        self.hosts = hosts
        self.max_retries = max_retries
        self.timeout = timeout
        self.client = self._create_client()

    def _create_client(self):
        """Create optimized Elasticsearch client"""

        # SSL context for security
        ssl_context = create_ssl_context()
        ssl_context.check_hostname = False
        ssl_context.verify_mode = ssl.CERT_NONE

        return Elasticsearch(
            hosts=self.hosts,

            # Connection pooling
            maxsize=25,          # Connection pool size per host
            max_retries=self.max_retries,
            retry_on_status={429, 502, 503, 504},
            retry_on_timeout=True,

            # Timeouts
            timeout=self.timeout,

            # Performance settings
            sniff_on_start=True,     # Discover cluster topology
            sniff_on_connection_fail=True,
            sniffer_timeout=60,

            # Health checking
            verify_certs=False,
            ssl_context=ssl_context,

            # Headers for debugging
            headers={'User-Agent': 'MyApp/1.0'}
        )

    def execute_with_retry(self, operation, *args, **kwargs):
        """Execute operation with automatic retry logic"""

        last_exception = None
        for attempt in range(self.max_retries + 1):
            try:
                return operation(*args, **kwargs)
            except Exception as e:
                last_exception = e
                if attempt < self.max_retries:
                    wait_time = 2 ** attempt  # Exponential backoff
                    time.sleep(wait_time)
                else:
                    raise last_exception

    def bulk_index_with_retries(self, documents, index_name):
        """Bulk index with retry and error handling"""

        def bulk_operation():
            from elasticsearch.helpers import bulk

            actions = [
                {
                    '_index': index_name,
                    '_source': doc
                }
                for doc in documents
            ]

            return bulk(self.client, actions, refresh='wait_for')

        return self.execute_with_retry(bulk_operation)

```

---

## 💾 Storage Architecture Patterns

### Hot-Warm-Cold Storage Tiers

**Storage tier configuration:**

```python
def design_storage_tiers(data_retention_days, daily_data_gb):
    """Design storage tier architecture"""

    tiers = {
        'hot': {
            'duration_days': 7,
            'storage_type': 'Local NVMe SSD',
            'node_type': 'data_hot',
            'characteristics': {
                'iops': '100,000+',
                'latency': '<1ms',
                'cost_per_gb': '$0.25/month'
            },
            'sizing': daily_data_gb * 7 * 1.5  # 50% buffer
        },
        'warm': {
            'duration_days': 30,
            'storage_type': 'Attached SSD',
            'node_type': 'data_warm',
            'characteristics': {
                'iops': '20,000',
                'latency': '1-3ms',
                'cost_per_gb': '$0.10/month'
            },
            'sizing': daily_data_gb * 30 * 1.2  # 20% buffer
        },
        'cold': {
            'duration_days': data_retention_days - 37,
            'storage_type': 'Network attached storage / Searchable snapshots',
            'node_type': 'data_cold',
            'characteristics': {
                'iops': '5,000',
                'latency': '5-10ms',
                'cost_per_gb': '$0.05/month'
            },
            'sizing': daily_data_gb * (data_retention_days - 37)
        }
    }

    # Calculate cost savings
    traditional_cost = (daily_data_gb * data_retention_days) * 0.25
    tiered_cost = (
        tiers['hot']['sizing'] * 0.25 +
        tiers['warm']['sizing'] * 0.10 +
        tiers['cold']['sizing'] * 0.05
    )

    return {
        'tiers': tiers,
        'cost_savings': {
            'traditional_monthly_cost': traditional_cost,
            'tiered_monthly_cost': tiered_cost,
            'savings_percentage': ((traditional_cost - tiered_cost) / traditional_cost) * 100
        }
    }

```

### Backup and Snapshot Strategy

**Comprehensive backup architecture:**

```yaml
# Snapshot repository configuration
path.repo: ["/mnt/backup", "s3://es-snapshots-bucket"]

# Repository settings
PUT /_snapshot/daily_snapshots
{
  "type": "s3",
  "settings": {
    "bucket": "es-snapshots-bucket",
    "region": "us-east-1",
    "base_path": "elasticsearch/daily",
    "compress": true,
    "chunk_size": "1gb",
    "server_side_encryption": true
  }
}

# Snapshot lifecycle policy
PUT /_slm/policy/daily_snapshots_policy
{
  "schedule": "0 30 1 * * ?",
  "name": "<daily-snapshot-{now/d}>",
  "repository": "daily_snapshots",
  "config": {
    "indices": "*",
    "ignore_unavailable": true,
    "include_global_state": false
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 100
  }
}

```

---

## 🔐 Security Architecture

### Network Security Patterns

**Security perimeter design:**

```python
def design_security_architecture():
    """Design comprehensive security architecture"""

    security_layers = {
        'network_security': {
            'vpc_isolation': 'Dedicated VPC for Elasticsearch',
            'subnets': 'Private subnets for all ES nodes',
            'security_groups': [
                'ES cluster communication (9200, 9300)',
                'Load balancer access (443 only)',
                'Monitoring access (from monitoring VPC)'
            ],
            'nat_gateway': 'For outbound internet access',
            'vpc_endpoints': 'S3, CloudWatch, Systems Manager'
        },
        'authentication': {
            'method': 'SAML/OIDC integration',
            'backup_method': 'Native realm with strong passwords',
            'api_keys': 'For service-to-service communication',
            'certificate_auth': 'For node-to-node communication'
        },
        'authorization': {
            'rbac': 'Role-based access control',
            'field_level_security': 'For sensitive data',
            'document_level_security': 'Multi-tenant isolation',
            'anonymization': 'For compliance requirements'
        },
        'encryption': {
            'in_transit': 'TLS 1.3 for all communications',
            'at_rest': 'Encrypted storage volumes',
            'snapshot_encryption': 'Encrypted S3 buckets',
            'certificate_management': 'Automated rotation'
        }
    }

    return security_layers

```

### Compliance and Auditing

**Audit configuration for compliance:**

```yaml
# Enable audit logging
xpack.security.audit.enabled: true
xpack.security.audit.outputs: ["index", "logfile"]

# Index audit events
xpack.security.audit.index.settings:
  number_of_shards: 1
  number_of_replicas: 1
  index.lifecycle.name: "audit_policy"

# Audit event filtering
xpack.security.audit.logfile.events.exclude:
  - "access_granted"
  - "anonymous_access_denied"

# What to audit
xpack.security.audit.logfile.events.include:
  - "access_denied"
  - "authentication_failed"
  - "connection_denied"
  - "tampered_request"
  - "run_as_denied"
  - "run_as_granted"

```

---

## 📊 Monitoring and Observability

### Comprehensive Monitoring Stack

**Monitoring architecture:**

```python
def design_monitoring_stack():
    """Design comprehensive monitoring for Elasticsearch"""

    monitoring_components = {
        'cluster_monitoring': {
            'tool': 'Elastic Stack Monitoring / Prometheus',
            'metrics': [
                'Cluster health and status',
                'Node resource utilization',
                'Index and shard metrics',
                'Query and indexing performance'
            ],
            'retention': '90 days'
        },
        'application_monitoring': {
            'tool': 'APM / Custom dashboards',
            'metrics': [
                'Query latency distribution',
                'Error rates by endpoint',
                'Business metrics (search conversion)',
                'User experience metrics'
            ],
            'retention': '30 days'
        },
        'infrastructure_monitoring': {
            'tool': 'CloudWatch / Datadog',
            'metrics': [
                'CPU, memory, disk, network',
                'JVM metrics',
                'OS-level performance',
                'Storage I/O patterns'
            ],
            'retention': '365 days'
        },
        'log_monitoring': {
            'tool': 'ELK Stack',
            'logs': [
                'Elasticsearch logs',
                'Application logs',
                'Load balancer logs',
                'Security audit logs'
            ],
            'retention': 'Based on compliance requirements'
        }
    }

    alerting_rules = {
        'critical': [
            'Cluster status RED',
            'Master node unavailable',
            'Disk usage >90%',
            'JVM heap usage >85%'
        ],
        'warning': [
            'Cluster status YELLOW',
            'Query latency P95 >500ms',
            'Indexing errors >1%',
            'Unassigned shards detected'
        ],
        'informational': [
            'Large shard detected',
            'Slow query detected',
            'Index rollover occurred'
        ]
    }

    return {
        'monitoring_components': monitoring_components,
        'alerting_rules': alerting_rules,
        'dashboard_categories': [
            'Cluster overview',
            'Performance metrics',
            'Capacity planning',
            'Security events'
        ]
    }

```

### Key Performance Indicators

**SLA and SLO definitions:**

```python
def define_service_levels():
    """Define service level objectives for Elasticsearch"""

    slos = {
        'availability': {
            'target': '99.9%',
            'measurement': 'Successful API responses / Total API requests',
            'window': 'Monthly'
        },
        'query_latency': {
            'target': 'P95 < 200ms, P99 < 500ms',
            'measurement': 'Search API response time',
            'window': 'Daily'
        },
        'indexing_latency': {
            'target': 'P95 < 1s',
            'measurement': 'Bulk API response time',
            'window': 'Daily'
        },
        'data_durability': {
            'target': '99.999%',
            'measurement': 'Successful snapshot creation and restoration',
            'window': 'Monthly'
        }
    }

    error_budgets = {
        'availability': {
            'monthly_budget': '0.1%',  # 43.8 minutes per month
            'daily_budget': '0.003%'   # 2.6 minutes per day
        },
        'performance': {
            'slow_query_budget': '1%',  # 1% of queries can be slow
            'timeout_budget': '0.1%'    # 0.1% of queries can timeout
        }
    }

    return {
        'service_level_objectives': slos,
        'error_budgets': error_budgets
    }

```

---

## 🚀 Deployment and Operations Patterns

### Infrastructure as Code

**Terraform configuration example:**

```hcl
# Elasticsearch cluster infrastructure
module "elasticsearch_cluster" {
  source = "./modules/elasticsearch"

  cluster_name     = "production-es"
  environment      = "prod"
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

  # Node configuration
  master_nodes = {
    count         = 3
    instance_type = "m5.large"
    volume_size   = 100
  }

  data_hot_nodes = {
    count         = 6
    instance_type = "r5.2xlarge"
    volume_size   = 1000
    volume_type   = "gp3"
    iops          = 10000
  }

  data_warm_nodes = {
    count         = 4
    instance_type = "r5.xlarge"
    volume_size   = 2000
    volume_type   = "gp3"
  }

  coordinating_nodes = {
    count         = 2
    instance_type = "c5.xlarge"
    volume_size   = 100
  }

  # Network configuration
  vpc_id            = data.aws_vpc.main.id
  private_subnet_ids = data.aws_subnets.private.ids

  # Security
  security_group_ids = [aws_security_group.elasticsearch.id]
  certificate_arn    = data.aws_acm_certificate.elasticsearch.arn

  # Monitoring
  enable_monitoring = true
  log_retention_days = 90

  tags = {
    Environment = "production"
    Team        = "platform"
    CostCenter  = "engineering"
  }
}

```

### Automated Operations

**Operational automation framework:**

```python
class ElasticsearchOperations:
    def __init__(self, cluster_client, monitoring_client):
        self.es = cluster_client
        self.monitoring = monitoring_client

    def health_check_automation(self):
        """Automated health checking and remediation"""

        health = self.es.cluster.health()

        if health['status'] == 'red':
            self.handle_red_cluster()
        elif health['status'] == 'yellow':
            self.handle_yellow_cluster()

        # Check individual node health
        nodes = self.es.cat.nodes(format='json')
        for node in nodes:
            if float(node['heap.percent']) > 85:
                self.alert_high_heap_usage(node)

            if float(node['disk.used_percent']) > 90:
                self.handle_disk_pressure(node)

    def handle_red_cluster(self):
        """Handle cluster in RED status"""

        # Get unassigned shards
        unassigned = self.es.cat.shards(h='index,shard,prirep,state,unassigned.reason',
                                       s='index', format='json')

        for shard in unassigned:
            if shard['state'] == 'UNASSIGNED':
                if shard['unassigned.reason'] == 'NODE_LEFT':
                    # Try to reallocate manually
                    self.force_shard_allocation(shard)
                elif shard['unassigned.reason'] == 'ALLOCATION_FAILED':
                    # Check allocation explain
                    self.investigate_allocation_failure(shard)

    def automated_index_management(self):
        """Automated index lifecycle management"""

        # Monitor index sizes and trigger rollovers
        indices = self.es.cat.indices(format='json')

        for index in indices:
            size_gb = float(index['store.size'].replace('gb', ''))

            if size_gb > 50:  # Rollover threshold
                if self.should_rollover(index['index']):
                    self.trigger_rollover(index['index'])

            # Check for old indices to delete
            if self.should_delete_index(index['index']):
                self.safe_delete_index(index['index'])

    def capacity_planning_automation(self):
        """Automated capacity planning alerts"""

        disk_usage = self.get_cluster_disk_usage()
        growth_rate = self.calculate_growth_rate()

        days_until_full = self.estimate_capacity_exhaustion(disk_usage, growth_rate)

        if days_until_full < 30:
            self.alert_capacity_planning({
                'current_usage': disk_usage,
                'growth_rate': growth_rate,
                'days_until_full': days_until_full
            })

```

---

## 💡 Key Takeaways

✅ **Design for failure from day one—plan for node failures, zone outages, and network partitions**

✅ **Use appropriate cluster topologies based on your availability and performance requirements**

✅ **Implement proper load balancing and client connection management**

✅ **Design storage tiers to optimize both performance and cost**

✅ **Secure your cluster with defense-in-depth principles**

✅ **Monitor comprehensively and automate operational tasks**

✅ **Use infrastructure as code for consistent, repeatable deployments**

---