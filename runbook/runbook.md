# 📌 Order Management Service - Operations Runbook

## Service Overview

* **Service Name:** order-management-service
* **Owner Team:** Orders Platform Team
* **Criticality:** Critical
* **Environments:** prod, stage, dev
* **Primary On-call:** orders-oncall@ecommerce.com
* **Slack Channel:** #orders-alerts
* **Repository:** https://github.com/ecommerce/order-service
* **Monitoring Dashboard:** https://grafana.ecommerce.com/d/order-service

---

## 1. Payment Processing Failures

**Error Code(s):**
`PAYMENT_FAILED`, `PAYMENT_GATEWAY_TIMEOUT`, `PAYMENT_DECLINED`

**Severity:** Critical

---

### Symptoms

* Orders stuck in `pending_payment` status
* High rate of payment failure errors in logs
* Customer complaints about failed checkouts
* Payment gateway timeout errors
* Increased cart abandonment rate

---

### Likely Causes

* Payment gateway API unavailable or degraded
* Network connectivity issues to payment provider
* Invalid payment credentials or API keys
* Payment gateway rate limiting
* Insufficient funds or declined cards (customer issue)

---

### How to Confirm

1. Check payment gateway status page
2. Verify payment API credentials are valid
3. Test payment processing manually
4. Review payment gateway response codes
5. Check network connectivity to payment provider

```bash
# Check payment service health
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/health/payment-gateway

# View payment failure logs
kubectl logs -l app=order-service | grep PAYMENT_FAILED

# Test payment gateway connectivity
kubectl exec -it deployment/order-service -- \
  curl -I https://api.stripe.com/v1/charges

# Check payment metrics
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/metrics | grep payment_success_rate
```

---

### Recommended Fix (Safe Actions)

1. **Verify gateway status**: Check payment provider status page
2. **Test connectivity**: Ensure network path to gateway is healthy
3. **Validate credentials**: Confirm API keys are correct and not expired
4. **Retry failed payments**: Trigger retry for stuck orders
5. **Fallback gateway**: Switch to backup payment processor if available

```bash
# Retry failed payments
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/retry-payments

# Check payment gateway configuration
kubectl get secret payment-gateway-credentials -o yaml

# Restart order service if needed
kubectl rollout restart deployment/order-service

# Monitor payment success rate
watch -n 5 'kubectl exec -it deployment/order-service -- \
  curl -s localhost:8080/metrics | grep payment_success_rate'
```

---

### Rollback / Recovery

* Failed payments are automatically retried up to 3 times
* Orders remain in pending state - no data loss
* Customers can retry payment from order page
* Contact payment provider support if widespread issue

---

### Notes

* Payment failures due to declined cards are normal - not a service issue
* Payment gateway downtime requires immediate escalation
* Always verify payment provider status before investigating service
* Keep backup payment processor configured for failover

---

## 2. Inventory Reservation Failures

**Error Code(s):**
`INVENTORY_UNAVAILABLE`, `RESERVATION_TIMEOUT`, `OVERSOLD_PRODUCT`

**Severity:** High

---

### Symptoms

* Orders failing with "out of stock" errors
* Products showing available but order creation fails
* Inventory count mismatches between services
* Race condition errors in logs
* Customer complaints about unavailable products

---

### Likely Causes

* Race condition in concurrent inventory updates
* Catalog service inventory sync lag
* Distributed lock failures
* Inventory reservation timeout
* Manual inventory adjustments not synced

---

### How to Confirm

1. Compare inventory in catalog vs order service
2. Check for distributed lock errors
3. Review inventory reservation metrics
4. Verify Kafka event lag for inventory updates
5. Check for concurrent order spikes

```bash
# Check inventory sync status
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/admin/inventory/sync-status

# View inventory reservation errors
kubectl logs -l app=order-service | grep INVENTORY_UNAVAILABLE

# Check Kafka consumer lag
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group order-inventory-consumer --describe

# Verify distributed locks
kubectl exec -it redis-0 -- redis-cli keys "lock:inventory:*"
```

