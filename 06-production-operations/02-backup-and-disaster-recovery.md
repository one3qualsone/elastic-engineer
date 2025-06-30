# Backup and Disaster Recovery

## Comprehensive Data Protection and Business Continuity

Backup and disaster recovery (DR) isn't just about copying data—it's about ensuring business continuity when everything goes wrong. This guide covers everything from basic snapshot strategies to advanced cross-region disaster recovery architectures.

---

## 🎯 The Reality of Data Protection

### Beyond Simple Backups

Traditional backup thinking focuses on **data preservation**. Modern Elasticsearch DR requires **business continuity planning**—ensuring systems can continue operating under various failure scenarios.

**The Disaster Recovery Hierarchy:**

| Scenario | Impact | RTO Target | RPO Target | Strategy Required |
| --- | --- | --- | --- | --- |
| **Index Corruption** | Single index | < 15 minutes | < 5 minutes | Automated snapshots + quick restore |
| **Node Failure** | Reduced capacity | < 5 minutes | 0 (real-time replicas) | Proper replication + auto-healing |
| **Cluster Failure** | Service down | < 30 minutes | < 15 minutes | Cross-cluster replication |
| **Data Centre Outage** | Complete loss | < 2 hours | < 30 minutes | Multi-region architecture |
| **Region Failure** | Catastrophic loss | < 4 hours | < 1 hour | Global DR architecture |

**RTO vs RPO Reality Check:**

```python
def calculate_business_impact():
    """Understand the true cost of downtime"""

    business_impact = {
        'e_commerce_platform': {
            'revenue_per_minute': 5000,  # £5,000/minute lost revenue
            'rto_target': '5 minutes',   # Max acceptable downtime
            'rpo_target': '1 minute',    # Max acceptable data loss
            'annual_budget_justification': '5000 * 5 * 12 = £300,000 saved vs 1 hour RTO'
        },

        'financial_services': {
            'regulatory_fine_risk': 50000,  # Potential fine per incident
            'reputation_damage': 500000,   # Long-term revenue impact
            'rto_target': '30 seconds',    # Regulatory requirement
            'rpo_target': '0 minutes',     # No data loss acceptable
        },

        'healthcare_records': {
            'patient_safety_risk': 'Incalculable',
            'legal_liability': 1000000,   # Potential lawsuit costs
            'rto_target': '1 minute',     # Life-critical systems
            'rpo_target': '0 minutes',    # Medical data cannot be lost
        }
    }

    return business_impact

```

---

## 📸 Snapshot Management Fundamentals

### Understanding Elasticsearch Snapshots

**What snapshots actually contain:**

```python
def explain_snapshot_contents():
    """Comprehensive breakdown of snapshot data"""

    snapshot_components = {
        'cluster_metadata': {
            'includes': [
                'Index templates and mappings',
                'ILM policies and settings',
                'User roles and permissions',
                'Pipeline configurations',
                'Cluster settings'
            ],
            'restoration_impact': 'Restores complete operational state'
        },

        'index_data': {
            'includes': [
                'All documents and their content',
                'Index settings and mappings',
                'Shard allocation preferences',
                'Alias configurations'
            ],
            'restoration_impact': 'Restores searchable data'
        },

        'what_snapshots_exclude': {
            'real_time_data': 'In-flight indexing operations',
            'node_specific_data': 'Per-node caches and temporary files',
            'security_certificates': 'SSL/TLS certificates and keys',
            'monitoring_data': 'Current performance metrics'
        }
    }

    return snapshot_components

```

### Repository Configuration Strategies

**Choosing the right repository type:**

| Repository Type | Use Case | Performance | Cost | Durability |
| --- | --- | --- | --- | --- |
| **Shared File System** | Local/network storage | High | Low | Medium |
| **S3** | Cloud storage | Medium | Medium | Very High |
| **Azure Blob** | Azure environments | Medium | Medium | Very High |
| **GCS** | Google Cloud | Medium | Medium | Very High |
| **HDFS** | Hadoop ecosystem | High | Low | High |

