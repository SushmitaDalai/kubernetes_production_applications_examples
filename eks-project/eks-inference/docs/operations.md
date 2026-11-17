# Defending the design in an interview

## Start with the failure model

“For the inference API, I keep compute replaceable and make the model version part of the release. I spread replicas across Availability Zones, keep spare capacity for a rollout and a node drain, and measure whether the remaining capacity can meet the latency target after losing an AZ. A versioned model in S3 lets a replacement Pod rebuild its local state. I recover a regional failure by restoring compute from Terraform and Git and serving the same tested image and model from regional replicas.”

### Availability and release choice

| Option | How it works here | What to defend |
|---|---|---|
| Rolling | Replace one replica at a time, preserving ready capacity | Lower temporary cost. Both revisions may serve concurrently, so validate API and model compatibility. Revert the release configuration if quality regresses. |
| Blue/green | Read blue.yaml, green.yaml and preview-services.yaml as a separate inference layout. Keep the namespace, priority class, inference service account and security policy. Replace the rolling Deployment, Service and PDB with the two tracks and their Services. | Preview the green endpoint from an authorized frontend Pod. Validate schema, known predictions, cold start, error rate, latency and load. Change the stable inference Service selector from track blue to green in Git. Endpoint propagation is asynchronous. Keep blue warm until verification completes and revert the selector if needed. Budget roughly double steady inference capacity during overlap. |
| Canary | Use a traffic-routing layer that supports weighted backends, or explicitly route a small traffic cohort at the frontend | A plain ClusterIP selector cannot enforce a traffic weight. Compare prediction quality as well as infrastructure health before widening the cohort. |

Place replicas across nodes and AZs and ensure the inference node group can replenish capacity in each zone. Spread and a PDB are placement and maintenance controls. Actual AZ resilience depends on the throughput of the surviving replicas, subnet addresses, node allocatable resources, and load balancer behavior. For production, evaluate HPA on concurrency or queue depth with CPU as a secondary signal, and node scaling with Karpenter or Cluster Autoscaler. Bound scaling by startup time, downstream limits and the capacity required after failure.

## Observability

The sample ServiceMonitor discovers the rolling inference Service by its metadata label and the named Service port. Install Prometheus Operator and its CRDs first. Configure the Prometheus instance to select the ServiceMonitor and align its Pod labels with the inference NetworkPolicy. In blue/green, use dedicated track-specific scrape Services and labels if you need metrics for both revisions independently of serving traffic.

| Signal | Instrument and respond |
|---|---|
| Metrics | Track request rate, error ratio, p95 latency, queue delay, model load duration, saturation, ready replicas, restart rate, pending Pods and ALB target health. Use bounded label cardinality. |
| Logs | Emit JSON to stdout with request ID, model version and result status. A node agent can forward to CloudWatch Logs. Enable EKS control plane audit logs for API activity. Set retention and restrict access. |
| Traces | Propagate trace context from frontend to inference. An OpenTelemetry collector can batch and export spans. Sample selectively and omit inputs and outputs. |
| Alerts | Page on sustained error-budget burn or inability to serve, and route rollout stalls, failed scrapes and model-load failures to the owning team. Example target for discussion is 99.9% successful requests and p95 under 500 ms under agreed load. |

## Troubleshooting walk-through

| Scenario | First checks and decision |
|---|---|
| Fourth inference Pod Pending during rollout | Describe Pod and inspect scheduler events for affinity, taints, allocatable CPU/memory, zone skew and pod/IP limits. Inspect Deployment and ReplicaSet events for quota admission failures. Add capacity or revisit placement before relaxing availability settings. |
| Model init fails | Inspect model-fetch container logs and Pod events. Confirm role trust subject, projected credentials, policy, endpoint reachability and object digest. |
| ALB returns 503 | Check target health and frontend /ready, Ingress events and target group registration, security groups and subnet discovery. Separately test frontend-to-inference DNS, EndpointSlices, and policy. |
| Drain refuses eviction | Inspect PDB allowed disruptions and current readiness. Restore healthy replica capacity before resuming maintenance. |
| PVC remains Pending | Check consumer scheduling, CSI controller permissions and StorageClass. For a previously bound EBS volume, inspect PV node affinity and available nodes in its AZ. |
| Readiness is green but model quality drops | Compare model versions and prediction quality against validation data. Revert the model reference or blue/green selector, then validate actual user traffic. |

## Regional disaster recovery

Start with pilot light in a secondary region. Pre-create networking and a minimal cluster if recovery speed requires it. Replicate ECR images and versioned S3 artifacts and verify destination digests before certifying recovery. Replication is asynchronous. Keep an approved release manifest with artifacts confirmed present in both regions. Provision regional ACM, DNS, IAM trust for the secondary OIDC issuer, and required KMS keys or grants. Keep Terraform state recoverable and avoid concurrent writes from two active recovery pipelines.

1. Declare failover and freeze promotion. Identify the last approved complete release available in both regions.
2. Scale up secondary node capacity, reconcile add-ons and deploy the approved release. Restore any authoritative state from cross-region backups and verify data integrity. The optional cache is disposable.
3. Run smoke, model quality and load checks from the corporate network. Update private DNS through the approved failover mechanism and verify from client networks. DNS TTL and resolver caching affect effective cutover.
4. Measure time from incident declaration to verified service and the newest recoverable data and artifacts. Example targets for an interview are 60 minutes RTO and 15 minutes RPO, subject to drills and replication evidence. Serving has no database writes in this example. RPO applies to recoverable release artifacts and any state introduced later.
5. Fail back only after restoring primary health, reconciling release versions and any writes, and rehearsing the reverse switch. For future write paths, fence writes to avoid split brain.

Warm standby buys faster recovery at higher steady cost. Active-active adds routing, release coordination and data consistency work. The deciding inputs are business recovery targets, regional dependency readiness and measured drill results.
