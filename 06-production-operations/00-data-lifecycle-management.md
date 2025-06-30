# Data Lifecycle Management

## Automated Data Aging and Cost Optimization

Data Lifecycle Management (ILM) transforms how you handle growing data volumes by automatically transitioning data through performance tiers as it ages. Instead of manually managing storage costs and performance trade-offs, ILM policies automate the entire process—from high-performance hot storage for recent data to cost-effective cold storage for archives, with automatic deletion when data reaches its retention limit.

---

## 🔄 The Data Lifecycle Paradigm

### Understanding Data Value Over Time

**Data access patterns reality:**

```
Data Access Frequency Over Time

High  |████████░░░░░░░░░░░░░░░░░░░░
      |████████░░░░░░░░░░░░░░░░░░░░
      |████████░░░░░░░░░░░░░░░░░░░░
      |████████░░░░░░░░░░░░░░░░░░░░
      |████████░░░░░░░░░░░░░░░░░░░░
Low   |████████░░░░░░░░░░░░░░░░░░░░
      +--+--+--+--+--+--+--+--+--+
      0  7  30 90 180 365 ...    Days

```

**Key insights:**

- **80% of queries** target data from the last 7 days
- **15% of queries** access data from 7-30 days old
- **5% of queries** need older historical data
- **Storage costs** can be reduced by 70-80% through tiering

### Traditional vs Lifecycle Management

**Traditional approach problems:**

```python
# Manual index management nightmare
indices_to_delete = [
    'logs-2023-01-01',  # Check if safe to delete?
    'logs-2023-01-02',  # Still needed for compliance?
    'logs-2023-01-03',  # Contains important investigation data?
    # ... 365 manual decisions per year
]

# Storage cost explosion
monthly_storage_cost = daily_data_gb * 365 * storage_cost_per_gb
# 10GB/day * 365 days * $0.25/GB = $912.50/month for 1 year retention

```

**ILM automated approach:**

```python
# Automated, policy-driven management
ilm_policy = {
    'hot_phase': '7 days on expensive NVMe SSD',
    'warm_phase': '23 days on cheaper SATA SSD',
    'cold_phase': '335 days on object storage',
    'delete_phase': 'automatic deletion after 365 days'
}

# Optimized cost structure
monthly_storage_cost = (
    (daily_data_gb * 7 * premium_storage_cost) +      # Hot: $1.75
    (daily_data_gb * 23 * medium_storage_cost) +      # Warm: $2.30
    (daily_data_gb * 335 * cheap_storage_cost)        # Cold: $16.75
)
# Total: $20.80/month vs $912.50 (98% cost reduction!)

```

---

## 🏗️ Index Lifecycle Management Fundamentals

### ILM Phases Explained

**The four phases of data life:**

| Phase | Purpose | Duration | Storage Type | Replicas | Actions |
| --- | --- | --- | --- | --- | --- |
| **Hot** | Active read/write | 0-7 days | NVMe SSD | 1-2 | None (active indexing) |
| **Warm** | Read-only queries | 7-30 days | SATA SSD | 0-1 | Read-only, force merge |
| **Cold** | Occasional access | 30+ days | Object storage | 0 | Searchable snapshots |
| **Delete** | End of lifecycle | After retention | N/A | N/A | Delete index |

### Basic ILM Policy Structure

**Simple time-based policy:**

```json
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "7d",
            "max_docs": 10000000
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "set_priority": {
            "priority": 50
          },
          "allocate": {
            "number_of_replicas": 0
          },
          "readonly": {},
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "cold_repository"
          }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}

```

### Rollover Mechanics

**Understanding index rollover:**

```python
def explain_rollover_logic():
    """Explain how ILM rollover works"""

    rollover_example = {
        'index_pattern': 'logs-000001',  # Initial index
        'alias': 'logs',                 # Write alias points here

        'rollover_triggers': {
            'max_size': '50GB',    # Size threshold
            'max_age': '7d',       # Time threshold
            'max_docs': '10M'      # Document count threshold
        },

        'rollover_process': [
            '1. Check if any trigger condition is met',
            '2. Create new index: logs-000002',
            '3. Update write alias to point to new index',
            '4. Old index (logs-000001) enters warm phase',
            '5. Continue indexing to new index'
        ],

        'benefits': [
            'Predictable index sizes',
            'Parallel processing across indices',
            'Granular retention control',
            'Optimized storage allocation'
        ]
    }

    return rollover_example

```