**Production repository configuration:**

```json
{
  "type": "s3",
  "settings": {
    "bucket": "production-elasticsearch-backups",
    "region": "eu-west-1",
    "base_path": "cluster-production",
    "compress": true,
    "chunk_size": "1gb",
    "server_side_encryption": true,
    "storage_class": "STANDARD_IA",
    "max_restore_bytes_per_sec": "500mb",
    "max_snapshot_bytes_per_sec": "500mb",
    "readonly": false
  }
}

```

### Snapshot Lifecycle Management (SLM)

**Automated backup policies:**

```python
def design_slm_strategy():
    """Comprehensive SLM policy design"""

    slm_policies = {
        'critical_data_policy': {
            'schedule': '0 30 1,13 * * ?',  # Every 12 hours
            'name': '<critical-{now/d}>',
            'repository': 'primary_backup_repo',
            'config': {
                'indices': ['user-data-*', 'financial-*', 'audit-*'],
                'include_global_state': True,
                'partial': False
            },
            'retention': {
                'expire_after': '30d',
                'min_count': 5,
                'max_count': 100
            }
        },

        'bulk_data_policy': {
            'schedule': '0 0 2 * * ?',      # Daily at 2 AM
            'name': '<bulk-{now/d}>',
            'repository': 'bulk_backup_repo',
            'config': {
                'indices': ['logs-*', 'metrics-*'],
                'include_global_state': False,
                'partial': True
            },
            'retention': {
                'expire_after': '7d',
                'min_count': 7,
                'max_count': 30
            }
        },

        'configuration_policy': {
            'schedule': '0 0 4 * * SUN',    # Weekly on Sunday
            'name': '<config-{now/w}>',
            'repository': 'config_backup_repo',
            'config': {
                'indices': [],              # No index data
                'include_global_state': True,
                'partial': False
            },
            'retention': {
                'expire_after': '90d',
                'min_count': 12,
                'max_count': 52
            }
        }
    }

    return slm_policies

```

---

## 🔄 Advanced Snapshot Strategies

### Incremental Snapshot Optimization

**Understanding snapshot efficiency:**

```python
def optimize_snapshot_performance():
    """Advanced snapshot optimization techniques"""

    optimization_strategies = {
        'incremental_snapshots': {
            'how_they_work': 'Only backup changed segments since last snapshot',
            'storage_savings': '80-95% for stable indices',
            'time_savings': '70-90% for subsequent snapshots',
            'implementation': 'Automatic with proper repository configuration'
        },

        'compression_strategies': {
            'algorithm': 'LZ4 (default) vs DEFLATE',
            'trade_offs': {
                'LZ4': 'Faster compression, larger size',
                'DEFLATE': 'Better compression ratio, slower'
            },
            'recommendation': 'Use LZ4 for frequent snapshots, DEFLATE for archival'
        },

        'parallel_processing': {
            'concurrent_streams': 'Configure based on repository bandwidth',
            'max_snapshot_bytes_per_sec': 'Balance between backup speed and cluster impact',
            'max_restore_bytes_per_sec': 'Optimize for restore time requirements',
            'node_concurrent_recoveries': 'Control resource usage during restore'
        },

        'storage_optimization': {
            'lifecycle_policies': 'Automatically transition to cheaper storage tiers',
            'cross_region_replication': 'Replicate critical snapshots for DR',
            'encryption_at_rest': 'Ensure compliance with data protection regulations',
            'access_logging': 'Monitor snapshot access patterns for optimization'
        }
    }

    return optimization_strategies

```

### Searchable Snapshots for Cold Storage

**Implementing cost-effective long-term storage:**

