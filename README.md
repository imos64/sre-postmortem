# SRE Postmortems

A collection of evidence-backed Site Reliability Engineering incident reports.
Each postmortem explains what failed, how the failure affected the service, how
recovery was verified, and what work remains to reduce recurrence and impact.

The purpose is operational learning: connect observations to causes, document
recovery decisions, and turn lessons into actions with clear acceptance criteria.
Reports focus on system behavior and contributing conditions rather than blame.

## Postmortem index

| Incident date | Report and architecture | Environment | Recorded outcome |
| --- | --- | --- | --- |
| 2026-09-08 | [Hyperledger Fabric Development Outage](incidents/2026-09-08-fabric-couchdb-oom/README.md) | TreeTracker development network on kind | 17 affected application Pods recovered; governance, monitoring and storage follow-ups remain |

[Full dated report](incidents/2026-09-08-fabric-couchdb-oom/2026-09-08-hyperledger-fabric-outage-postmortem.md)
· [Public evidence](incidents/2026-09-08-fabric-couchdb-oom/evidence/README.md)
· [Incident tag: 2026-09-08](https://github.com/imos64/sre-postmortem/tree/incident-2026-09-08)

## What a postmortem should establish

| Area | Questions the report addresses |
| --- | --- |
| Incident context | Which service, environment, dependencies and users were involved? |
| Impact | Which capabilities were unavailable or degraded, and what was actually measured? |
| Architecture | Where did the failure originate, and how did it propagate across dependencies? |
| Detection and timeline | What was observed, when was it observed, and where are timestamps uncertain? |
| Root cause | What mechanism explains the failure, and what evidence distinguishes it from other hypotheses? |
| Contributing conditions | Which configuration, operational or design conditions increased the likelihood or impact? |
| Recovery | What changed, why was it chosen, and which state or controls had to be preserved? |
| Verification | What service-level checks established recovery beyond process or Pod readiness? |
| Corrective actions | What prevents recurrence, limits impact, improves detection or strengthens recovery? |
| Evidence limits | Which logs are incomplete, what was not tested, and what remains unknown? |

Severity, outage duration, customer impact and data-loss claims should reflect
available evidence. When a value is unknown, the report states that uncertainty
rather than inventing a precise figure. A likely trigger is distinguished from a
confirmed cause; a running dependency is distinguished from a verified service.

## Repository structure

```text
incidents/
└── YYYY-MM-DD-incident-slug/
    ├── README.md                         Incident guide and architecture
    ├── YYYY-MM-DD-...-postmortem.md       Full dated postmortem
    └── evidence/
        ├── README.md                     Evidence inventory and capture limits
        ├── *.txt / *.json                Reviewed excerpts and verification records
        ├── PROVENANCE.json              Origins and documented transformations
        └── SHA256SUMS                    Published evidence checksums
```

Start with an incident's README for its system context and architecture. Read the
full dated report for the detailed timeline, decisions and follow-up register.
Use the evidence index to inspect the observations supporting each conclusion.

## Evidence and publication standards

- Preserve historical results and identify the environment and time of each
  observation. A recovered incident is not a statement of current service health.
- Distinguish raw command output, selected excerpts, investigator summaries and
  later analysis. Record redactions, filtering and truncation explicitly.
- Link public evidence to source records and include checksums for the published
  files. Checksums establish byte integrity, not independent proof of an incident.
- Keep credentials, signing keys, private configuration and unrestricted raw
  archives outside this public repository. Publish reviewed material sufficient
  to explain and assess the conclusions.
- Identify upstream links that require private repository access. Include public
  summaries where readers cannot inspect those original systems directly.

## Corrective action tracking

Actions should name a priority, an owner or clearly proposed owner, and a
verifiable completion criterion. Separate immediate restoration from longer-term
prevention, detection, capacity and disaster-recovery improvements.

Do not mark an action complete because a patch exists. Its acceptance evidence
should match the action: for example, a delivered alert, a successful restore,
an identity-rotation check, or a controlled restart that preserves service health.
Open actions remain visible in the incident report.

## Maintaining the collection

For a new incident, add its dated directory, architecture guide, report and
reviewed evidence, then update the index above. Use an incident-date tag to
identify a published snapshot. Later documentation improvements can appear on
`main` without rewriting an existing tag or changing original evidence bytes.

Before publication, check links, diagram rendering, factual consistency,
credential exclusions and evidence checksums. This repository contains incident
documentation; recovery actions described in a report are historical records,
not instructions to run against another environment without review.
