# UniWeave Roadmap

Last updated: 2026-03-06

Two layers, four platform components, six vertical packages.

---

## Layer 1 — Core platform (shared across all verticals)

These capabilities underpin every vertical deployment. They must be stable before vertical packages can be repeatable.

**1A — Vertical Intelligence Engine**
- Domain-specific AI training framework (per-vertical data pipelines and model tuning)
- Industry terminology recognition and intent classification
- Vertical system integrations (GDS, PMS, billing systems, EHR, logistics APIs)
- Policy and compliance logic per vertical (fare rules, HIPAA, PCI DSS)
- Evaluation framework: per-vertical accuracy benchmarks and golden test sets

**1B — Omni-Channel Execution**
- Unified interaction runtime across voice, chat, email, SMS, social, messaging
- Real-time voice transcription with industry terminology correction
- Visual AI for document processing (images, PDFs, forms)
- Contextual conversation state across channel switches
- Interrupt handling, barge-in, turn detection (voice-specific)

**1C — Human-in-the-Loop Orchestration**
- Intelligent routing engine (complexity, sentiment, customer value signals)
- Real-time agent assist (next-best-action, knowledge retrieval)
- Full context transfer on AI-to-human handoff
- Post-call AI summarisation and case note population
- Continuous learning loop: human correction feedback to model improvement

**1D — Analytics & Insights**
- 100% interaction coverage (AI and human)
- Industry-specific KPI dashboards
- Containment rate, resolution time, CSAT, escalation rate tracking
- Predictive analytics for volume and staffing
- Quality monitoring and error taxonomy

**1E — Platform Foundation**
- Outcome-based pricing infrastructure (containment tracking, billing, reporting)
- Governance: audit trails, policy controls, RBAC, compliance artefacts
- Observability: end-to-end traces, latency monitoring, tool success rates
- Multi-model orchestration and provider failover
- C-Zentrix integration layer (voice streaming, WebRTC, call lifecycle)

---

## Layer 2 — Vertical execution packages

Each vertical package includes: pre-trained domain models, workflow playbooks, system integration connectors, vertical-specific evals, and deployment templates.

| Vertical | Key capabilities | Status |
|---|---|---|
| **Travel (Airlines)** | GDS integration (PNR, rebooking, seat changes), flight disruption workflows, loyalty actions | **Active delivery** — Saudia + IndiGo |
| **Hospitality** | Reservation management, check-in/out, loyalty, service requests | **First customer** — Chalet Hotels |
| **Logistics & Transport** | Shipment tracking, delivery exceptions, route queries, customs docs | Roadmap |
| **Telecom** | Billing disputes, plan changes, tech troubleshooting, outage notifications | Roadmap |
| **Healthcare** | Appointments, insurance verification, HIPAA-compliant interactions | Roadmap |

---

## Phasing

**Phase 1 — Stabilise and productise (current)**
- Stabilise core platform layers 1A–1E
- Convert Saudia/IndiGo deliveries into repeatable Travel vertical package
- Ship Chat Module (multimodal), IntelliRAG, IntelliPulse integration
- AI Gateway deployed and routing production traffic
- Unified auth/RBAC across all services

**Phase 2 — Prove repeatability**
- Deploy Travel package to second airline customer
- Select and begin second vertical (Hospitality in progress via Chalet)
- Enterprise certifications (Salesforce AppExchange as first target)
- Analytics layer as customer-facing capability

**Phase 3 — Scale**
- Second vertical package to production
- C-Zentrix co-sell pipeline generating qualified opportunities
- 50+ logos target, Americas/EMEA/APAC presence

---

## April 23 milestone (within Phase 1)

Internal gate: April 9. Public launch: April 23 (company 2-year anniversary).

- 60% already live (Saudia + IndiGo deployments)
- Net new build: Chat Engine + session management (highest risk), IntelliRAG config UI, Pulse integration, AI Gateway deploy
- See [VISION.md](VISION.md) for the full April 23 checklist