```python
def implement_searchable_snapshots():
    """Searchable snapshots for massive cost savings"""

    searchable_snapshot_strategy = {
        'cost_benefits': {
            'storage_cost_reduction': '90-95% vs. local storage',
            'compute_cost_reduction': '70-80% vs. full replicas',
            'operational_complexity': 'Reduced - automatic management',
            'query_performance': 'Acceptable for analytical workloads'
        },

        'implementation_pattern': {
            'hot_phase': 'Local NVMe SSD for active data (7 days)',
            'warm_phase': 'Local SATA SSD for recent data (23 days)',
            'cold_phase': 'Searchable snapshots for archive data (11 months)',
            'frozen_phase': 'Compressed snapshots for compliance (years)'
        },

        'query_characteristics': {
            'first_query_latency': '500ms - 2 seconds (cache miss)',
            'subsequent_queries': '50-200ms (cache hit)',
            'concurrent_query_limit': 'Lower than hot storage',
            'best_use_cases': 'Reporting, compliance, historical analysis'
        },

        'configuration_example': {
            'repository_setup': 'S3 with intelligent tiering',
            'mount_type': 'fully_mounted for frequent access',
            'cache_configuration': 'Shared cache across nodes',
            'monitoring_requirements': 'Track cache hit rates and query performance'
        }
    }

    return searchable_snapshot_strategy

```

---

## 🌍 Cross-Cluster Replication (CCR)

### Real-Time Data Replication

**Implementing active-passive DR architectures:**

```python
def design_ccr_architecture():
    """Cross-cluster replication for disaster recovery"""

    ccr_patterns = {
        'active_passive_dr': {
            'primary_cluster': {
                'location': 'Primary data centre',
                'role': 'Handles all read/write operations',
                'hardware': 'High-performance nodes',
                'monitoring': 'Full monitoring and alerting'
            },
            'dr_cluster': {
                'location': 'Secondary data centre/region',
                'role': 'Receives replicated data via CCR',
                'hardware': 'Can be lower spec for cost optimization',
                'monitoring': 'Replication lag and cluster health'
            },
            'failover_process': {
                'detection': 'Primary cluster health monitoring',
                'dns_switchover': 'Update DNS to point to DR cluster',
                'application_restart': 'Restart applications with new endpoints',
                'data_verification': 'Verify data integrity post-failover'
            }
        },

        'bi_directional_replication': {
            'use_case': 'Global deployment with regional access',
            'complexity': 'Higher - requires conflict resolution',
            'benefits': 'Lower latency for global users',
            'challenges': 'Split-brain scenarios, data consistency'
        },

        'selective_replication': {
            'critical_indices_only': 'Replicate only business-critical data',
            'cost_optimization': 'Reduce DR cluster resource requirements',
            'recovery_time_trade_off': 'Faster for critical systems, manual for others'
        }
    }

    return ccr_patterns

```

### CCR Configuration and Monitoring

**Setting up production-ready CCR:**

```bash
# Configure remote cluster connection
PUT /_cluster/settings
{
  "persistent": {
    "cluster.remote.dr_cluster.seeds": [
      "dr-node1:9300",
      "dr-node2:9300",
      "dr-node3:9300"
    ],
    "cluster.remote.dr_cluster.skip_unavailable": false
  }
}

# Set up follower index with specific settings
PUT /follower_index/_ccr/follow
{
  "remote_cluster": "dr_cluster",
  "leader_index": "leader_index",
  "settings": {
    "index.number_of_replicas": 0,
    "index.number_of_shards": 1
  },
  "max_read_request_operation_count": 5120,
  "max_outstanding_read_requests": 12,
  "max_read_request_size": "32mb",
  "max_write_request_operation_count": 5120,
  "max_outstanding_write_requests": 9,
  "max_write_request_size": "9mb",
  "max_write_buffer_count": 2147483647,
  "max_write_buffer_size": "512mb",
  "max_retry_delay": "500ms",
  "read_poll_timeout": "1m"
}

```

**CCR monitoring and alerting:**

