---
name: trueforge-policy-and-steps
description: Authoring guide and generator for TrueForge YAML safety policies and skill-like runbook steps. Use when configuring guardrail rules in config/policy.yaml or authoring executable YAML runbooks in runbooks/*.yaml.
---

# TrueForge Policy & Runbook Steps Generation Guide

This skill guides you through defining safety policies in `config/policy.yaml` and authoring skill-like, human-readable runbook playbooks in `runbooks/*.yaml`.

---

## Part 1: Generating & Authoring `config/policy.yaml`

The TrueForge Policy Engine enforces real-time guardrails before any command or tool mutation is executed.

### 1.1 Policy Architecture & Schema

`config/policy.yaml` defines:
1. **Reversible Actions & Commands:** Automatically captured in pre-flight snapshots and auto-executed.
2. **Destructive Actions & Commands:** Automatically paused for explicit human SRE approval (`YES` / `NO`).
3. **Policies (Ordered Rules):** Evaluated top-to-bottom. The **first** rule where all conditions match determines the verdict (`allowed`, `requiresHumanApproval`, and `reason`).
4. **Default Action:** Fallback verdict if no explicit rule matches.

```yaml
version: "2.0"
name: "TrueForge Kubernetes Operational Safety Policy"
description: >
  Two-tier safety policy: reversible commands auto-execute with pre-flight snapshots;
  destructive commands require explicit human SRE approval.

reversibleActions:
  - restart_deployment
  - scale_deployment
  - scale_resource
  - rollout_undo
  - apply_manifest
  - patch_resource
  - update_deployment_image

destructiveActions:
  - delete_resource
  - force_delete_pod
  - drain_node
  - cordon_node
  - scale_to_zero

# Raw command keyword guardrails:
destructiveCommands:
  - "delete"
  - "drain"
  - "cordon"
  - "--replicas=0"
  - "scale 0"
  - "--force"
  - "rm "
  - "truncate"

reversibleCommands:
  - "rollout restart"
  - "scale"
  - "set image"
  - "apply"
  - "patch"

policies:
  # Rule 1: Safety block on sensitive namespaces (evaluated first!)
  - name: "KUBE_SYSTEM_MUTATIONS_BLOCKED"
    description: "Mutations targeting critical cluster control-plane namespaces are blocked."
    conditions:
      namespaces: ["kube-system", "kube-public", "kube-node-lease"]
    action:
      allowed: false
      requiresHumanApproval: false
      reason: "Mutations targeting Kubernetes system namespaces are permanently forbidden."

  # Rule 2: Reversible actions in non-prod
  - name: "REVERSIBLE_NON_PROD_AUTO"
    description: "Reversible operations in dev/staging environments execute automatically."
    conditions:
      actionTypes: ["reversible"]
      environment: ["dev", "development", "staging", "test", "local"]
    action:
      allowed: true
      requiresHumanApproval: false
      reason: "Reversible action in non-production — auto-executed with state capture."

  # Rule 3: Reversible actions in production
  - name: "REVERSIBLE_PROD_AUTO_WITH_AUDIT"
    description: "Reversible operations in production run automatically with audit trail."
    conditions:
      actionTypes: ["reversible"]
      environment: ["production", "prod"]
    action:
      allowed: true
      requiresHumanApproval: false
      reason: "Reversible production action — auto-executed with pre-flight snapshot."

  # Rule 4: Scale to zero is always treated as destructive
  - name: "SCALE_TO_ZERO_ALWAYS_APPROVAL"
    description: "Scaling workloads to zero causes downtime and requires approval."
    conditions:
      actionNames: ["scale_to_zero"]
    action:
      allowed: true
      requiresHumanApproval: true
      reason: "Scaling to zero replicas causes immediate downtime — human approval required."

  # Rule 5: Destructive actions require human approval
  - name: "DESTRUCTIVE_ALWAYS_REQUIRES_APPROVAL"
    description: "Destructive operations halt for human SRE approval."
    conditions:
      actionTypes: ["destructive"]
    action:
      allowed: true
      requiresHumanApproval: true
      reason: "Destructive Kubernetes operation — human approval required before execution."

defaultAction:
  allowed: true
  requiresHumanApproval: false
  policyName: "STANDARD_AUTO_EXECUTION"
  reason: "Operation permitted under standard automated execution policy."
```

### 1.2 Rule Precedence Best Practices
- **Hard Blocks First:** Always place namespace security blocks (e.g. `KUBE_SYSTEM_MUTATIONS_BLOCKED`) at the top of the `policies` list before general approval rules.
- **Specific Before General:** Special destructive cases (such as `scale_to_zero`) should precede generic `actionTypes: ["destructive"]`.
- **Exact Keyword Substrings:** Any raw command containing any substring in `destructiveCommands` automatically triggers the destructive approval flow.

---

## Part 2: Generating Skill-Like Runbook Steps (`runbooks/*.yaml`)

TrueForge runbooks use clean, skill-like instructions rather than rigid machine action schemas. Each step guides both the human engineer and AI agent with clear diagnostic, remediation, and verification instructions containing concrete `kubectl` commands.

### 2.1 Runbook Schema

Save runbook files directly in the `./runbooks/` directory (e.g. `./runbooks/rb-k8s-scale-payment.yaml`):

```yaml
runbookId: "rb-k8s-<slug>"
name: "<Human Readable Playbook Title>"
targetService: "<target-microservice-name>"
whatItProvides: "<One sentence summary of the playbook outcome>"
scenario: "<Triggering conditions, alert names, or metric thresholds>"
steps:
  - step: 1
    phase: "Diagnosis"
    instruction: "<Investigation instructions including diagnostic kubectl commands>"
  - step: 2
    phase: "Remediation"
    instruction: "<Remediation instructions with exact executable kubectl command>"
  - step: 3
    phase: "Verification"
    instruction: "<Validation commands to verify service recovery and readiness>"
  - step: 4
    phase: "Incident Resolution"
    instruction: "<Steps to record the pre-flight snapshot ID and resolve the incident ticket>"
```

### 2.2 Standard 4-Phase Runbook Pattern

When authoring steps for any production playbook, structure them across these 4 phases:

1. **Phase 1: Diagnosis**
   - Directs the agent/operator to inspect the affected workload.
   - Example: `Inspect payment-service pod health and ready replicas using 'kubectl get pods -l app=payment-service -n production' and check recent logs for timeout patterns.`
2. **Phase 2: Remediation**
   - Details the primary healing command (e.g. scale, rolling restart, rollback, or patch).
   - Example: `Scale payment-service deployment up to 5 replicas using 'kubectl scale deployment payment-service --replicas=5 -n production' to absorb incoming load.`
3. **Phase 3: Verification**
   - Specifies the verification command to confirm health restore.
   - Example: `Verify rollout status using 'kubectl rollout status deployment/payment-service -n production' and confirm all 5 pods are Ready.`
4. **Phase 4: Incident Resolution**
   - Instructs updating the incident ticket with the pre-flight snapshot ID and resolution notes.
   - Example: `Post resolution summary to incident ticket with execution details, pre-flight snapshot ID, and MTTR confirmation.`

---

## Part 3: Concrete Playbook Examples

### Example A: Pod Restart & Memory Stabilization
File: `runbooks/restart_service.yaml`
```yaml
runbookId: "rb-k8s-restart-service"
name: "CrashLoop & Memory Leak Stabilization Playbook"
targetService: "payment-service"
whatItProvides: "Graceful zero-downtime rolling restart and worker eviction for memory-leaking, degraded, or CrashLooping pods."
scenario: "Triggered when pods exceed 90% memory threshold, fail liveness probes, or enter CrashLoopBackOff state."
steps:
  - step: 1
    phase: "Diagnosis"
    instruction: "Check pod status, restart counts, and container termination messages for OOMKilled signals using `kubectl describe pod -l app=payment-service -n production`."
  - step: 2
    phase: "Remediation"
    instruction: "Trigger zero-downtime rolling restart of payment-service pods to reload runtime memory and clear deadlocks using `kubectl rollout restart deployment/payment-service -n production`."
  - step: 3
    phase: "Verification"
    instruction: "Poll Kubernetes API to verify fresh pods have spun up and all endpoints report healthy status using `kubectl rollout status deployment/payment-service -n production`."
  - step: 4
    phase: "Incident Resolution"
    instruction: "Log completion in Jira ticket noting zero-downtime restart metrics and memory recovery."
```

### Example B: Horizontal Autoscaling Under High Traffic
File: `runbooks/scale_payment.yaml`
```yaml
runbookId: "rb-k8s-scale-payment"
name: "Payment Service High-Load Scaling Playbook"
targetService: "payment-service"
whatItProvides: "Automated scaling, rollout recovery, and SLO stabilization for payment-service under heavy traffic or memory saturation."
scenario: "Triggered when payment-service latency exceeds 500ms, HTTP 5xx error rate spikes > 3%, or Prometheus alert HighErrorRate fires."
steps:
  - step: 1
    phase: "Diagnosis"
    instruction: "Inspect payment-service pod health and ready replicas using `kubectl get pods -l app=payment-service -n production` and check recent logs for timeout patterns."
  - step: 2
    phase: "Remediation"
    instruction: "Scale payment-service deployment up to 5 replicas using `kubectl scale deployment payment-service --replicas=5 -n production` to absorb incoming load."
  - step: 3
    phase: "Verification"
    instruction: "Verify rollout status using `kubectl rollout status deployment/payment-service -n production` and confirm all 5 pods are Ready."
  - step: 4
    phase: "Incident Resolution"
    instruction: "Post resolution summary to Jira incident ticket with execution details, pre-flight snapshot ID, and MTTR confirmation."
```

---

## Part 4: Validation & Testing

After generating or modifying `config/policy.yaml` or runbook steps in `runbooks/*.yaml`, always run the test suites to ensure syntax and policy correctness:

1. **Verify Policy Engine & Rules:**
   ```bash
   npm run test:policy
   ```
   Validates YAML syntax, rule parsing, guardrail evaluation, and namespace protection.

2. **Verify Tool Registry:**
   ```bash
   npm run test:tools
   ```
   Ensures all 21 MCP tools are registered and ready.

3. **Verify End-to-End Governed Execution:**
   ```bash
   npx ts-node src/test_governed_command_e2e.ts
   ```
   Validates pre-flight snapshot capture, reversible command auto-execution, destructive command pause, and human SRE approval.

4. **Verify In-Built Incident Tracker:**
   ```bash
   npx ts-node src/test_incident_tickets.ts
   ```
   Validates ticket creation, pre-flight snapshot linkage, and SQLite updates.