---

### Recommended Fix (Safe Actions)

1. **Check sync status**: Verify inventory sync with catalog service
2. **Clear stale locks**: Remove expired distributed locks
3. **Trigger sync**: Force inventory resync from catalog
4. **Increase timeout**: Temporarily increase reservation timeout
5. **Monitor**: Watch inventory reservation success rate

```bash
# Trigger inventory resync
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/inventory/resync

# Clear stale locks (older than 5 minutes)
kubectl exec -it redis-0 -- redis-cli --scan --pattern "lock:inventory:*" | \
  xargs -L 1 redis-cli DEL

# Check reservation success rate
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/metrics | grep inventory_reservation_success
```

---

### Rollback / Recovery

* Failed reservations are automatically released after timeout
* Inventory sync runs every 5 minutes automatically
* No manual cleanup needed for failed reservations
* Monitor for overselling - may need to cancel orders

---

### Notes

* Inventory reservation timeout is 30 seconds
* Distributed locks prevent race conditions
* Always sync with catalog service as source of truth
* High concurrency can cause temporary reservation failures

---

## 3. Order Stuck in Processing State

**Error Code(s):**
`ORDER_PROCESSING_TIMEOUT`, `WORKFLOW_STUCK`

**Severity:** Medium

---

### Symptoms

* Orders not progressing from "processing" status
* Fulfillment workflow not triggering
* Orders older than 2 hours still in processing
* Workflow state machine errors in logs
* Warehouse not receiving order notifications

---

### Likely Causes

* Kafka event publishing failure
* Workflow engine service unavailable
* State machine transition error
* Webhook delivery failure to warehouse
* Database transaction timeout

---

### How to Confirm

1. Check order status and last updated timestamp
2. Verify Kafka events were published
3. Review workflow engine logs
4. Check webhook delivery status
5. Verify database transaction completed

```bash
# Find stuck orders
kubectl exec -it deployment/order-service -- \
  curl "localhost:8080/admin/orders/stuck?status=processing&olderThan=2h"

# Check Kafka events for order
kubectl exec -it kafka-0 -- kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --from-beginning | grep "ord_xyz789abc"

# Check workflow engine status
kubectl get pods -l app=workflow-engine

# View webhook delivery logs
kubectl logs -l app=order-service | grep "webhook:fulfillment"
```

---

### Recommended Fix (Safe Actions)

1. **Identify stuck orders**: Query for orders in processing >2 hours
2. **Retry workflow**: Trigger workflow retry for stuck orders
3. **Republish events**: Manually publish missing Kafka events
4. **Check dependencies**: Verify workflow engine and warehouse systems
5. **Manual intervention**: Update status manually if workflow failed

```bash
# Retry workflow for stuck order
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/orders/ord_xyz789abc/retry-workflow

# Republish order events
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/orders/ord_xyz789abc/republish-events

# Manually update order status (last resort)
kubectl exec -it deployment/order-service -- \
  curl -X PATCH localhost:8080/admin/orders/ord_xyz789abc/status \
  -d '{"status": "shipped", "reason": "manual_intervention"}'
```

---

### Rollback / Recovery

* Workflow retries are idempotent - safe to retry multiple times
* Events can be republished without side effects
* Manual status updates should be documented in order notes
* Notify warehouse team of manual interventions

---

### Notes

* Normal processing time is 5-15 minutes
* Stuck orders >2 hours require investigation
* Always check Kafka events before manual intervention
* Document manual status changes for audit trail

---

## 4. Refund Processing Delays

**Error Code(s):**
`REFUND_FAILED`, `REFUND_TIMEOUT`, `PAYMENT_GATEWAY_ERROR`

**Severity:** Medium

---

### Symptoms