```python
def monitor_ccr_health():
    """Comprehensive CCR monitoring strategy"""

    ccr_monitoring = {
        'replication_metrics': {
            'replication_lag': {
                'measurement': 'Time difference between leader and follower',
                'alert_threshold': '> 30 seconds',
                'action': 'Investigate network or resource constraints'
            },
            'operations_behind': {
                'measurement': 'Number of operations follower is behind',
                'alert_threshold': '> 1000 operations',
                'action': 'Check follower cluster capacity'
            },
            'failed_read_requests': {
                'measurement': 'Failed reads from leader cluster',
                'alert_threshold': '> 1% error rate',
                'action': 'Check network connectivity and leader health'
            }
        },

        'follower_cluster_health': {
            'indexing_performance': 'Monitor follower indexing throughput',
            'resource_utilization': 'Ensure adequate CPU/memory/disk',
            'queue_sizes': 'Monitor CCR thread pool queues',
            'error_rates': 'Track CCR-specific errors and rejections'
        },

        'network_monitoring': {
            'bandwidth_utilization': 'Monitor network usage between clusters',
            'latency_measurements': 'Track inter-cluster communication latency',
            'connection_stability': 'Monitor connection drops and retries',
            'security_compliance': 'Ensure encrypted communication'
        }
    }

    return ccr_monitoring

```

---

## 🚨 Disaster Recovery Procedures

### Failure Scenario Planning

**Comprehensive DR playbooks:**

```python
def create_dr_playbooks():
    """Detailed procedures for various disaster scenarios"""

    disaster_scenarios = {
        'single_node_failure': {
            'detection': 'Node leaves cluster, cluster health yellow/red',
            'immediate_actions': [
                'Verify node is truly offline (network, hardware)',
                'Check if automatic shard reallocation is occurring',
                'Monitor cluster stability during reallocation',
                'Scale cluster if needed to maintain capacity'
            ],
            'recovery_steps': [
                'Fix underlying issue (hardware, network, configuration)',
                'Restart node with proper cluster discovery settings',
                'Monitor shard rebalancing and cluster health',
                'Update monitoring and alerting based on lessons learned'
            ],
            'rto_target': '< 10 minutes',
            'rpo_target': '0 (no data loss with proper replication)'
        },

        'complete_cluster_failure': {
            'detection': 'All nodes unreachable, applications cannot connect',
            'immediate_actions': [
                'Activate DR cluster if available',
                'Update DNS/load balancer to point to DR cluster',
                'Notify stakeholders of service switch',
                'Begin investigation of primary cluster failure'
            ],
            'recovery_steps': [
                'Identify root cause of cluster failure',
                'Rebuild/repair primary cluster infrastructure',
                'Restore from latest snapshots if data corruption',
                'Sync any changes made during DR operation',
                'Plan and execute failback to primary cluster'
            ],
            'rto_target': '< 2 hours',
            'rpo_target': '< 15 minutes (with CCR)'
        },

        'data_corruption': {
            'detection': 'Search errors, inconsistent results, shard failures',
            'immediate_actions': [
                'Stop all indexing operations to prevent further corruption',
                'Isolate affected indices/shards',
                'Identify scope of corruption through verification queries',
                'Notify users of potential data inconsistencies'
            ],
            'recovery_steps': [
                'Restore affected indices from clean snapshots',
                'Replay any missed data from application logs/queues',
                'Verify data integrity through sampling and validation',
                'Resume normal operations after validation'
            ],
            'rto_target': '< 4 hours',
            'rpo_target': 'Depends on snapshot frequency'
        }
    }

    return disaster_scenarios

```

### Data Recovery Procedures

**Step-by-step recovery processes:**

