# Visa — Wallet Tokenization & Platform Migration (Jul 2018 – Oct 2021)
Role: Senior SWE, Platform Engineering · Denver

## Impact
- Card provisioning for Apple Pay / Google Pay / Samsung Pay tokenization at network scale.
- 12+ microservices migrated on-prem → AWS EKS with zero downtime; pattern became the org standard.
- p99 latency −60% on the transaction history service; 99.99% availability DR for payment authorization.

## The story in 2 minutes
Built card provisioning APIs (Java) for digital-wallet tokenization and the issuer decision
service approving/declining transactions in real time across millions of daily events inside
the authorization latency budget. Kafka event fan-out of authorization and card-lifecycle
events to partner banks with retries and DLQs. EKS migration with Terraform. Transaction
history p99 cut via index redesign, partition pruning, caching, read replicas. Multi-region
active-passive DR sustaining 99.99% for authorization.

## Hardest decisions / trade-offs
- Active-passive vs active-active DR (consistency vs availability for authorization).
- Migration sequencing: strangler pattern vs big-bang per service.
- Caching transaction history: freshness expectations vs p99.

## Nasty questions
1. Zero-downtime migration of a payment service — walk the cutover minute by minute.
2. Tokenization: what does the issuer decision service actually decide, on what data?
3. Active-passive failover — RPO/RTO numbers, and how were they tested?
4. p99 −60% — which single change mattered most, and how do you know?
5. Partner bank consumer is down — fan-out backlog behavior?

## Gaps to fill
- [ ] Tokenization volume, decision-service latency budget, DR test cadence