---

## 🔥 Hot Phase Optimization

### Hot Phase Configuration

**Optimizing for active data:**

```json
{
  "hot": {
    "min_age": "0ms",
    "actions": {
      "rollover": {
        "max_size": "30gb",        // Optimal shard size
        "max_age": "1d",           // Daily rollover
        "max_docs": 50000000       // 50M docs max
      },
      "set_priority": {
        "priority": 100            // Highest recovery priority
      }
    }
  }
}

```

**Hot phase best practices:**

```python
def optimize_hot_phase_settings():
    """Optimize index settings for hot phase"""

    hot_settings = {
        'index_settings': {
            'number_of_shards': 1,         # Start small, scale if needed
            'number_of_replicas': 1,       # High availability
            'refresh_interval': '1s',      # Real-time search
            'translog.durability': 'request',  # Durability for critical data
            'codec': 'default'             # Speed over compression
        },

        'node_allocation': {
            'require_tier': 'data_hot',
            'hardware_specs': {
                'storage': 'NVMe SSD',
                'memory': '64GB+',
                'cpu': 'High performance'
            }
        },

        'monitoring_focus': [
            'Indexing rate and latency',
            'Query response times',
            'Resource utilization',
            'Rollover timing'
        ]
    }

    return hot_settings

```

### Dynamic Hot Phase Scaling

**Auto-scaling based on ingestion rate:**

```python
class HotPhaseManager:
    def __init__(self, es_client):
        self.es = es_client

    def analyze_ingestion_patterns(self, index_pattern):
        """Analyze ingestion to optimize rollover"""

        stats = self.es.indices.stats(index=index_pattern)

        analysis = {}
        for index_name, index_stats in stats['indices'].items():
            docs_total = index_stats['total']['docs']['count']
            size_bytes = index_stats['total']['store']['size_in_bytes']

            # Calculate ingestion rate
            index_age_hours = self.get_index_age_hours(index_name)
            docs_per_hour = docs_total / max(index_age_hours, 1)

            analysis[index_name] = {
                'docs_per_hour': docs_per_hour,
                'size_gb': size_bytes / (1024**3),
                'projected_daily_size_gb': (size_bytes / max(index_age_hours, 1)) * 24 / (1024**3),
                'recommendation': self.recommend_rollover_settings(docs_per_hour, size_bytes)
            }

        return analysis

    def recommend_rollover_settings(self, docs_per_hour, current_size_bytes):
        """Recommend optimal rollover settings"""

        if docs_per_hour > 100000:  # High volume
            return {
                'max_size': '20gb',
                'max_age': '12h',
                'reason': 'High ingestion rate requires frequent rollover'
            }
        elif docs_per_hour > 10000:  # Medium volume
            return {
                'max_size': '30gb',
                'max_age': '1d',
                'reason': 'Standard rollover for medium volume'
            }
        else:  # Low volume
            return {
                'max_size': '50gb',
                'max_age': '7d',
                'reason': 'Less frequent rollover for low volume'
            }

```

---

## 🌡️ Warm Phase Optimization

### Warm Phase Transitions

**Optimizing for read-only access:**

```json
{
  "warm": {
    "min_age": "7d",
    "actions": {
      "readonly": {},              // Prevent further writes
      "allocate": {
        "number_of_replicas": 0,   // Reduce replica overhead
        "include": {
          "data_tier": "data_warm"
        }
      },
      "forcemerge": {
        "max_num_segments": 1      // Optimize for read performance
      },
      "set_priority": {
        "priority": 50             // Lower recovery priority
      }
    }
  }
}

```

### Force Merge Strategy

**Understanding segment optimization:**

