# VOLUND Operational Runbook

Last updated: 2026-03-28

This runbook covers operational procedures for the VOLUND multi-tenant AI agent platform. Each section includes symptoms, investigation steps, and remediation procedures.

---

## Table of Contents

1. [Health Checks and Monitoring](#1-health-checks-and-monitoring)
2. [Warm Pool Scaling](#2-warm-pool-scaling)
3. [Claim Latency Troubleshooting](#3-claim-latency-troubleshooting)
4. [NATS Failure Scenarios](#4-nats-failure-scenarios)
5. [Database Issues](#5-database-issues)
6. [Redis Failures](#6-redis-failures)
7. [Agent Pod Crashes and Stuck Pods](#7-agent-pod-crashes-and-stuck-pods)
8. [Quota and Billing Issues](#8-quota-and-billing-issues)
9. [Security Incidents](#9-security-incidents)
10. [Disaster Recovery](#10-disaster-recovery)

---

## 1. Health Checks and Monitoring

### Service Endpoints

| Component        | Health Endpoint             | Port  |
|------------------|-----------------------------|-------|
| volund gateway   | `GET /healthz`              | 8080  |
| volund gRPC      | gRPC health check           | 9090  |
| volund metrics   | `GET /metrics`              | 8080  |
| volund-operator  | `GET /healthz`              | 8080  |
| volund-agent     | Heartbeat via NATS          | --    |

### Quick Health Check

```bash
# Gateway health
kubectl exec -it deploy/volund-gateway -n volund -- curl -s localhost:8080/healthz

# Operator health
kubectl get pods -n volund -l app=volund-operator -o wide

# All VOLUND pods status
kubectl get pods -n volund --sort-by=.status.startTime

# Check CRD status
kubectl get agentwarmpool -A
kubectl get agentinstance -A
kubectl get agentprofile -A
kubectl get skill -A
```

### Key Prometheus Queries

```promql
# Claim latency percentiles
histogram_quantile(0.50, rate(volund_claim_duration_seconds_bucket[5m]))
histogram_quantile(0.95, rate(volund_claim_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(volund_claim_duration_seconds_bucket[5m]))

# Active instances per tenant
volund_active_instances{} by (tenant)

# Warm pool utilization
volund_active_instances / volund_warm_pool_size

# Claim error rate
rate(volund_claim_errors_total[5m]) / rate(volund_claim_total[5m])
```

### Alert Reference

| Alert                      | Condition                | Severity |
|----------------------------|--------------------------|----------|
| ClaimLatencyP95High        | P95 > 200ms             | Warning  |
| ClaimLatencyP99Critical    | P99 > 500ms             | Critical |
| WarmPoolExhaustion         | >10% unavailable        | Warning  |
| WarmPoolDown               | No successful claims     | Critical |
| ClaimErrorRate             | >5% errors              | Warning  |
| WarmPoolHighUtilization    | >80% utilization        | Warning  |

### SLA Targets

- P50 claim latency: < 50ms
- P95 claim latency: < 200ms
- P99 claim latency: < 500ms

---

## 2. Warm Pool Scaling

### Symptoms

- `WarmPoolHighUtilization` alert firing (>80% utilization).
- `WarmPoolExhaustion` alert firing (>10% pods unavailable).
- Elevated claim latency as pool runs low.
- Tenants reporting slow agent startup.

### Investigation

```bash
# Check current warm pool status for all tenants
kubectl get agentwarmpool -A -o wide

# Inspect a specific warm pool
kubectl describe agentwarmpool <pool-name> -n <tenant-ns>

# Check current utilization
kubectl get agentwarmpool -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.ready}/{.spec.minReady}{"\t"}{.status.active}/{.spec.maxSize}{"\n"}{end}'

# Review recent scaling events
kubectl get events -n <tenant-ns> --field-selector reason=ScalingWarmPool --sort-by=.lastTimestamp

# Node resource availability
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources"
```

### Remediation

**Increase minReady (more warm pods standing by):**

```bash
kubectl patch agentwarmpool <pool-name> -n <tenant-ns> \
  --type merge -p '{"spec":{"minReady": 10}}'
```

**Increase maxSize (allow more concurrent instances):**

```bash
kubectl patch agentwarmpool <pool-name> -n <tenant-ns> \
  --type merge -p '{"spec":{"maxSize": 50}}'
```

**Adjust scaling thresholds:**

```bash
kubectl patch agentwarmpool <pool-name> -n <tenant-ns> \
  --type merge -p '{"spec":{"scaleUpThreshold": 0.7, "scaleDownThreshold": 0.3}}'
```

**Emergency: scale up nodes if cluster resources are exhausted:**

```bash
# Check pending pods
kubectl get pods -n <tenant-ns> --field-selector=status.phase=Pending

# If using cluster autoscaler, verify it is active
kubectl get pods -n kube-system -l app=cluster-autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=50
```

**Validation after changes:**

```bash
# Watch pool converge to new minReady
kubectl get agentwarmpool <pool-name> -n <tenant-ns> -w

# Confirm new pods are entering Ready state
kubectl get pods -n <tenant-ns> -l volund.io/pool=<pool-name> --sort-by=.status.startTime
```

---

## 3. Claim Latency Troubleshooting

### Symptoms

- `ClaimLatencyP95High` or `ClaimLatencyP99Critical` alerts firing.
- Users experiencing slow conversation starts (>500ms).
- Claim latency histogram showing distribution shift.

### Investigation

```bash
# Check claim latency distribution in Prometheus
# P50, P95, P99 over last 15 minutes
# Use the Prometheus queries from Section 1

# Check if warm pool has available pods
kubectl get agentwarmpool -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}ready={.status.ready}{"\t"}active={.status.active}{"\n"}{end}'

# Check gateway logs for slow claims
kubectl logs deploy/volund-gateway -n volund --since=10m | grep -i "claim" | grep -E "duration_ms=[0-9]{3,}"

# Check for pod scheduling delays
kubectl get events -n <tenant-ns> --sort-by=.lastTimestamp | grep -i "schedule\|pull\|create"

# Check Redis routing table performance (used during claim)
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc SLOWLOG GET 10

# Check NATS connectivity from gateway
kubectl exec -it deploy/volund-gateway -n volund -- nats server ping
```

### Common Causes and Remediation

**Warm pool depleted:**
See Section 2 for scaling procedures. Increase minReady to maintain a buffer.

**Redis routing table slow:**
See Section 6. Check Redis memory and connection count.

**Pod startup slow (image pull):**
```bash
# Check if image pull is the bottleneck
kubectl get events -n <tenant-ns> --field-selector reason=Pulling --sort-by=.lastTimestamp

# Ensure images are cached on nodes
kubectl get pods -n <tenant-ns> -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}' | sort -u
```

**Network latency between gateway and agent pods:**
```bash
# Test latency from gateway to an agent pod
kubectl exec -it deploy/volund-gateway -n volund -- curl -o /dev/null -s -w "time_connect: %{time_connect}\ntime_total: %{time_total}\n" http://<agent-pod-ip>:8080/healthz
```

**NATS message delivery slow:**
See Section 4. Check NATS server health and subject queue depth.

---

## 4. NATS Failure Scenarios

### 4a. NATS Connection Loss

#### Symptoms

- Gateway logs: `nats: connection lost` or `nats: reconnecting`.
- Agent heartbeats stop arriving on `volund.events.io.volund.agent.heartbeat`.
- Claims fail with NATS timeout errors.
- Conversations hang mid-stream.

#### Investigation

```bash
# Check NATS server pods
kubectl get pods -n volund -l app=nats

# Check NATS server health
kubectl exec -it nats-0 -n volund -- nats server report connections
kubectl exec -it nats-0 -n volund -- nats server report jetstream

# Check gateway NATS connection status
kubectl logs deploy/volund-gateway -n volund --since=5m | grep -i "nats"

# Check agent NATS connection status
kubectl logs -n <tenant-ns> -l volund.io/component=agent --since=5m | grep -i "nats"

# Verify NATS cluster health
kubectl exec -it nats-0 -n volund -- nats server list
```

#### Remediation

```bash
# If NATS pods are in CrashLoopBackOff, check storage
kubectl describe pod nats-0 -n volund
kubectl exec -it nats-0 -n volund -- df -h /data

# Restart NATS pods one at a time (rolling)
kubectl delete pod nats-0 -n volund
# Wait for nats-0 to be Running before proceeding
kubectl wait --for=condition=Ready pod/nats-0 -n volund --timeout=120s
kubectl delete pod nats-1 -n volund
kubectl wait --for=condition=Ready pod/nats-1 -n volund --timeout=120s

# Force gateway reconnection if stuck
kubectl rollout restart deploy/volund-gateway -n volund
```

### 4b. Message Loss / Delivery Failures

#### Symptoms

- Agent tasks dispatched but never received (check `volund.pool.{profileName}` subjects).
- Conversation streams (`volund.conv.{convId}.stream`) missing events.
- Users see partial or stalled agent responses.

#### Investigation

```bash
# Check subject statistics
kubectl exec -it nats-0 -n volund -- nats sub --count=1 --timeout=5s "volund.pool.>"

# Check queue group membership for task dispatch
kubectl exec -it nats-0 -n volund -- nats server report accounts

# Inspect a specific conversation stream
kubectl exec -it nats-0 -n volund -- nats sub --count=5 --timeout=10s "volund.conv.<convId>.stream"

# Verify agent is subscribed to its task subject
kubectl exec -it nats-0 -n volund -- nats server report connections --subscriptions | grep "volund.agent.<instanceId>"

# Check for slow consumers
kubectl exec -it nats-0 -n volund -- nats server report connections | grep -i "slow"
```

#### Remediation

```bash
# If slow consumers detected, check agent pod resource usage
kubectl top pods -n <tenant-ns> -l volund.io/component=agent

# Increase agent pod resources if needed
kubectl patch agentwarmpool <pool-name> -n <tenant-ns> \
  --type merge -p '{"spec":{"template":{"resources":{"requests":{"cpu":"500m","memory":"512Mi"}}}}}'

# If NATS JetStream is involved, check stream health
kubectl exec -it nats-0 -n volund -- nats stream ls
kubectl exec -it nats-0 -n volund -- nats stream info <stream-name>

# Purge a stuck stream (DATA LOSS -- use only as last resort)
kubectl exec -it nats-0 -n volund -- nats stream purge <stream-name> --force
```

---

## 5. Database Issues

### 5a. Connection Pool Exhaustion

#### Symptoms

- Application logs: `too many connections` or `connection pool exhausted`.
- New claims and API requests fail with database errors.
- Prometheus: `volund_db_pool_active` approaching `volund_db_pool_max`.

#### Investigation

```bash
# Check current connections from PostgreSQL
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# Check max connections setting
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SHOW max_connections;"

# Check which clients are holding connections
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT client_addr, usename, count(*) FROM pg_stat_activity GROUP BY client_addr, usename ORDER BY count DESC;"

# Check for long-running transactions
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT pid, now() - xact_start AS duration, query FROM pg_stat_activity WHERE state = 'active' AND xact_start < now() - interval '30 seconds' ORDER BY duration DESC;"

# If using PgBouncer
kubectl logs deploy/pgbouncer -n volund --tail=50
```

#### Remediation

```bash
# Kill long-running idle connections
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND state_change < now() - interval '10 minutes';"

# Kill a specific long-running query
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT pg_terminate_backend(<pid>);"

# Increase max connections (requires restart)
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "ALTER SYSTEM SET max_connections = 200;"
kubectl delete pod postgres-0 -n volund
```

### 5b. Slow Queries

#### Symptoms

- Elevated API latency across multiple endpoints.
- PostgreSQL CPU high.
- Logs showing query durations in seconds.

#### Investigation

```bash
# Check slow queries (if pg_stat_statements is enabled)
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT query, calls, mean_exec_time, total_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"

# Check for missing indexes
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT relname, seq_scan, seq_tup_read, idx_scan, idx_tup_fetch FROM pg_stat_user_tables WHERE seq_scan > 1000 ORDER BY seq_scan DESC LIMIT 10;"

# Check table sizes (pgvector tables can grow large)
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS total_size FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 10;"

# Check for lock contention
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT blocked.pid AS blocked_pid, blocked.query AS blocked_query, blocking.pid AS blocking_pid, blocking.query AS blocking_query FROM pg_stat_activity blocked JOIN pg_locks bl ON bl.pid = blocked.pid JOIN pg_locks lock_b ON lock_b.locktype = bl.locktype AND lock_b.relation = bl.relation AND lock_b.pid != bl.pid JOIN pg_stat_activity blocking ON blocking.pid = lock_b.pid WHERE NOT bl.granted;"
```

#### Remediation

```bash
# Run ANALYZE to update query planner statistics
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "ANALYZE;"

# Rebuild pgvector indexes if vector search is slow
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "REINDEX INDEX CONCURRENTLY <index_name>;"

# Vacuum large tables
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "VACUUM (VERBOSE, ANALYZE) <table_name>;"
```

### 5c. Migration Failures

#### Symptoms

- Deployment fails during migration step.
- Schema version mismatch errors in application logs.
- `dirty` state in migration tracking table.

#### Investigation

```bash
# Check migration status table
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT * FROM schema_migrations ORDER BY version DESC LIMIT 10;"

# Check for dirty flag
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT * FROM schema_migrations WHERE dirty = true;"

# Check migration job logs
kubectl logs job/volund-migrate -n volund
```

#### Remediation

```bash
# Fix dirty migration state (after manually verifying the schema)
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "UPDATE schema_migrations SET dirty = false WHERE version = <version>;"

# Re-run migrations
kubectl delete job volund-migrate -n volund
kubectl apply -f deploy/jobs/migrate.yaml

# If a migration partially applied, manually roll back then retry
kubectl exec -it postgres-0 -n volund -- psql -U volund -f /path/to/rollback.sql
```

---

## 6. Redis Failures

### 6a. Routing Table Impact

#### Symptoms

- Claims fail because the gateway cannot look up or update pod routing.
- Gateway logs: `redis: connection refused` or `redis: timeout`.
- Existing conversations may continue (agent pods are still running) but new claims fail.

#### Investigation

```bash
# Check Redis pod status
kubectl get pods -n volund -l app=redis

# Check Redis connectivity from gateway
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc PING

# Check Redis memory usage
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc INFO memory

# Check routing table key count
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc DBSIZE

# Check for Redis slow log
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc SLOWLOG GET 10

# Check Redis connection count
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc INFO clients
```

#### Remediation

```bash
# If Redis is OOM, flush expired keys
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc --scan --pattern "volund:session:*" | head -20

# Restart Redis if unresponsive (will lose routing table -- gateway must rebuild)
kubectl delete pod -n volund -l app=redis

# After Redis restart, force gateway to rebuild routing table
kubectl rollout restart deploy/volund-gateway -n volund

# Increase Redis memory limit
kubectl patch deploy redis -n volund --type merge \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"redis","resources":{"limits":{"memory":"2Gi"}}}]}}}}'
```

### 6b. Session Cache Loss

#### Symptoms

- Users report losing conversation context after a Redis restart.
- Gateway returns session-not-found errors.
- Active conversations may fail to route to the correct agent pod.

#### Investigation

```bash
# Check if session keys exist
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc KEYS "volund:session:<convId>"

# Check session TTL
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc TTL "volund:session:<convId>"
```

#### Remediation

Session cache is ephemeral. After Redis recovery:

1. Active conversations will need to be re-established by the client.
2. The gateway will re-populate routing entries as new claims occur.
3. If Redis persistence (RDB/AOF) is enabled, verify it loaded correctly:

```bash
kubectl exec -it redis-0 -n volund -- redis-cli -h redis.volund.svc INFO persistence
kubectl logs -n volund -l app=redis | grep -i "rdb\|aof\|loading"
```

---

## 7. Agent Pod Crashes and Stuck Pods

### 7a. CrashLoopBackOff

#### Symptoms

- Agent pods in `CrashLoopBackOff` state.
- Warm pool cannot maintain minReady count.
- `WarmPoolExhaustion` alert firing.

#### Investigation

```bash
# Find crashing pods
kubectl get pods -n <tenant-ns> -l volund.io/component=agent --field-selector=status.phase!=Running

# Check crash logs
kubectl logs -n <tenant-ns> <pod-name> --previous

# Check pod events
kubectl describe pod -n <tenant-ns> <pod-name> | tail -30

# Check for OOMKilled
kubectl get pods -n <tenant-ns> -l volund.io/component=agent -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].lastState.terminated.reason}{"\n"}{end}'

# Check resource usage of running agent pods
kubectl top pods -n <tenant-ns> -l volund.io/component=agent
```

#### Remediation

```bash
# If OOMKilled, increase memory limits
kubectl patch agentwarmpool <pool-name> -n <tenant-ns> \
  --type merge -p '{"spec":{"template":{"resources":{"limits":{"memory":"1Gi"}}}}}'

# If crash is due to bad agent config, check the AgentProfile
kubectl get agentprofile -n <tenant-ns> -o yaml

# Delete and recreate stuck pods
kubectl delete pod -n <tenant-ns> <pod-name> --grace-period=0 --force

# If widespread, roll the entire warm pool
kubectl delete pods -n <tenant-ns> -l volund.io/pool=<pool-name>
```

### 7b. Stuck / Zombie Pods

#### Symptoms

- Pods in `Running` state but not processing tasks.
- Heartbeats missing on `volund.events.io.volund.agent.heartbeat`.
- Conversations assigned to these pods hang.
- Active instance count stays high but throughput drops.

#### Investigation

```bash
# Check pods that have been running a long time without activity
kubectl get pods -n <tenant-ns> -l volund.io/component=agent --sort-by=.status.startTime

# Check if agent responds to health probes
kubectl exec -it -n <tenant-ns> <pod-name> -- curl -s localhost:8080/healthz

# Check agent logs for signs of deadlock or hang
kubectl logs -n <tenant-ns> <pod-name> --tail=100

# Check NATS subscription status for a specific agent
kubectl exec -it nats-0 -n volund -- nats server report connections | grep <pod-ip>

# Check if pod is still receiving heartbeat traffic
kubectl exec -it nats-0 -n volund -- nats sub --count=3 --timeout=30s "volund.events.io.volund.agent.heartbeat" | grep <instanceId>
```

#### Remediation

```bash
# Delete the stuck pod (operator will replace it)
kubectl delete pod -n <tenant-ns> <pod-name>

# If the AgentInstance CRD is stuck, clean it up
kubectl get agentinstance -n <tenant-ns> | grep <instance-name>
kubectl delete agentinstance -n <tenant-ns> <instance-name>

# Force-release claimed pods that are no longer responsive
# (gateway should detect this via heartbeat timeout, but if not)
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc DEL "volund:route:<instanceId>"
```

---

## 8. Quota and Billing Issues

### Symptoms

- Tenants receiving 429 (rate limit) or 402 (payment required) responses.
- Claims rejected with `quota exceeded` errors.
- Unexpected warm pool scale-downs.

### Investigation

```bash
# Check tenant quota status in the database
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT tenant_id, plan, max_instances, current_usage, billing_status FROM tenants WHERE tenant_id = '<tenant-id>';"

# Check current active instances for the tenant
kubectl get agentinstance -n <tenant-ns> --no-headers | wc -l

# Check gateway logs for quota rejections
kubectl logs deploy/volund-gateway -n volund --since=10m | grep -i "quota\|rate.limit\|billing" | grep "<tenant-id>"

# Check warm pool spec vs quota
kubectl get agentwarmpool -n <tenant-ns> -o jsonpath='{.items[0].spec.maxSize}'
```

### Remediation

```bash
# Temporarily increase tenant quota (emergency)
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "UPDATE tenants SET max_instances = 100 WHERE tenant_id = '<tenant-id>';"

# Reset rate limit counters in Redis
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc DEL "volund:ratelimit:<tenant-id>"

# Verify billing status
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT billing_status, last_payment_at, subscription_ends_at FROM tenants WHERE tenant_id = '<tenant-id>';"

# If billing webhook missed, manually update status
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "UPDATE tenants SET billing_status = 'active' WHERE tenant_id = '<tenant-id>';"
```

After any manual quota or billing override, create a ticket for the billing team to reconcile.

---

## 9. Security Incidents

### 9a. Sandbox Escape

#### Symptoms

- Agent pod accessing resources outside its namespace.
- Unexpected network connections from agent pods.
- Alerts from network policy violations or PodSecurityPolicy/PodSecurity admission.

#### Investigation

```bash
# Check network policy enforcement
kubectl get networkpolicy -n <tenant-ns>

# Check pod security context
kubectl get pod <pod-name> -n <tenant-ns> -o jsonpath='{.spec.containers[0].securityContext}'

# Check for suspicious processes in the pod
kubectl exec -it -n <tenant-ns> <pod-name> -- ps aux

# Check for unexpected outbound connections
kubectl exec -it -n <tenant-ns> <pod-name> -- ss -tnp

# Audit logs for the namespace
kubectl logs -n kube-system -l component=kube-apiserver --since=1h | grep "<tenant-ns>"

# Check if the pod has mounted any unexpected volumes
kubectl get pod <pod-name> -n <tenant-ns> -o jsonpath='{.spec.volumes[*].name}'
```

#### Remediation

**Immediate containment:**

```bash
# Isolate the pod by applying a deny-all network policy
cat <<'EOF' | kubectl apply -n <tenant-ns> -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: emergency-isolate
spec:
  podSelector:
    matchLabels:
      volund.io/instance: <instance-id>
  policyTypes:
  - Ingress
  - Egress
EOF

# Kill the compromised pod
kubectl delete pod -n <tenant-ns> <pod-name> --grace-period=0 --force

# Delete the AgentInstance to prevent re-scheduling
kubectl delete agentinstance -n <tenant-ns> <instance-name>
```

**Post-incident:**

1. Preserve pod logs before deletion: `kubectl logs -n <tenant-ns> <pod-name> > /tmp/incident-<pod-name>.log`
2. Review the AgentProfile and Skill CRDs for the tenant to identify what code ran.
3. Audit the tenant's API usage patterns.
4. Rotate any credentials that were accessible to the pod.
5. File an incident report.

### 9b. Credential Leak

#### Symptoms

- Credentials appearing in logs or metrics.
- Unauthorized API calls using valid tenant credentials.
- Alerts from secret scanning tools.

#### Investigation

```bash
# Check if secrets are properly mounted (not in env vars or args)
kubectl get pod <pod-name> -n <tenant-ns> -o yaml | grep -A 3 "env:"

# Check for secrets in recent logs
kubectl logs deploy/volund-gateway -n volund --since=1h | grep -iE "api.key|token|secret|password|bearer"

# Check Kubernetes secrets access audit
kubectl logs -n kube-system -l component=kube-apiserver --since=1h | grep "secrets" | grep "<tenant-ns>"
```

#### Remediation

```bash
# Rotate the compromised credentials immediately
# 1. Generate new credentials
# 2. Update the Kubernetes secret
kubectl create secret generic <secret-name> -n <tenant-ns> \
  --from-literal=api-key=<new-key> \
  --dry-run=client -o yaml | kubectl apply -f -

# 3. Restart affected deployments to pick up new secrets
kubectl rollout restart deploy/volund-gateway -n volund

# 4. Revoke the old credentials at the provider level

# 5. Audit all API calls made with the compromised credentials
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT * FROM audit_log WHERE tenant_id = '<tenant-id>' AND created_at > now() - interval '24 hours' ORDER BY created_at DESC;"
```

---

## 10. Disaster Recovery

### Backup Procedures

#### PostgreSQL Backup

```bash
# Manual pg_dump
kubectl exec -it postgres-0 -n volund -- pg_dump -U volund -Fc volund > /tmp/volund-backup-$(date +%Y%m%d%H%M%S).dump

# Verify backup integrity
pg_restore --list /tmp/volund-backup-*.dump | head -20

# Automated backup (check CronJob)
kubectl get cronjob -n volund -l app=postgres-backup
kubectl get jobs -n volund -l app=postgres-backup --sort-by=.status.startTime | tail -5
```

#### NATS JetStream Backup

```bash
# Export stream data
kubectl exec -it nats-0 -n volund -- nats stream ls
kubectl exec -it nats-0 -n volund -- nats stream backup <stream-name> /tmp/nats-backup/
```

#### Redis Backup

```bash
# Trigger RDB snapshot
kubectl exec -it redis-0 -n volund -- redis-cli -h redis.volund.svc BGSAVE

# Check snapshot status
kubectl exec -it redis-0 -n volund -- redis-cli -h redis.volund.svc LASTSAVE

# Copy RDB file
kubectl cp volund/redis-0:/data/dump.rdb /tmp/redis-dump-$(date +%Y%m%d).rdb
```

#### Kubernetes Resources Backup

```bash
# Export all VOLUND CRDs
for crd in agentwarmpool agentinstance agentprofile skill; do
  kubectl get $crd -A -o yaml > /tmp/volund-${crd}-backup-$(date +%Y%m%d).yaml
done

# Export namespace configs
kubectl get all,configmap,secret,networkpolicy -n volund -o yaml > /tmp/volund-ns-backup-$(date +%Y%m%d).yaml
```

### Restore Procedures

#### PostgreSQL Restore

```bash
# Restore from pg_dump
kubectl exec -i postgres-0 -n volund -- pg_restore -U volund -d volund --clean --if-exists < /tmp/volund-backup-<timestamp>.dump

# Verify restore
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT count(*) FROM tenants;"
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT version FROM schema_migrations ORDER BY version DESC LIMIT 1;"
```

#### NATS JetStream Restore

```bash
# Restore stream
kubectl exec -it nats-0 -n volund -- nats stream restore <stream-name> /tmp/nats-backup/
```

#### Redis Restore

```bash
# Stop Redis, replace dump, restart
kubectl exec -it redis-0 -n volund -- redis-cli -h redis.volund.svc SHUTDOWN NOSAVE
kubectl cp /tmp/redis-dump-<date>.rdb volund/redis-0:/data/dump.rdb
kubectl delete pod redis-0 -n volund
# Pod will restart and load the RDB file
```

#### Full Platform Recovery Sequence

If performing a complete platform recovery, follow this order:

1. **PostgreSQL** -- Restore database first (source of truth for tenant data, profiles, configuration).
2. **Redis** -- Restore or allow gateway to rebuild routing table. Session cache can be rebuilt.
3. **NATS** -- Restore JetStream streams if applicable. Core NATS subjects are ephemeral and do not require restore.
4. **Kubernetes CRDs** -- Apply AgentProfile and Skill CRDs. These define agent behavior.
5. **AgentWarmPool CRDs** -- Apply warm pool definitions. Operator will begin provisioning pods.
6. **Gateway + Operator** -- Restart gateway and operator deployments.

```bash
# Verify full platform health after restore
kubectl get pods -n volund
kubectl get agentwarmpool -A
kubectl exec -it deploy/volund-gateway -n volund -- curl -s localhost:8080/healthz
kubectl exec -it nats-0 -n volund -- nats server report connections
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT 1;"
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc PING
```

---

## Appendix: Common Commands Reference

```bash
# Quick status of all VOLUND components
kubectl get pods -n volund -o wide
kubectl get agentwarmpool -A
kubectl get agentinstance -A --no-headers | wc -l

# Follow gateway logs
kubectl logs -f deploy/volund-gateway -n volund

# Follow operator logs
kubectl logs -f deploy/volund-operator -n volund

# NATS quick diagnostics
kubectl exec -it nats-0 -n volund -- nats server report connections
kubectl exec -it nats-0 -n volund -- nats server report jetstream

# Redis quick diagnostics
kubectl exec -it deploy/volund-gateway -n volund -- redis-cli -h redis.volund.svc INFO

# PostgreSQL quick diagnostics
kubectl exec -it postgres-0 -n volund -- psql -U volund -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"
```
