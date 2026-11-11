# Redis Cache Eviction Incident Runbook

## Overview
When Redis memory usage exceeds 95% and eviction errors occur on cache-cluster-01:

1. Check current memory usage using redis-cli info memory.
2. Verify if the maxmemory-policy is set to volatile-lru or allkeys-lru.
3. If memory fragmentation ratio is above 1.5, trigger memory defragmentation by setting activedefrag yes.
4. Scale up the Redis cache node size or increase shard count via Terraform module redis-cluster.