```python
def explain_force_merge_benefits():
    """Explain why force merge matters in warm phase"""

    before_merge = {
        'segments_per_shard': 47,
        'memory_per_segment_mb': 2.5,
        'total_memory_mb': 47 * 2.5,  # 117.5 MB
        'query_overhead': 'Check 47 segments per query',
        'disk_fragmentation': 'High - many small files'
    }

    after_merge = {
        'segments_per_shard': 1,
        'memory_per_segment_mb': 12.3,
        'total_memory_mb': 12.3,      # 89% reduction
        'query_overhead': 'Check 1 segment per query',
        'disk_fragmentation': 'Minimal - single optimized file'
    }

    benefits = {
        'memory_savings': '90%+ reduction in segment metadata',
        'query_performance': '20-40% faster searches',
        'storage_efficiency': '10-20% disk space savings',
        'os_cache_efficiency': 'Better file system cache utilization'
    }

    return {
        'before': before_merge,
        'after': after_merge,
        'benefits': benefits
    }

```

**Force merge monitoring:**

```python
def monitor_force_merge_progress(es_client, index_pattern):
    """Monitor force merge operations"""

    # Check active force merge operations
    force_merge_tasks = es_client.tasks.list(
        actions='*forcemerge*',
        detailed=True
    )

    if force_merge_tasks['nodes']:
        for node_id, node_tasks in force_merge_tasks['nodes'].items():
            for task_id, task in node_tasks['tasks'].items():
                print(f"Force merge progress: {task['description']}")
                print(f"Running time: {task['running_time_in_nanos'] / 1000000}ms")

    # Check segment counts after merge
    segments = es_client.cat.segments(
        index=index_pattern,
        v=True,
        h='index,shard,prirep,segment,size,memory'
    )

    return segments

```

---

## ❄️ Cold Phase and Searchable Snapshots

### Cold Storage Architecture

**Searchable snapshots implementation:**

```json
{
  "cold": {
    "min_age": "30d",
    "actions": {
      "searchable_snapshot": {
        "snapshot_repository": "s3_cold_repository",
        "force_merge_index": true
      }
    }
  }
}

```

**Repository configuration for cold storage:**

```json
{
  "type": "s3",
  "settings": {
    "bucket": "elasticsearch-cold-storage",
    "region": "us-east-1",
    "base_path": "cold_indices",
    "compress": true,
    "chunk_size": "1gb",
    "server_side_encryption": true,
    "storage_class": "STANDARD_IA",
    "max_restore_bytes_per_sec": "100mb",
    "max_snapshot_bytes_per_sec": "100mb"
  }
}

```

### Searchable Snapshots Benefits

**Cost and performance analysis:**

```python
def analyze_searchable_snapshots_benefits(index_size_gb, query_frequency_per_day):
    """Analyze benefits of searchable snapshots"""

    traditional_storage = {
        'storage_type': 'Local SSD',
        'cost_per_gb_per_month': 0.25,
        'total_monthly_cost': index_size_gb * 0.25,
        'query_latency_ms': 50,
        'availability': '99.9%'
    }

    searchable_snapshot = {
        'storage_type': 'S3 Standard-IA',
        'cost_per_gb_per_month': 0.05,
        'retrieval_cost_per_request': 0.0001,
        'total_monthly_cost': (
            index_size_gb * 0.05 +  # Storage cost
            query_frequency_per_day * 30 * 0.0001  # Retrieval cost
        ),
        'query_latency_ms': 150,   # Slightly higher due to network
        'availability': '99.99%'   # S3 durability
    }

    savings = {
        'cost_reduction_percentage': (
            (traditional_storage['total_monthly_cost'] -
             searchable_snapshot['total_monthly_cost']) /
            traditional_storage['total_monthly_cost']
        ) * 100,
        'latency_impact_ms': (
            searchable_snapshot['query_latency_ms'] -
            traditional_storage['query_latency_ms']
        )
    }

    return {
        'traditional': traditional_storage,
        'searchable_snapshot': searchable_snapshot,
        'savings': savings
    }

# Example: 100GB index, 1000 queries/day
result = analyze_searchable_snapshots_benefits(100, 1000)
print(f"Cost reduction: {result['savings']['cost_reduction_percentage']:.1f}%")
# Output: Cost reduction: 76.0%

```

### Cold Phase Query Optimization

**Optimizing queries for cold data:**

