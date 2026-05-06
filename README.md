# Permitting & Licensing Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native platform for online permit applications, review workflows, and inspection scheduling — built for the government agencies and applicants underserved by today's enterprise GovTech incumbents.

This project aims to deliver a modern permitting and licensing system covering application intake, multi-department workflow routing, plan review, inspection scheduling, fee processing, and code enforcement. It targets city and county building departments, licensing divisions, and state agencies that need a configurable platform without the cost and implementation overhead of legacy enterprise suites.

---

## Why Permitting & Licensing Platform?

- Enterprise incumbents (Tyler EnerGov, Oracle, Accela) range from $100,000 to $1 million+ per year for large municipalities, with implementation timelines that stretch into years.
- No production-ready open-source platform exists in this category — every major product is commercial proprietary, locking governments into long-term vendor relationships.
- AI-assisted plan review is only just emerging at incumbents (Clariti/CivCheck, Accela ePermitHub) and is not natively integrated into the market leader as of early 2026.
- Mid-market platforms still leave gaps in advanced analytics, cross-jurisdiction interoperability, and BLDS-compliant open data publishing.
- More than 68% of agencies have shifted from manual workflows to automated platforms, but smaller municipalities remain underserved by affordable, modern alternatives.

---

## Key Features

### Application Intake & Workflow

- Online permit and licence application intake with configurable form fields and conditional logic
- Workflow engine with step assignment, reviewer queues, status tracking, and SLA thresholds
- Multi-department routing across planning, building, public works, and fire
- Constituent self-service portal for submission, status tracking, document upload, and payment

### Plan Review & Inspections

- Digital plan review with markup tooling (Bluebeam-compatible or native)
- Inspection scheduling, assignment, and results recording with mobile access
- Photo documentation and offline field capability for inspectors
- Re-inspection management and real-time result publishing to applicants

### Licensing & Code Enforcement

- Business and contractor licence lifecycle management (application, issuance, renewal, suspension)
- Code enforcement case management with violation notice generation
- Fee calculation engine with configurable schedules and online payment processing

### AI-Assisted Capabilities

- AI-powered completeness checker validating uploaded documents against permit type requirements
- Natural-language eligibility assistant guiding applicants to the correct permit pathway
- AI plan review automation for code compliance checking against IBC and fire code rules
- Predictive inspection scheduling optimisation across inspector workloads and routes

### GIS, Data & Integrations

- GIS parcel lookup using an open or pluggable GIS layer
- BLDS-compliant open permit data export API
- Email/SMS automated notifications at each workflow milestone
- Analytics dashboard for processing times, workload by department, and application volume trends

---

## AI-Native Advantage

AI is woven through the workflow rather than bolted onto a legacy core. Computer vision pre-screens construction drawings against IBC code rules to compress plan review from weeks to days; intelligent intake validates completeness and routes applications without human triaging; ML models optimise inspector schedules across geography, complexity, and historical duration; and a conversational eligibility assistant maps plain-language project descriptions to the correct permit pathway. Anomaly detection on licence renewals automates compliance enforcement that incumbents handle manually.

---

## Tech Stack & Deployment

The platform is designed for cloud-native deployment with self-hosting support for jurisdictions with data residency requirements. Cloud deployment accounts for 58.7% of the existing market. Architecture aligns with relevant open standards: BLDS for permit data exchange, Open311 for service request APIs, CityGML/GML for parcel and land use data, and ICC Digital Codes for inspection rule references. Government-facing interfaces target ADA, Section 504, and Section 508 accessibility compliance. Plan review integrates with Bluebeam and DigEplan; GIS layers connect via Esri-compatible and open alternatives.

---

## Market Context

The narrow government permit software market was approximately $266 million in 2026, projected to reach $401 million by 2035 (4.7% CAGR). Broader enterprise permitting and licensing definitions place the 2025 market at $3.8–4.8 billion, growing to $8.6–10.6 billion by 2034 at roughly 9–10% CAGR. Tyler Technologies holds approximately 14% global share. Primary buyers are city and county building departments, local licensing divisions, state environmental and transportation agencies, and construction contractors seeking faster permit turnaround.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
