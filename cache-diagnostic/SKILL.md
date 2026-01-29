---
name: cache-diagnostic
description: Specialized Magento 2 cache diagnostic skill for analyzing cache configuration, performance, hit rates, and optimization opportunities across all cache layers (Full Page Cache, Application Cache, Redis, Varnish).
---

This skill provides deep cache analysis and optimization recommendations for Magento 2.

## What This Skill Does

1. **Cache Configuration Analysis**
   - Cache type status for all 15+ Magento cache types
   - Backend configuration (File, Redis, Memcached)
   - Full Page Cache (FPC) backend (Varnish, Built-in, Redis)
   - Cache prefix and tag configuration
   - Cache lifetime settings

2. **Redis Analysis** (if applicable)
   - Redis connection and version
   - Memory usage and fragmentation
   - Keyspace analysis and database usage
   - Eviction policy and maxmemory settings
   - Hit rate statistics
   - Slow log analysis

3. **Varnish Analysis** (if applicable)
   - Varnish version and configuration
   - Hit rate and cache effectiveness
   - Backend health and response times
   - ESI hole configuration
   - Cache invalidation (PURGE) functionality
   - VCL configuration validation

4. **Cache Performance Metrics**
   - Cache hit/miss ratios per cache type
   - Cache size and growth trends
   - Cache warming status
   - Invalidation frequency and patterns
   - Cache-related errors and warnings

5. **Application Cache Analysis**
   - Config cache efficiency
   - Layout cache effectiveness
   - Block HTML cache usage
   - Collections cache status
   - EAV cache efficiency

## Diagnostic Commands

```bash
# Cache Status
bin/magento cache:status
bin/magento config:show --scope=default | grep cache

# Redis Analysis
redis-cli INFO
redis-cli INFO stats
redis-cli --latency
redis-cli SLOWLOG GET 10

# Varnish Analysis
varnishstat -1
varnishlog -q "RespStatus >= 400" -n 100
varnishadm vcl.list