* Refunds stuck in "pending" status
* Payment gateway refund errors
* Customer complaints about delayed refunds
* Refund processing taking >24 hours
* High refund failure rate

---

### Likely Causes

* Payment gateway API issues
* Original payment method no longer valid
* Refund amount exceeds original charge
* Payment gateway rate limiting
* Network connectivity issues

---

### How to Confirm

1. Check refund status in payment gateway dashboard
2. Review refund error logs
3. Verify original payment still exists
4. Check refund amount vs original charge
5. Test payment gateway connectivity

```bash
# Check pending refunds
kubectl exec -it deployment/order-service -- \
  curl "localhost:8080/admin/refunds?status=pending&olderThan=24h"

# View refund error logs
kubectl logs -l app=order-service | grep REFUND_FAILED

# Check payment gateway refund status
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/admin/refunds/ref_abc123/gateway-status

# Test refund API
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/test-refund
```

---

### Recommended Fix (Safe Actions)

1. **Check gateway status**: Verify payment provider is operational
2. **Retry refund**: Trigger retry for failed refunds
3. **Validate payment**: Ensure original payment exists
4. **Manual refund**: Process refund manually via gateway dashboard
5. **Alternative method**: Issue store credit if refund fails

```bash
# Retry failed refund
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/refunds/ref_abc123/retry

# Check original payment
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/admin/payments/pay_xyz789/details

# Issue store credit (alternative)
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/refunds/ref_abc123/store-credit
```

---

### Rollback / Recovery

* Failed refunds can be retried safely
* Store credit is alternative to payment refund
* Manual refunds via gateway require order update
* Always notify customer of refund status

---

### Notes

* Refunds typically process in 5-10 business days
* Payment gateway may have daily refund limits
* Store credit is instant alternative to refund
* Document manual refunds in order notes

---

## 5. Shipping Label Generation Failures

**Error Code(s):**
`LABEL_GENERATION_FAILED`, `CARRIER_API_ERROR`, `INVALID_ADDRESS`

**Severity:** Medium

---

### Symptoms

* Orders ready to ship but no label generated
* Carrier API errors in logs
* Invalid address validation errors
* Label generation timeout
* Warehouse unable to ship orders

---

### Likely Causes

* Carrier API unavailable or rate limited
* Invalid or incomplete shipping address
* Carrier account credentials expired
* Package dimensions/weight missing
* Unsupported shipping destination

---

### How to Confirm

1. Check carrier API status
2. Validate shipping address format
3. Review carrier API credentials
4. Verify package dimensions provided
5. Check carrier service availability for destination

```bash
# Check label generation failures
kubectl logs -l app=order-service | grep LABEL_GENERATION_FAILED

# Test carrier API connectivity
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/admin/carriers/test-connection

# Validate address
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/validate-address \
  -d '{"street": "123 Main St", "city": "San Francisco", "state": "CA", "zip": "94105"}'

# Check carrier credentials
kubectl get secret shipping-carrier-credentials -o yaml
```

---

### Recommended Fix (Safe Actions)

1. **Verify carrier status**: Check carrier API availability
2. **Validate address**: Ensure address is complete and valid
3. **Retry generation**: Trigger label regeneration
4. **Update credentials**: Refresh carrier API credentials if expired
5. **Manual label**: Generate label manually via carrier portal

```bash
# Retry label generation
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/orders/ord_xyz789abc/regenerate-label

# Update carrier credentials
kubectl create secret generic shipping-carrier-credentials \
  --from-literal=api-key=$NEW_API_KEY --dry-run=client -o yaml | kubectl apply -f -

# Restart to apply new credentials
kubectl rollout restart deployment/order-service
```

---

### Rollback / Recovery

* Label generation can be retried without side effects
* Manual labels can be uploaded to order
* Address corrections may require customer contact
* Carrier API issues require escalation to carrier support

---

### Notes

* Label generation timeout is 30 seconds
* Address validation prevents most failures
* Keep backup carrier configured for redundancy
* Some destinations may not support all carriers