```python
def implement_recovery_procedures():
    """Detailed recovery implementation guide"""

    recovery_procedures = {
        'full_cluster_restore': {
            'preparation': [
                'Ensure target cluster has sufficient capacity',
                'Verify repository access and permissions',
                'Plan restoration order (configuration first, then data)',
                'Prepare rollback plan in case of issues'
            ],
            'execution_steps': [
                '1. Restore cluster configuration and templates',
                '2. Restore critical indices first (user data, security)',
                '3. Restore remaining indices in priority order',
                '4. Verify data integrity and completeness',
                '5. Resume normal operations gradually'
            ],
            'validation_requirements': [
                'Document count verification',
                'Sample data integrity checks',
                'Application functionality testing',
                'Performance baseline comparison'
            ]
        },

        'selective_index_restore': {
            'use_cases': [
                'Single index corruption',
                'Accidental data deletion',
                'Index mapping issues',
                'Performance optimization rollback'
            ],
            'process': [
                'Create temporary index with snapshot data',
                'Verify restored data quality and completeness',
                'Switch aliases to point to restored index',
                'Delete corrupted index after validation'
            ],
            'considerations': [
                'Index version compatibility',
                'Mapping changes since snapshot',
                'Ongoing write operations',
                'Application downtime requirements'
            ]
        },

        'point_in_time_recovery': {
            'scenario': 'Need to recover to specific timestamp',
            'challenges': [
                'Elasticsearch snapshots are not point-in-time consistent',
                'Need to combine snapshot + transaction log replay',
                'Complex coordination across multiple indices'
            ],
            'implementation': [
                'Use application-level transaction logs',
                'Implement event sourcing patterns',
                'Maintain external change logs',
                'Consider using database features for ACID requirements'
            ]
        }
    }

    return recovery_procedures

```

---

## 🔧 Advanced DR Architectures

### Multi-Region Disaster Recovery

**Global resilience strategies:**

```python
def design_global_dr_architecture():
    """Multi-region disaster recovery implementation"""

    global_dr_design = {
        'three_region_strategy': {
            'primary_region': {
                'location': 'eu-west-1',
                'role': 'Active production cluster',
                'capacity': '100% of production load',
                'replication_targets': ['us-east-1', 'ap-southeast-1']
            },
            'secondary_region': {
                'location': 'us-east-1',
                'role': 'Hot standby via CCR',
                'capacity': '80% of production load',
                'failover_time': '< 30 minutes'
            },
            'tertiary_region': {
                'location': 'ap-southeast-1',
                'role': 'Cold standby with snapshots',
                'capacity': '50% of production load',
                'failover_time': '< 2 hours'
            }
        },

        'traffic_routing': {
            'global_load_balancer': 'Route 53 health checks',
            'dns_failover': 'Automatic DNS updates on failure',
            'application_awareness': 'Applications detect and adapt to region changes',
            'user_session_handling': 'Stateless design or session replication'
        },

        'data_consistency': {
            'eventually_consistent': 'Accept brief inconsistency during failover',
            'conflict_resolution': 'Last-write-wins or application-specific logic',
            'data_validation': 'Post-failover consistency checks',
            'rollback_procedures': 'Process for reverting problematic changes'
        }
    }

    return global_dr_design

```

### Backup Verification and Testing

**Ensuring backups actually work:**

```python
def implement_backup_verification():
    """Comprehensive backup testing strategy"""

    verification_strategy = {
        'automated_testing': {
            'snapshot_integrity': {
                'frequency': 'Every snapshot creation',
                'tests': [
                    'Verify snapshot completion without errors',
                    'Check snapshot metadata consistency',
                    'Validate repository accessibility',
                    'Test partial restore of sample data'
                ]
            },
            'restore_testing': {
                'frequency': 'Weekly for critical data, monthly for bulk data',
                'process': [
                    'Restore to isolated test cluster',
                    'Verify document counts and data integrity',
                    'Run application test suites',
                    'Performance test critical queries'
                ]
            }
        },

        'dr_rehearsals': {
            'tabletop_exercises': {
                'frequency': 'Quarterly',
                'participants': 'All teams involved in DR process',
                'scenarios': 'Various failure modes and recovery procedures',
                'outcomes': 'Updated procedures and improved coordination'
            },
            'full_failover_tests': {
                'frequency': 'Annually or after major changes',
                'scope': 'Complete production failover simulation',
                'coordination': 'All stakeholders and dependent systems',
                'success_criteria': 'RTO/RPO targets met with minimal issues'
            }
        },

        'continuous_validation': {
            'monitoring_backup_quality': 'Track backup size, duration, and success rates',
            'repository_health_checks': 'Regular connectivity and permission verification',
            'cross_region_replication_monitoring': 'CCR lag and error tracking',
            'compliance_reporting': 'Regular reports for auditing and compliance'
        }
    }

    return verification_strategy

```

