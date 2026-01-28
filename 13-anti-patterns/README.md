# Chapter 17: Performance Anti-Patterns

## Overview

Anti-patterns are common approaches that seem reasonable but lead to performance, scalability, or reliability problems in cloud applications. Recognizing and avoiding these patterns is essential for Cloud Architects.

## Why Anti-Patterns Occur

1. **Migration from on-premises** - Patterns that worked on-prem don't translate to cloud
2. **Incremental development** - New features introduce problems into clean designs
3. **Insufficient testing** - Can't simulate real production loads
4. **Wrong abstractions** - Over-engineering or premature optimization

## Anti-Pattern Catalog

| Anti-Pattern | Problem | Impact |
|--------------|---------|--------|
| [Busy Database](01-busy-database.md) | Database does too much processing | CPU, throughput limits |
| [Busy Front End](02-busy-frontend.md) | UI thread handles resource-intensive work | Poor user experience |
| [Chatty I/O](03-chatty-io.md) | Too many small network requests | Latency, bandwidth |
| [Extraneous Fetching](04-extraneous-fetching.md) | Retrieving more data than needed | Wasted bandwidth |
| [Improper Instantiation](05-improper-instantiation.md) | Creating/destroying reusable objects | Memory, CPU |
| [Monolithic Persistence](06-monolithic-persistence.md) | Single data store for all patterns | Bottleneck |
| [No Caching](07-no-caching.md) | Failing to cache frequently accessed data | Unnecessary load |
| [Noisy Neighbor](08-noisy-neighbor.md) | One tenant consumes shared resources | Service degradation |
| [Retry Storm](09-retry-storm.md) | Excessive retries overwhelm services | Cascading failures |
| [Synchronous I/O](10-synchronous-io.md) | Blocking threads during I/O | Poor scalability |

## Quick Reference: Anti-Pattern Detection

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    ANTI-PATTERN DETECTION GUIDE                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   SYMPTOM                              LIKELY ANTI-PATTERN                       │
│   ───────                              ──────────────────                        │
│                                                                                  │
│   Database CPU constantly high    ──►  Busy Database                            │
│   UI freezes during operations    ──►  Busy Front End                           │
│   High latency, low throughput    ──►  Chatty I/O                               │
│   Large memory allocations        ──►  Extraneous Fetching                      │
│   GC pressure, memory spikes      ──►  Improper Instantiation                   │
│   All queries slow                ──►  Monolithic Persistence                   │
│   Repeated identical queries      ──►  No Caching                               │
│   One user affects all users      ──►  Noisy Neighbor                           │
│   Service failures cascade        ──►  Retry Storm                              │
│   Thread pool exhaustion          ──►  Synchronous I/O                          │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Anti-Pattern Severity Matrix

| Anti-Pattern | Performance | Scalability | Reliability | Cost |
|--------------|-------------|-------------|-------------|------|
| Busy Database | High | High | Medium | High |
| Busy Front End | High | Low | Low | Low |
| Chatty I/O | High | High | Medium | Medium |
| Extraneous Fetching | Medium | Medium | Low | High |
| Improper Instantiation | Medium | Medium | Low | Low |
| Monolithic Persistence | High | High | High | High |
| No Caching | High | High | Medium | High |
| Noisy Neighbor | High | High | High | Medium |
| Retry Storm | High | High | High | Medium |
| Synchronous I/O | High | High | Medium | Low |

---

## Anti-Pattern Summaries

### 1. Busy Database

**Problem**: Offloading too much processing to the database (stored procedures, views with complex logic, data transformation).

**Symptoms**:
- Database CPU constantly high
- Database becomes bottleneck
- Can't scale application independently

**Solution**:
- Move processing to application tier
- Use appropriate compute services (Functions, App Service)
- Database should store/retrieve, not compute

### 2. Busy Front End

**Problem**: Resource-intensive work running on the UI thread, blocking user interaction.

**Symptoms**:
- UI freezes or becomes unresponsive
- User experience degrades under load
- Long response times

**Solution**:
- Move intensive work to background threads
- Use async/await patterns
- Offload to worker services or Azure Functions

### 3. Chatty I/O

**Problem**: Making many small network requests instead of fewer, larger requests.

**Symptoms**:
- High latency despite low payload
- Network bandwidth not fully utilized
- Many roundtrips for single operations

**Solution**:
- Batch requests where possible
- Use bulk APIs
- Implement data aggregation endpoints

### 4. Extraneous Fetching

**Problem**: Retrieving more data than needed, often entire entities when only a few fields are required.

**Symptoms**:
- Large response payloads
- Memory pressure
- Unnecessary network traffic

**Solution**:
- Use projection (SELECT specific columns)
- Implement GraphQL for flexible querying
- Paginate large result sets

### 5. Improper Instantiation

**Problem**: Repeatedly creating and destroying objects that should be shared (HTTP clients, database connections).

**Symptoms**:
- High memory allocation rates
- GC pressure
- Socket exhaustion

**Solution**:
- Use dependency injection with proper lifetimes
- Implement object pooling
- Use IHttpClientFactory

### 6. Monolithic Persistence

**Problem**: Using a single data store for all data regardless of access patterns.

**Symptoms**:
- Everything goes to one database
- Can't optimize for different workloads
- Single point of failure

**Solution**:
- Polyglot persistence (right tool for the job)
- Separate OLTP from OLAP
- Use caching for read-heavy data

### 7. No Caching

**Problem**: Repeatedly fetching rarely-changing data from slower data stores.

**Symptoms**:
- Repeated identical database queries
- Slow response times
- Unnecessary database load

**Solution**:
- Implement distributed caching (Redis)
- Use CDN for static content
- Cache at multiple layers

### 8. Noisy Neighbor

**Problem**: In multi-tenant systems, one tenant consumes disproportionate resources.

**Symptoms**:
- Performance varies unpredictably
- Some tenants experience degradation
- Resource exhaustion

**Solution**:
- Implement tenant isolation
- Use resource quotas
- Consider tenant-per-database for enterprise customers

### 9. Retry Storm

**Problem**: When services fail, aggressive retries from many clients overwhelm them, preventing recovery.

**Symptoms**:
- Cascading failures
- Services can't recover
- Load increases during failures

**Solution**:
- Exponential backoff with jitter
- Circuit breakers
- Bulkhead isolation

### 10. Synchronous I/O

**Problem**: Blocking threads while waiting for I/O operations to complete.

**Symptoms**:
- Thread pool exhaustion
- Poor scalability under load
- Timeouts

**Solution**:
- Use async/await consistently
- Avoid `.Result` and `.Wait()`
- Configure async pipeline

---

## Learning Objectives

After completing this chapter, you will:

1. **Identify** common anti-patterns in existing systems
2. **Diagnose** performance issues using symptom analysis
3. **Remediate** anti-patterns with proven solutions
4. **Prevent** anti-patterns through design reviews
5. **Monitor** for anti-pattern emergence

---

*Continue to [Busy Database](01-busy-database.md)*

*Back to [Main Guide](../README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