---

## 6. Database Connection Pool Exhaustion

**Error Code(s):**
`DB_POOL_EXHAUSTED`, `CONNECTION_TIMEOUT`, `TOO_MANY_CONNECTIONS`

**Severity:** Critical

---

### Symptoms

* API requests timing out
* Database connection errors in logs
* 500 errors for all endpoints
* Connection pool at 100% utilization
* Slow query performance

---

### Likely Causes

* Connection leak in application code
* Long-running queries holding connections
* Sudden traffic spike
* Database maintenance or failover
* Connection pool size too small

---

### How to Confirm

1. Check connection pool metrics
2. Review active database connections
3. Identify long-running queries
4. Check traffic patterns
5. Verify database health

```bash
# Check connection pool usage
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/metrics | grep db_pool

# View active connections
kubectl exec -it postgres-0 -- psql -U orders -c \
  "SELECT count(*) FROM pg_stat_activity WHERE state = 'active';"

# Find long-running queries
kubectl exec -it postgres-0 -- psql -U orders -c \
  "SELECT pid, now() - query_start as duration, query FROM pg_stat_activity WHERE state = 'active' ORDER BY duration DESC;"

# Check database health
kubectl get pods -l app=postgres
```

---

### Recommended Fix (Safe Actions)

1. **Restart service**: Release stale connections
2. **Kill long queries**: Terminate blocking queries
3. **Increase pool size**: Temporarily increase connection limit
4. **Scale service**: Add more replicas to distribute load
5. **Monitor**: Watch connection usage after changes

```bash
# Restart order service
kubectl rollout restart deployment/order-service

# Kill long-running query (if identified)
kubectl exec -it postgres-0 -- psql -U orders -c \
  "SELECT pg_terminate_backend(12345);"

# Increase connection pool (temporary)
kubectl set env deployment/order-service DB_POOL_SIZE=50

# Scale up replicas
kubectl scale deployment/order-service --replicas=6
```

---

### Rollback / Recovery

* Revert pool size after identifying root cause
* Scale down replicas after traffic normalizes
* Fix connection leaks in code before next deployment
* Monitor connection usage closely

---

### Notes

* Default pool size is 20 connections per pod
* Connection leaks must be fixed in code
* Restart is safe - service has multiple replicas
* Database failover can cause temporary connection issues

---

## 7. Kafka Event Publishing Lag

**Error Code(s):**
`KAFKA_PUBLISH_FAILED`, `EVENT_LAG_HIGH`, `CONSUMER_LAG`

**Severity:** High

---

### Symptoms

* Downstream services showing stale order data
* Kafka consumer lag increasing
* Event publishing errors in logs
* Order status updates not propagating
* Notification delays

---

### Likely Causes

* Kafka broker unavailable
* Network issues to Kafka cluster
* Event serialization errors
* Kafka topic partition issues
* Producer buffer overflow

---

### How to Confirm

1. Check Kafka cluster health
2. Measure consumer group lag
3. Review event publishing metrics
4. Test Kafka connectivity
5. Check topic partition status

```bash
# Check Kafka cluster status
kubectl get pods -l app=kafka

# Check consumer lag
kafka-consumer-groups --bootstrap-server kafka:9092 --describe --all-groups

# View event publishing errors
kubectl logs -l app=order-service | grep KAFKA_PUBLISH_FAILED

# Test Kafka connectivity
kubectl exec -it deployment/order-service -- \
  nc -zv kafka-0.kafka-headless 9092

# Check topic status
kafka-topics --bootstrap-server kafka:9092 --describe --topic order-events
```

---

### Recommended Fix (Safe Actions)

1. **Verify Kafka health**: Ensure brokers are healthy
2. **Restart consumers**: Reset consumer groups if stuck
3. **Republish events**: Replay failed events from dead letter queue
4. **Increase resources**: Scale Kafka if under-provisioned
5. **Monitor lag**: Watch consumer lag metrics