```python
def optimize_cold_queries():
    """Best practices for querying cold data"""

    optimization_strategies = {
        'query_design': {
            'use_filters': 'Filters are cached and faster on cold data',
            'limit_result_size': 'Reduce data transfer from object storage',
            'avoid_wildcard_queries': 'Expensive on object storage',
            'use_specific_time_ranges': 'Leverage time-based partitioning'
        },

        'caching_strategies': {
            'request_cache': 'Enable for frequently accessed cold data',
            'query_cache': 'Cache filter results',
            'node_query_cache': 'Warm up frequently accessed data'
        },

        'performance_expectations': {
            'first_query_latency': '500ms - 2s (cache miss)',
            'subsequent_queries': '50-200ms (cache hit)',
            'throughput': 'Lower than hot/warm but sufficient for archive queries'
        }
    }

    return optimization_strategies

```

---

## 🗑️ Delete Phase and Retention Management

### Automated Deletion Policies

**Safe deletion with verification:**

```json
{
  "delete": {
    "min_age": "365d",
    "actions": {
      "wait_for_snapshot": {
        "policy": "daily_snapshots"
      },
      "delete": {}
    }
  }
}

```

**Advanced deletion logic:**

```python
class RetentionManager:
    def __init__(self, es_client):
        self.es = es_client

    def validate_deletion_safety(self, index_name):
        """Validate that index can be safely deleted"""

        safety_checks = {
            'snapshot_exists': self.check_snapshot_exists(index_name),
            'no_active_queries': self.check_active_queries(index_name),
            'compliance_period_expired': self.check_compliance_retention(index_name),
            'stakeholder_approval': self.check_deletion_approval(index_name)
        }

        all_checks_pass = all(safety_checks.values())

        return {
            'safe_to_delete': all_checks_pass,
            'checks': safety_checks,
            'recommendation': 'Proceed with deletion' if all_checks_pass else 'Delay deletion'
        }

    def soft_delete_with_grace_period(self, index_name, grace_period_days=7):
        """Implement soft delete with grace period"""

        # Mark index for deletion
        self.es.indices.put_settings(
            index=index_name,
            body={
                'settings': {
                    'index.blocks.write': True,
                    'index.blocks.read': True,
                    'index.metadata.deletion_scheduled': True,
                    'index.metadata.deletion_date': (
                        datetime.now() + timedelta(days=grace_period_days)
                    ).isoformat()
                }
            }
        )

        # Schedule actual deletion
        self.schedule_delayed_deletion(index_name, grace_period_days)

    def emergency_restore_from_snapshot(self, index_name, snapshot_name):
        """Emergency restore for accidentally deleted data"""

        restore_request = {
            'indices': index_name,
            'rename_pattern': index_name,
            'rename_replacement': f"{index_name}_restored_{int(time.time())}",
            'include_global_state': False
        }

        return self.es.snapshot.restore(
            repository='emergency_restore',
            snapshot=snapshot_name,
            body=restore_request
        )

```

---

## 📊 ILM Monitoring and Troubleshooting

### ILM Health Monitoring

**Monitoring ILM policy execution:**

```python
def monitor_ilm_health(es_client):
    """Comprehensive ILM health monitoring"""

    # Get ILM status
    ilm_status = es_client.ilm.get_status()

    # Get policy execution details
    policies = es_client.ilm.get_lifecycle()

    health_report = {
        'ilm_enabled': ilm_status['operation_mode'] == 'RUNNING',
        'policies': {},
        'problematic_indices': [],
        'stuck_actions': []
    }

    # Check each policy
    for policy_name, policy_config in policies.items():
        policy_health = analyze_policy_health(es_client, policy_name)
        health_report['policies'][policy_name] = policy_health

    # Find stuck indices
    indices_in_policy = es_client.ilm.explain_lifecycle(index='*')

    for index_name, index_ilm in indices_in_policy['indices'].items():
        if 'step_info' in index_ilm and 'error' in index_ilm['step_info']:
            health_report['problematic_indices'].append({
                'index': index_name,
                'error': index_ilm['step_info']['error'],
                'current_step': index_ilm['step'],
                'phase': index_ilm['phase']
            })

    return health_report

def analyze_policy_health(es_client, policy_name):
    """Analyze health of specific ILM policy"""

    # Get indices using this policy
    indices = es_client.cat.indices(
        h='index,health,status,pri,rep,docs.count,store.size',
        s='index'
    )

    policy_indices = [
        idx for idx in indices
        if es_client.indices.get_settings(index=idx['index'])
        .get(idx['index'], {})
        .get('settings', {})
        .get('index', {})
        .get('lifecycle', {})
        .get('name') == policy_name
    ]

    return {
        'total_indices': len(policy_indices),
        'healthy_indices': len([idx for idx in policy_indices if idx['health'] == 'green']),
        'avg_size_gb': calculate_average_size(policy_indices),
        'phase_distribution': calculate_phase_distribution(es_client, policy_indices)
    }

```

