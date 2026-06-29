# Audit Logging

**Aliases:** Audit Log, Audit Trail, Tamper-Evident Log, Compliance Log, Security Event Log, Activity Log
**Category:** Operations / Security / Compliance
**Sources:**
[Microsoft Azure — Audit Logging guidance](https://learn.microsoft.com/en-us/azure/architecture/microservices/logging-monitoring#auditing) ·
[microservices.io — Audit Logging](https://microservices.io/patterns/observability/audit-logging.html) ·
[OWASP — Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) ·
[NIST SP 800-92 — *Guide to Computer Security Log Management*](https://csrc.nist.gov/publications/detail/sp/800-92/final) ·
[PCI-DSS, HIPAA, SOX, GDPR, SOC 2 audit requirements](https://www.pcisecuritystandards.org/) ·
*Building Secure and Reliable Systems* (Adkins, Beyer, Blankinship, Lewandowski, Oprea, Stubblefield — O'Reilly 2020)

---

## Problem

> [!TIP]
> **ELI5.** Months after an incident, someone asks "who deleted that customer's data on June 14?" If your only record is the application's debug log (rotated, scattered, mutable, possibly tampered), you cannot answer. Worse: in regulated industries, you're legally required to be able to answer — financial regulators (SOX), healthcare regulators (HIPAA), payment processors (PCI), the EU (GDPR), and most enterprise customers (SOC 2) all require **tamper-evident, retained, queryable records of who did what to sensitive data**. The **audit log** is the answer: a separate, immutable, append-only record specifically for these events, designed to be reviewed by auditors and security investigators.

Application logs (see [Log Aggregation](log-aggregation-metrics.md)) are operational — they help engineers debug systems. Audit logs are different — they exist for **accountability and compliance**, with different requirements:

- **Immutable**: an audit log that can be edited isn't an audit log. Period.
- **Tamper-evident**: if the log is altered, you can prove it.
- **Long retention**: 7 years (SOX, financial), 6 years (HIPAA, healthcare), 1+ year (PCI-DSS, payment cards), indefinitely sometimes (GDPR for accountability).
- **Separately stored**: the system being audited cannot also control the audit log — that's a fox guarding the henhouse.
- **ACL-protected**: only auditors and security read; engineers cannot delete; nobody can edit.
- **Complete with no gaps**: missing events look like cover-ups.
- **Queryable**: investigators must be able to search "what did user X do between dates Y and Z?"
- **Includes provenance**: who/what/when/where for every record.

The pattern's roots are in physical paper trails (accounting ledgers, hospital records, classified material handling) — older than computing. Modern digital systems must reproduce those properties electronically. Doing so badly creates compliance failures, regulatory fines, and lost customer trust.

For startups, audit logging is often "later" — until the first SOC 2 audit, the first enterprise customer, the first regulatory question, the first incident investigation. Then it becomes urgent. Doing it right from the start is much cheaper than retrofitting under deadline.

## How it works

> [!TIP]
> **ELI5.** When a sensitive action happens — admin views a customer's data, someone changes a permission, a user is created, a payment is processed — your code emits a structured **audit event** alongside the business operation (atomically, via [outbox](../data/outbox.md), so they cannot diverge). The event flows through a separate pipeline to a separate, locked-down store designed for long retention and tamper evidence. Auditors query it; engineers cannot modify it.

### What goes in an audit log

The discipline is *what to log* — too little misses required events, too much creates noise and PII risk:

![What goes in an audit log](../diagrams/svg/audit-log.svg)

**In scope** (typical):

- **Entity changes on sensitive data**: create/update/delete of customer records, accounts, permissions, billing data, configuration, secrets, classified data.
- **Authentication events**: login success/failure, MFA challenges, password resets, account lockouts.
- **Authorization events**: role assignments/revocations, permission changes, group membership changes.
- **Privileged actions**: admin accessing customer data, support impersonating user, accessing PHI/PII/PCI data.
- **System events**: schema migrations, secret rotations, deployments, key generation, security policy changes.
- **Bulk operations**: data exports, mass updates, mass deletes.
- **Regulatory-specific events**: GDPR data access requests fulfilled, HIPAA disclosures, PCI cardholder data access.

**Not in scope** (these belong elsewhere):

- Routine application log lines → [application logs](log-aggregation-metrics.md).
- Routine HTTP access logs → access logs.
- **Secrets themselves** — log *that* a secret was rotated, not the secret value.
- **Raw PII** unless required for the audit — hash/redact otherwise.
- High-volume telemetry — metrics for that.

**Record fields** (the universal "five W's" pattern):

- **Who** — user, service principal, API key, on-behalf-of (when admin impersonates).
- **What** — action verb (`user.delete`, `role.grant`, `data.export`); affected entity; before/after state or diff.
- **When** — server-side timestamp in UTC (never client time — client clocks are untrusted).
- **Where** — source IP, user-agent, region, geographic info.
- **Why** — business reason or ticket reference if available (helps reduce false-positive alerts).
- **Correlation ID** — links to the broader request trace for forensic context.

### Architecture: separate pipeline, separate store

The architectural rule: **the audit log must be independent of the system it audits**.

![Audit log architecture](../diagrams/svg/audit-log-architecture.svg)

The flow:

1. **Application performs a sensitive action.** In the same database transaction as the business write, it inserts an `AuditEvent` row into an [outbox table](../data/outbox.md). This guarantees the audit event is emitted *whenever and only when* the business action succeeds.

2. **Audit relay reads outbox.** Either by polling or [CDC](../data/cdc.md), the relay publishes audit events to a separate audit pipeline.

3. **Audit event bus** (Kafka, Kinesis, Pub/Sub) carries all audit events from all services.

4. **Audit store** persists them. Options:
   - **WORM object storage** (AWS S3 Object Lock, Azure Immutable Blob, GCP Bucket Lock) — physically immutable; popular for high-compliance.
   - **Append-only DB** with revoked DELETE/UPDATE permissions.
   - **Dedicated SIEM** (Splunk, Sumo Logic, Datadog Cloud SIEM, Elastic SIEM, AWS Security Lake, Google Chronicle).
   - **Specialized audit databases** (Amazon QLDB — though deprecated 2024; was specifically designed for this).
   - **Hash-chained or signed records** for tamper evidence (each record's hash includes previous record's hash — similar to git or blockchain).

5. **Auditors / investigators** query via dedicated tooling, with their own RBAC.

The separation is what gives the audit log integrity:
- Engineers who built the app cannot modify audit events.
- A breach of the app doesn't compromise the audit trail.
- Operations team running the audit store does not have ability to delete records.

### Tamper evidence techniques

Real-world tamper evidence:

- **Append-only by structure**: WORM storage at the cloud-provider level (S3 Object Lock with compliance mode).
- **Revoked DELETE/UPDATE** at DB level: ACL allows INSERT and SELECT only.
- **Hash chain**: each record stores hash of previous record; tampering breaks the chain.
- **Digital signatures**: each record signed by writer's key; verifier can detect alteration.
- **Merkle tree of records**: periodic publication of root hash to external system (e.g., blockchain, public newspaper for the truly paranoid — Certificate Transparency does this).
- **Off-site replication**: copies in multiple jurisdictions, owned by different teams.
- **Air-gapped backup**: periodic export to a system isolated from production.

For most companies, WORM storage + ACLs + audit logging of the audit log itself (meta-audit) is sufficient. For high-stakes regulated environments, hash chains and signed records add cryptographic assurance.

### Atomicity: writing audit and business state together

A subtle but critical issue: if the business write succeeds but the audit write fails (or vice versa), the audit log is wrong — either omitting events that happened or recording events that didn't.

The [transactional outbox pattern](../data/outbox.md) solves this exactly:

```sql
BEGIN TRANSACTION;
  UPDATE users SET status='deleted' WHERE id=12345;
  INSERT INTO audit_outbox (event_type, entity_id, actor, action, occurred_at, payload)
    VALUES ('user.delete', 12345, 'admin-bob', 'DELETE', now(),
            '{"user_email": "alice@...", "reason": "GDPR request"}');
COMMIT;
```

A relay process reads the outbox table and publishes to the audit pipeline. Because the audit row is written in the same transaction as the business change, they cannot diverge.

This pattern is so important for audit logging that some frameworks (e.g., Hibernate Envers, ActiveRecord's PaperTrail) build it in transparently — you just configure "audit this entity" and the framework adds the audit-event insert to every save.

### Privacy considerations

Audit logs contain sensitive data by definition (who did what to whom). They must respect:

- **Need-to-know access**: only auditors / SREs / security read.
- **Redaction of unnecessary PII**: log "admin viewed customer record" not "admin viewed: {full customer JSON}".
- **Right-to-erasure (GDPR)**: tricky — you can't delete audit events for a forgotten user, but you can pseudonymize the user identifier so the event is preserved without identifying the individual.
- **Regional residency**: EU users' audit events may need to stay in EU storage.
- **Meta-audit**: log every query against the audit log (who looked at what audit data when).

These tensions make audit-log design legally nuanced. Compliance teams should be involved in design.

### Retention and archival

Different regulations have different retention requirements:

| Regulation | Audit retention requirement |
|---|---|
| SOX (financial) | 7 years |
| HIPAA (healthcare) | 6 years |
| PCI-DSS (payment cards) | 1 year online, then archive (3+ years total) |
| GDPR | Variable; "as long as necessary for accountability" |
| SOC 2 | 1+ year typically |
| FedRAMP | Often 3+ years |
| Industry-specific (FINRA, etc.) | Up to 7 years |

Practical approach:
- **Hot tier** (last 90 days): immediately queryable, in SIEM or hot store.
- **Warm tier** (1+ year): queryable with delay, cheaper storage.
- **Cold archive** (years): compressed, low-cost (Glacier-class), occasionally queryable.
- **Automated retention enforcement**: delete on schedule, but only after retention met.

### What to alert on

The audit log isn't just for post-hoc review; modern SIEMs alert on suspicious patterns in real time:

- **Privileged user accessing many customer records** (insider threat).
- **Failed login burst** (brute force).
- **Permission grants outside normal hours**.
- **First-time use of admin function by a user**.
- **Audit event burst from one source** (anomaly).
- **Export of large amounts of data** (exfiltration).
- **Audit log gaps** (tampering signal).

This is the core of **SIEM** (Security Information and Event Management): turn the audit log into a real-time security monitoring system.

### Costs and trade-offs

Audit logging is expensive in non-obvious ways:

- **Storage**: years of detailed records add up.
- **Compute**: tamper-evidence (signing, hashing) costs CPU.
- **Operational**: separate pipeline, separate ACLs, separate alerting.
- **Talent**: dedicated security/compliance team.
- **Process**: change-management for what gets logged.
- **Latency**: synchronous audit-write adds latency to sensitive paths.

But the alternative — not having an audit log — is worse:

- Regulatory fines (GDPR up to 4% of global revenue, HIPAA up to $1.5M/year, PCI up to $100K/month).
- Failed audits → lost enterprise customers.
- No forensic capability → can't investigate incidents.
- No accountability → erosion of trust internally.

Audit logging is one of those investments that looks unnecessary until the day it's urgent.

### Anti-patterns

- **Logging audit events to the same store as app logs** — no isolation, easy to delete by mistake.
- **Engineers having delete access** — audit log integrity gone.
- **Async-fire-and-forget audit emission** — drops events under load.
- **Audit log contains business secrets** (passwords, API keys, full card numbers) — turns the audit log into a sensitive data hoard.
- **Logging too much** — noise drowns out signal; PII overload.
- **Logging too little** — gaps when investigators need detail.
- **No retention policy** — store grows unbounded.
- **Manual log review only** — must integrate with SIEM / real-time alerting.
- **No separation between auditor and ops** — fox guarding henhouse.
- **Client-side timestamps** — clients lie or have clock skew.
- **Logging the secret value** instead of the secret rotation event.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Application Audit Log** | App-level: actions on entities. |
| **Database Audit Log** | DB-level: every query/change captured by the DB. |
| **OS/Kernel Audit Log** | OS-level: file access, syscall (auditd on Linux). |
| **Network Audit Log** | Network-level: firewall logs, flow logs. |
| **Cloud Audit Log** | Provider-level: AWS CloudTrail, GCP Cloud Audit Logs, Azure Activity Log. |
| **WORM Storage** | Tamper-evident at storage layer. |
| **Hash-chained / Signed Records** | Cryptographic tamper evidence. |
| **SIEM Integration** | Real-time alerting on audit events. |
| **Event Sourcing** (event sourcing) | Audit log as the primary source of truth. |
| **[Transactional Outbox](../data/outbox.md)** | Ensures atomicity with business writes. |
| **[Log Aggregation](log-aggregation-metrics.md)** | App logs; complementary. |
| **[Externalized Config](externalized-config.md)** | Config changes are themselves auditable. |
| **[Correlation ID](distributed-tracing.md)** | Links audit events to broader traces. |

## When NOT to use

- **No sensitive data / no regulatory requirements** — low-stakes systems may not need formal audit (still log security events).
- **Without isolation from production engineers** — audit log integrity gone.
- **Without retention enforcement** — store grows unbounded.
- **As a substitute for application logging** — they serve different purposes.

---

## Real-world implementations

| Tool / Service | Notes |
|---|---|
| **AWS CloudTrail** | Cloud-provider audit log. |
| **GCP Cloud Audit Logs** | Cloud-provider audit log. |
| **Azure Activity Log + Monitor** | Cloud-provider audit log. |
| **AWS Security Lake** | Centralized security data lake. |
| **Splunk SIEM / Enterprise Security** | Commercial SIEM market leader. |
| **Sumo Logic, Datadog Cloud SIEM** | Commercial SIEM. |
| **Elastic SIEM (Elastic Security)** | OSS-based SIEM. |
| **Google Chronicle** | Google's SIEM/SOAR. |
| **IBM QRadar** | Enterprise SIEM. |
| **Amazon QLDB** | Append-only ledger DB; was popular for audit (deprecated 2024). |
| **HashiCorp Vault audit devices** | Built-in audit for secret access. |
| **Auth0, Okta logs** | Identity-provider audit. |
| **OWASP Logging Cheat Sheet** | Reference. |
| **PaperTrail, Hibernate Envers** | Framework-level entity audit. |
| **Linux auditd** | OS-level audit. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **AWS** | CloudTrail is the canonical example; audit of every API call. | ✅ Verified — AWS docs |
| **GCP** | Cloud Audit Logs across all services. | ✅ Verified — GCP docs |
| **Microsoft Azure** | Activity Log + Monitor. | ✅ Verified — Azure docs |
| **All public companies (SOX)** | Required by law to have audit trails on financial systems. | ✅ Universally required |
| **All healthcare / health-tech (HIPAA)** | Required by law on PHI. | ✅ Universally required |
| **All payment processors (PCI-DSS)** | Required by industry standard. | ✅ Universally required |
| **Stripe, Square, PayPal** | Heavy audit logging for compliance. | ⚠ Implied by regulation; specific posts vary |
| **Banks, fintech, insurance** | Heavy regulatory audit logging. | ✅ Required |
| **Most SaaS with enterprise customers** | SOC 2 requires it. | ✅ Industry standard |

---

## Further reading

- OWASP Logging Cheat Sheet — practical guidance.
- NIST SP 800-92 — *Guide to Computer Security Log Management*.
- *Building Secure and Reliable Systems* (Google SRE security book, 2020) — chapter on auditing.
- *Security Engineering* (Ross Anderson, 3rd ed.) — broader security context.
- *The DevOps Handbook* — chapter on auditing and compliance.
- PCI-DSS, HIPAA, SOX, GDPR official guidance.
- Microsoft Azure audit logging docs.
- AWS CloudTrail and Security Lake docs.
- *Designing Data-Intensive Applications* — Kleppmann's discussion of event sourcing and auditability.

---

*Diagram sources: [`../diagrams/src/audit-log.d2`](../diagrams/src/audit-log.d2), [`../diagrams/src/audit-log-architecture.d2`](../diagrams/src/audit-log-architecture.d2).*