```bash
# Restart order service to reset Kafka connections
kubectl rollout restart deployment/order-service

# Reset consumer group (if stuck)
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group notification-consumer --reset-offsets --to-latest --execute --all-topics

# Republish failed events
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/events/republish-failed

# Check Kafka broker resources
kubectl top pods -l app=kafka
```

---

### Rollback / Recovery

* Failed events stored in dead letter queue for 7 days
* Consumer lag will catch up automatically after fix
* Event replay is idempotent - safe to retry
* Monitor downstream services after recovery

---

### Notes

* Normal consumer lag is <100 messages
* Event publishing is asynchronous - service continues if Kafka down
* Dead letter queue prevents event loss
* Kafka maintenance should be scheduled during low traffic

---

## 8. High Memory Usage / OOM Kills

**Error Code(s):**
`OUT_OF_MEMORY`, `CONTAINER_OOM_KILLED`

**Severity:** High

---

### Symptoms

* Pods being killed and restarted
* OOMKilled status in pod events
* Memory usage at 95%+ of limit
* Service unavailability during restarts
* Increased pod restart count

---

### Likely Causes

* Memory leak in application code
* Large order export job
* Inefficient data loading
* Insufficient memory limits
* Memory-intensive query

---

### How to Confirm

1. Check pod events for OOMKilled
2. Review memory usage metrics
3. Identify memory-intensive operations
4. Check heap dump if available
5. Review recent code changes

```bash
# Check pod events
kubectl describe pod <pod-name> | grep -A 10 Events

# Check memory usage
kubectl top pods -l app=order-service

# Check pod restart count
kubectl get pods -l app=order-service -o jsonpath='{.items[*].status.containerStatuses[*].restartCount}'

# Get heap dump (if JVM)
kubectl exec -it <pod-name> -- jmap -dump:live,format=b,file=/tmp/heap.hprof 1
```

---

### Recommended Fix (Safe Actions)

1. **Increase memory**: Temporarily increase memory limits
2. **Scale up**: Add more replicas to distribute load
3. **Investigate**: Review heap dump for memory leaks
4. **Optimize**: Fix memory-intensive operations
5. **Monitor**: Watch memory usage after changes

```bash
# Increase memory limit
kubectl set resources deployment/order-service --limits=memory=4Gi

# Scale up replicas
kubectl scale deployment/order-service --replicas=6

# Restart pods
kubectl rollout restart deployment/order-service

# Monitor memory usage
watch -n 5 'kubectl top pods -l app=order-service'
```

---

### Rollback / Recovery

* Revert memory increase after fixing root cause
* Scale down replicas after load normalizes
* Deploy memory leak fix in next release
* Monitor memory trends

---

### Notes

* Memory increase is temporary solution
* Common causes: large exports, inefficient queries
* Use pagination for large data operations
* Implement memory limits on batch jobs

---

## 9. Return Processing Failures

**Error Code(s):**
`RETURN_APPROVAL_FAILED`, `RETURN_WINDOW_EXPIRED`, `INVALID_RETURN`

**Severity:** Low

---

### Symptoms

* Return requests stuck in pending status
* Return approval workflow not triggering
* Customer complaints about return delays
* Return label generation failures
* Refund not processing after return received

---

### Likely Causes

* Return window validation incorrect
* ML fraud detection false positives
* Return label generation API failure
* Workflow engine issues
* Database constraint violations

---

### How to Confirm

1. Check return request status
2. Review fraud detection scores
3. Verify return window calculation
4. Check workflow engine logs
5. Validate return eligibility

```bash
# Check pending returns
kubectl exec -it deployment/order-service -- \
  curl "localhost:8080/admin/returns?status=pending&olderThan=24h"

# View return approval logs
kubectl logs -l app=order-service | grep "return:approval"

# Check fraud detection score
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/admin/returns/ret_abc123/fraud-score

# Verify return window
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/admin/orders/ord_xyz789abc/return-eligibility
```