---

## 📋 DR Planning and Documentation

### Business Continuity Planning

**Aligning technical DR with business requirements:**

```python
def develop_business_continuity_plan():
    """Comprehensive business continuity planning"""

    bcp_framework = {
        'business_impact_analysis': {
            'critical_functions': {
                'user_authentication': 'RTO: 5 minutes, RPO: 0',
                'search_functionality': 'RTO: 15 minutes, RPO: 5 minutes',
                'data_ingestion': 'RTO: 30 minutes, RPO: 15 minutes',
                'reporting_analytics': 'RTO: 2 hours, RPO: 1 hour'
            },
            'dependency_mapping': {
                'upstream_systems': 'Data sources, authentication systems',
                'downstream_systems': 'Applications, dashboards, reports',
                'external_services': 'Cloud providers, network services',
                'human_resources': 'On-call teams, escalation procedures'
            }
        },

        'recovery_strategies': {
            'immediate_response': '0-30 minutes',
            'short_term_recovery': '30 minutes - 4 hours',
            'long_term_recovery': '4 hours - 24 hours',
            'business_restoration': '24+ hours'
        },

        'communication_plan': {
            'internal_notification': 'Escalation matrix for different severity levels',
            'customer_communication': 'Status page updates and direct notification',
            'stakeholder_updates': 'Regular updates to executives and affected teams',
            'post_incident_communication': 'Post-mortem reports and lessons learned'
        }
    }

    return bcp_framework

```

### DR Documentation Requirements

**Essential documentation for effective DR:**

```python
def create_dr_documentation():
    """Comprehensive DR documentation structure"""

    documentation_requirements = {
        'runbooks': {
            'emergency_contact_list': 'Current contact information for all team members',
            'step_by_step_procedures': 'Detailed commands and actions for each scenario',
            'decision_trees': 'Flowcharts for determining appropriate response',
            'rollback_procedures': 'How to reverse actions if recovery fails'
        },

        'technical_documentation': {
            'architecture_diagrams': 'Current and DR cluster architectures',
            'configuration_details': 'All relevant settings and dependencies',
            'network_topology': 'Connectivity requirements and firewall rules',
            'credential_management': 'Secure access to recovery resources'
        },

        'operational_procedures': {
            'change_management': 'How DR procedures are updated and tested',
            'access_controls': 'Who can execute DR procedures',
            'audit_requirements': 'Logging and reporting for DR activities',
            'training_programs': 'Regular training for DR team members'
        },

        'compliance_documentation': {
            'regulatory_requirements': 'Specific compliance obligations',
            'audit_trails': 'Documentation of DR tests and actual incidents',
            'data_protection': 'GDPR, HIPAA, and other privacy considerations',
            'business_continuity_reporting': 'Regular reports to management and auditors'
        }
    }

    return documentation_requirements

```

---

## 💡 Key Takeaways

✅ **Design DR strategy around business requirements, not technical capabilities**

✅ **Implement multiple layers of protection: replication, snapshots, and cross-region DR**

✅ **Test backup and recovery procedures regularly - untested backups are worthless**

✅ **Monitor all aspects of DR infrastructure proactively**

✅ **Document everything and train team members on procedures**

✅ **Consider searchable snapshots for cost-effective long-term data retention**

✅ **Plan for various failure scenarios, not just the obvious ones**

---