### Common ILM Issues and Solutions

**Troubleshooting framework:**

```python
class ILMTroubleshooter:
    def __init__(self, es_client):
        self.es = es_client

    def diagnose_stuck_index(self, index_name):
        """Diagnose why index is stuck in ILM"""

        explanation = self.es.ilm.explain_lifecycle(index=index_name)
        index_info = explanation['indices'][index_name]

        diagnosis = {
            'current_phase': index_info.get('phase'),
            'current_action': index_info.get('action'),
            'current_step': index_info.get('step'),
            'error': index_info.get('step_info', {}).get('error'),
            'possible_causes': [],
            'recommended_actions': []
        }

        # Analyze specific issues
        if 'allocation' in diagnosis['current_action']:
            diagnosis['possible_causes'].append('No nodes match allocation requirements')
            diagnosis['recommended_actions'].append('Check node attributes and allocation settings')

        if 'forcemerge' in diagnosis['current_action']:
            diagnosis['possible_causes'].append('Force merge operation failed or taking too long')
            diagnosis['recommended_actions'].append('Check merge progress and node resources')

        if 'snapshot' in diagnosis['current_action']:
            diagnosis['possible_causes'].append('Snapshot repository unavailable or policy misconfigured')
            diagnosis['recommended_actions'].append('Verify snapshot repository and permissions')

        return diagnosis

    def fix_common_issues(self, index_name, issue_type):
        """Automated fixes for common ILM issues"""

        if issue_type == 'allocation_failed':
            # Retry allocation
            return self.es.cluster.reroute(
                body={
                    'commands': [{
                        'allocate_replica': {
                            'index': index_name,
                            'shard': 0,
                            'node': self.find_available_node()
                        }
                    }]
                }
            )

        elif issue_type == 'force_merge_stuck':
            # Cancel stuck force merge and retry
            self.cancel_force_merge_tasks(index_name)
            return self.retry_ilm_step(index_name)

        elif issue_type == 'snapshot_failed':
            # Retry snapshot creation
            return self.retry_ilm_step(index_name)

    def retry_ilm_step(self, index_name):
        """Retry current ILM step"""
        return self.es.ilm.retry(index=index_name)

```

---

## 💰 Cost Optimization Strategies

### Storage Cost Analysis

**Total Cost of Ownership calculator:**

```python
def calculate_ilm_cost_savings(data_profile):
    """Calculate cost savings from ILM implementation"""

    # Input data profile
    daily_ingest_gb = data_profile['daily_ingest_gb']
    retention_days = data_profile['retention_days']
    replication_factor = data_profile['replication_factor']

    # Storage tier costs (per GB per month)
    costs = {
        'hot_storage': 0.25,    # NVMe SSD
        'warm_storage': 0.10,   # SATA SSD
        'cold_storage': 0.03,   # Object storage
        'network_transfer': 0.01 # Data transfer costs
    }

    # Scenario 1: Traditional (all hot storage)
    traditional_cost = (
        daily_ingest_gb * retention_days *
        replication_factor * costs['hot_storage']
    )

    # Scenario 2: ILM optimized
    hot_days = 7
    warm_days = 23
    cold_days = retention_days - 30

    ilm_cost = (
        (daily_ingest_gb * hot_days * replication_factor * costs['hot_storage']) +
        (daily_ingest_gb * warm_days * 1 * costs['warm_storage']) +  # Reduced replicas
        (daily_ingest_gb * cold_days * 0.5 * costs['cold_storage'])  # Compressed snapshots
    )

    savings = {
        'traditional_monthly_cost': traditional_cost,
        'ilm_monthly_cost': ilm_cost,
        'absolute_savings': traditional_cost - ilm_cost,
        'percentage_savings': ((traditional_cost - ilm_cost) / traditional_cost) * 100,
        'annual_savings': (traditional_cost - ilm_cost) * 12
    }

    return savings

# Example calculation
data_profile = {
    'daily_ingest_gb': 50,      # 50GB per day
    'retention_days': 365,      # 1 year retention
    'replication_factor': 1     # 1 replica
}

savings = calculate_ilm_cost_savings(data_profile)
print(f"Annual savings: ${savings['annual_savings']:,.2f}")
print(f"Percentage savings: {savings['percentage_savings']:.1f}%")

```