---

### Recommended Fix (Safe Actions)

1. **Review fraud score**: Check if legitimate return flagged
2. **Manual approval**: Override fraud detection if false positive
3. **Retry workflow**: Trigger return workflow retry
4. **Extend window**: Override return window if justified
5. **Generate label**: Manually generate return label

```bash
# Manually approve return
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/returns/ret_abc123/approve \
  -d '{"reason": "manual_review_passed"}'

# Retry return workflow
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/returns/ret_abc123/retry-workflow

# Override return window
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/returns/ret_abc123/extend-window \
  -d '{"days": 7, "reason": "customer_service_exception"}'
```

---

### Rollback / Recovery

* Manual approvals are logged for audit
* Return window extensions documented in return notes
* Workflow retries are idempotent
* Notify customer service of manual interventions

---

### Notes

* Standard return window is 30 days
* Fraud detection has 2% false positive rate
* Manual approvals require manager authorization
* Document all exceptions in return notes

---

## 10. Order Export Job Failures

**Error Code(s):**
`EXPORT_FAILED`, `EXPORT_TIMEOUT`, `FILE_GENERATION_ERROR`

**Severity:** Low

---

### Symptoms

* Export jobs stuck in processing
* Export job timing out
* Incomplete CSV/JSON files
* Export download links not generated
* Memory errors during large exports

---

### Likely Causes

* Export query timeout
* File size exceeding limits
* Memory exhaustion during export
* S3 upload failure
* Database query performance issues

---

### How to Confirm

1. Check export job status
2. Review export job logs
3. Verify S3 bucket accessibility
4. Check database query performance
5. Validate export parameters

```bash
# Check export job status
kubectl exec -it deployment/order-service -- \
  curl localhost:8080/admin/exports/exp_abc123

# View export job logs
kubectl logs -l app=order-service | grep "export:exp_abc123"

# Test S3 bucket access
aws s3 ls s3://ecommerce-exports/ --profile prod

# Check export query performance
kubectl exec -it postgres-0 -- psql -U orders -c \
  "EXPLAIN ANALYZE SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';"
```

---

### Recommended Fix (Safe Actions)

1. **Split export**: Break large exports into smaller date ranges
2. **Retry job**: Trigger export retry
3. **Optimize query**: Add indexes for export queries
4. **Increase timeout**: Extend job timeout for large exports
5. **Manual export**: Run export manually with optimized parameters

```bash
# Retry export job
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/exports/exp_abc123/retry

# Cancel stuck job
kubectl exec -it deployment/order-service -- \
  curl -X DELETE localhost:8080/admin/exports/exp_abc123

# Create new export with smaller range
kubectl exec -it deployment/order-service -- \
  curl -X POST localhost:8080/admin/exports \
  -d '{"format": "csv", "startDate": "2024-01-01", "endDate": "2024-01-15"}'
```

---

### Rollback / Recovery

* Failed exports can be retried safely
* Partial exports are automatically cleaned up
* Split large exports into weekly batches
* Monitor export job duration

---

### Notes

* Maximum export size: 100,000 orders
* Export timeout: 30 minutes
* Use streaming for very large exports
* Schedule large exports during low traffic

---

## Escalation Path

* **Primary Escalation:** Orders Platform Team (#orders-team)
* **Secondary Escalation:** Platform Engineering (#platform-eng)
* **Payment Issues:** Payment Team (#payments)
* **Inventory Issues:** Catalog Team (#catalog-team)
* **When to Escalate:** Issue persists >20 minutes or affects >25% of orders

---

## Emergency Contacts

* **On-call Engineer:** orders-oncall@ecommerce.com (PagerDuty)
* **Team Lead:** alex.thompson@ecommerce.com
* **Engineering Manager:** maria.garcia@ecommerce.com

---

## Last Updated: 2024-03-10

---