### Performance vs Cost Trade-offs

**Optimization decision matrix:**

```python
def optimize_ilm_phases(use_case_requirements):
    """Optimize ILM phases based on use case requirements"""

    optimizations = {
        'real_time_analytics': {
            'hot_duration': '3d',      # Longer hot phase
            'warm_duration': '7d',     # Shorter warm
            'cold_duration': '355d',
            'rationale': 'Prioritize query performance over cost'
        },

        'compliance_logging': {
            'hot_duration': '1d',      # Minimal hot phase
            'warm_duration': '6d',     # Quick transition
            'cold_duration': '358d',   # Long cold storage
            'rationale': 'Minimize cost while meeting compliance requirements'
        },

        'operational_monitoring': {
            'hot_duration': '7d',      # Standard hot phase
            'warm_duration': '23d',    # Balanced approach
            'cold_duration': '335d',
            'rationale': 'Balance between performance and cost'
        },

        'historical_research': {
            'hot_duration': '1d',      # Minimal hot
            'warm_duration': '0d',     # Skip warm phase
            'cold_duration': '364d',   # Maximize cold storage
            'rationale': 'Optimize for long-term storage costs'
        }
    }

    return optimizations.get(
        use_case_requirements['use_case_type'],
        optimizations['operational_monitoring']
    )

```

---

## 🔧 Advanced ILM Patterns

### Custom ILM Actions

**Advanced policy with custom logic:**

```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "30gb",
            "max_age": "1d"
          },
          "set_priority": {"priority": 100}
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0,
            "include": {"data_tier": "data_warm"},
            "require": {"zone": "warm_zone"}
          },
          "readonly": {},
          "forcemerge": {"max_num_segments": 1},
          "shrink": {
            "number_of_shards": 1,
            "allow_write_after_shrink": false
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "cold_repo",
            "force_merge_index": true
          }
        }
      },
      "delete": {
        "min_age": "2555d",
        "actions": {
          "wait_for_snapshot": {
            "policy": "long_term_retention"
          },
          "delete": {}
        }
      }
    }
  }
}

```

### Multi-Tier Storage Integration

**Integrating with cloud storage tiers:**

```python
def design_multi_tier_storage():
    """Design integration with cloud storage tiers"""

    storage_strategy = {
        'hot_tier': {
            'storage': 'Local NVMe SSD',
            'node_type': 'data_hot',
            'use_case': 'Active indexing and frequent queries',
            'cost_factor': '10x',
            'performance': 'Highest'
        },

        'warm_tier': {
            'storage': 'Network attached SSD',
            'node_type': 'data_warm',
            'use_case': 'Read-only queries, moderate frequency',
            'cost_factor': '3x',
            'performance': 'High'
        },

        'cold_tier': {
            'storage': 'S3 Standard-IA via searchable snapshots',
            'node_type': 'data_cold',
            'use_case': 'Occasional queries, archive access',
            'cost_factor': '1x',
            'performance': 'Moderate'
        },

        'frozen_tier': {
            'storage': 'S3 Glacier via searchable snapshots',
            'node_type': 'data_frozen',
            'use_case': 'Compliance, rare access',
            'cost_factor': '0.1x',
            'performance': 'Low (minutes to restore)'
        }
    }

    return storage_strategy

```

---

## 💡 Key Takeaways

✅ **ILM automates data lifecycle management, reducing costs by 70-90% while maintaining query performance**

✅ **Design rollover policies based on data volume patterns and query requirements**

✅ **Use force merge in warm phase to optimize storage and query performance**

✅ **Searchable snapshots enable massive cost savings for cold data storage**

✅ **Monitor ILM health and have troubleshooting procedures for stuck indices**

✅ **Balance performance requirements with cost optimization based on access patterns**

✅ **Plan retention policies with compliance requirements and business needs in mind**

---