# Permitting & Licensing Platform — Feature & Functionality Survey

> Candidate #228 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Tyler Technologies (EnerGov) | Government SaaS | Commercial — enterprise | https://www.tylertech.com/products/enterprise-permitting-licensing |
| Accela Civic Platform | Government SaaS | Commercial — enterprise | https://www.accela.com/ |
| Oracle Permitting and Licensing | Government SaaS | Commercial — enterprise | https://www.oracle.com/government/state-local/permitting-licensing-software/ |
| OpenGov Permitting & Licensing | Government SaaS | Commercial — subscription | https://opengov.com/products/permitting-and-licensing/ |
| Cloudpermit | Government SaaS | Commercial — subscription | https://cloudpermit.com/ |
| Granicus SmartGov | Government SaaS | Commercial — subscription | https://granicus.com/product/permitting-compliance-licensing-smartgov/ |
| Clariti (formerly Camino) + CivCheck | Government SaaS | Commercial — subscription | https://www.claritisoftware.com/ |
| PermitFlow | Contractor SaaS | Commercial — per-permit / subscription | https://www.permitflow.com/ |
| Salesforce Public Sector (Permitting) | Platform SaaS | Commercial — enterprise | https://www.salesforce.com/government/state-local-government-software/permitting-software/ |
| iWorQ | Government SaaS | Commercial — subscription | https://iworq.com/ |
| MyGovernmentOnline | Government SaaS | Commercial — no upfront cost | https://www.mygovernmentonline.org/ |

---

## Feature Analysis by Solution

### Tyler Technologies EnerGov

**Core features**
- Permit application intake, review routing, approval, and issuance
- Business and professional licence management with renewal workflows
- Inspection scheduling, dispatch, and results recording
- GIS-embedded parcel lookup using Esri ArcGIS technology
- Electronic plan review and digital document management
- Code enforcement tracking and violation notices
- Fee calculation, payment processing, and receipt generation
- Constituent self-service portal (online applications and status tracking)
- Multi-department workflow routing across planning, building, public works, and fire
- Asset management integration

**Differentiating features**
- Deep Esri integration providing dimensional GIS visualisation across all permit records
- Broad departmental coverage spanning community development, transportation, land control, storm water, and fire safety
- MyGov acquisition (2025) extending reach to smaller municipalities

**UX patterns**
- Staff-facing desktop interface with GIS map views
- Citizen-facing portal for online application submission and status lookup
- Mobile inspection app for field work

**Integration points**
- Esri ArcGIS (embedded GIS)
- SeeClickFix CRM via REST integration
- Laserfiche document management
- Payment gateways (jurisdiction-specific)
- Tyler's broader ERP suite (Munis, New World)

**Known gaps**
- Long and complex implementation timelines
- Limited advanced analytics and operational reporting out of the box
- AI-assisted plan review not yet natively integrated (as of early 2026)
- High cost constrains smaller-municipality adoption

**Licence / IP notes**
- Commercial proprietary; government enterprise licensing; no open-source components identified

---

### Accela Civic Platform

**Core features**
- Permit and licence application intake, review, and issuance
- Configurable workflow engine with task assignment and SLA tracking
- Inspection scheduling, routing, and field mobile app
- Online applicant portal (new Public Portal replacing legacy Citizen Access)
- GIS integration for parcel visualisation and inspection routing
- Fee calculation and payment processing
- Document management with plan review tools
- Code enforcement case management
- Open REST API (Construct API v4) for third-party integration

**Differentiating features**
- ePermitHub (acquired 2025) adds AI-driven plan review automation and COMET Version Engine for automated code compliance checking
- New Public Portal provides modern constituent UX as a replacement for the legacy Citizen Access (ACA) product
- Mature API ecosystem used by 600+ agency implementations worldwide

**UX patterns**
- Agency staff portal with workflow dashboards
- New Public Portal for applicant self-service (modern, mobile-responsive)
- Mobile inspector app with offline capability

**Integration points**
- REST API (developer.accela.com) supporting iOS, Android, and web apps
- GIS systems (Esri and others)
- Payment processors
- Document management systems
- OpenCounter business licensing pre-screening integration

**Known gaps**
- Legacy architecture (ACA) still in use at many agencies pending portal migration
- Complex configuration for jurisdiction-specific fee schedules and workflows
- Requires significant implementation project effort for enterprise deployments

**Licence / IP notes**
- Commercial proprietary; enterprise government licensing

---

### Oracle Permitting and Licensing

**Core features**
- Online permit and licence application submission and status tracking
- Configurable multi-step approval workflows
- Inspection scheduling and results recording
- Business licensing lifecycle management (application, issuance, renewal, suspension)
- Code enforcement case management
- Fee calculation, online payments, and reconciliation
- Planning and zoning application management
- Constituent self-service portal
- Reporting and analytics dashboards

**Differentiating features**
- Oracle Fusion Cloud integration — shares infrastructure with Oracle ERP, HCM, and financial systems
- Purpose-built for state and large local government with enterprise-grade scalability
- Bi-annual cloud update cadence (26A, 26B releases) keeping platform current

**UX patterns**
- Guided intake questionnaires for applicants
- Staff workflow queues with SLA visibility
- Executive analytics dashboards for permit volumes and compliance metrics

**Integration points**
- Oracle Fusion Cloud ERP and financial systems
- REST APIs documented through Oracle Help Center
- Oracle Integration Cloud for third-party connectors

**Known gaps**
- High cost and Oracle licensing complexity deters mid-market adoption
- Implementation requires certified Oracle consultants
- Less flexible for highly custom jurisdiction-specific workflows compared to pure GovTech platforms

**Licence / IP notes**
- Commercial proprietary Oracle cloud licensing

---

### OpenGov Permitting & Licensing

**Core features**
- No-code drag-and-drop form builder with conditional logic
- Visual workflow designer with step assignment and branching
- Online permit and business licence application intake
- Fee calculation engine with configurable fee schedules
- Online payment processing
- Inspection scheduling and results tracking
- Applicant self-service portal with real-time status visibility
- GIS integration for parcel lookup (Esri partnership)
- Code enforcement module
- AI-generated form creation to accelerate configuration

**Differentiating features**
- Fastest time-to-launch among enterprise alternatives — agencies can go live in as few as six months
- Applications processed up to 5x faster than legacy manual systems
- AI-generated form builder reduces configuration effort significantly
- Cox Enterprises ownership provides stability and investment capacity

**UX patterns**
- No-code admin interface for workflow and form configuration without developer involvement
- Visual workflow tracker visible to applicants during review
- Mobile-friendly applicant portal

**Integration points**
- Esri ArcGIS (Esri partner)
- OpenGov CRM, budgeting, and reporting modules
- Payment gateways
- REST API via OpenGov Developer Portal (developer.opengov.com)

**Known gaps**
- Depth of code enforcement features trails Accela and Tyler for complex compliance workflows
- API ecosystem less mature than Accela's for deep third-party customisation
- Smaller agency reference list than legacy incumbents

**Licence / IP notes**
- Commercial proprietary; government SaaS subscription

---

### Cloudpermit

**Core features**
- Building permit application intake and processing
- Plan review with integrated Bluebeam and DigEplan connectors
- Mobile field inspection with offline capability
- Inspection scheduling and results recording with photo documentation
- Business licensing and planning application modules
- Code enforcement management
- Applicant self-service portal with automated status notifications
- Configurable workflow routing

**Differentiating features**
- Best-in-class mobile inspection experience — full offline functionality with photo attachment
- Seamless plan review integration directly within the platform (Bluebeam, DigEplan)
- Cost-effective pricing targeted at mid-sized municipalities as a Tyler/Accela alternative

**UX patterns**
- Clean cloud-native interface with minimal training requirement
- Mobile-first inspector experience (iOS/Android)
- Applicant portal with automated email/SMS status updates

**Integration points**
- Bluebeam Studio for digital plan review markup
- DigEplan for electronic plan review
- Payment gateways
- GIS parcel data connectors

**Known gaps**
- Smaller integration ecosystem compared to Accela or Tyler
- Limited advanced analytics and open data publishing tools
- Less configurable for complex multi-department enterprise workflows
- Primarily building permit focused; licensing depth limited

**Licence / IP notes**
- Commercial proprietary; government SaaS subscription

---

### Granicus SmartGov

**Core features**
- Permit application intake, routing, and issuance
- Plan review workflow with reviewer assignment and comment tracking
- Inspection scheduling, real-time results, and re-inspection management
- Business and contractor licence management
- Code enforcement case tracking
- Fee calculation and online payment
- ArcGIS GIS layer visualisation alongside permit and enforcement records
- Constituent self-service portal with permit status tracking
- Configurable workflow routing across departments

**Differentiating features**
- Part of Granicus government experience platform — integration with govDelivery notifications, govAccess web services, and citizen engagement tools
- Strong inspection workflow with real-time result publishing to applicants

**UX patterns**
- Staff interface with GIS map overlays for permit/enforcement visualisation
- Constituent portal for online application, payment, and status
- Mobile field app for inspectors

**Integration points**
- Esri ArcGIS
- Granicus govDelivery (mass notification)
- Payment processors
- Document management systems

**Known gaps**
- Document upload workflow has been flagged as cumbersome in user reviews
- Integration with external systems (GIS, payment, document management) can require project work
- Not a full end-to-end administrative platform — finance and HR must be covered separately
- Configuration of jurisdiction-specific fee schedules can be time-consuming

**Licence / IP notes**
- Commercial proprietary; government SaaS subscription; part of Granicus suite

---

### Clariti (formerly Camino Technologies) + CivCheck AI

**Core features**
- Online permit application intake and tracking
- Digital plan review with multi-cycle tracking
- AI plan review automation (CivCheck, acquired October 2025): 97%+ accuracy with claim of 80%+ faster permit approvals
- AI-assisted completeness checking and intake routing
- AI copilots guiding applicants through code checks
- Inspection scheduling (including virtual inspections)
- GIS parcel detail integration
- Development Guide helping applicants determine permitted uses and applicable permits
- Business licensing module

**Differentiating features**
- CivCheck is the first AI plan review platform supporting every code check required for building permits — not just basic pre-screening
- AI intake validation flags missing documents before staff review
- Virtual inspection capability for remote project assessment
- AI document generation and AI code compliance for inspections (in development, 2026)

**UX patterns**
- Applicant-friendly guided portal with AI assistance at intake
- Staff portal with AI-powered validation queue
- AI copilot surfacing relevant code sections and pre-check calculations on one screen

**Integration points**
- GIS parcel data
- e-PlanSoft integration for digital plan review (announced)
- API for third-party permitting platform integration (planned for Clariti Enterprise, spring 2026)
- Bluebeam and other plan review tools

**Known gaps**
- Newer entrant with smaller agency reference list than Tyler, Accela, or Oracle
- CivCheck integrations with third-party permitting platforms still rolling out
- Smaller integration ecosystem

**Licence / IP notes**
- Commercial proprietary; government SaaS subscription; CivCheck AI proprietary

---

### PermitFlow

**Core features**
- Jurisdiction requirements research using AI-powered Research Agent
- Permit package preparation and form completion
- Direct filing with authority having jurisdiction (AHJ) on behalf of contractor
- AHJ comment tracking and response management
- Real-time permit status monitoring across 1,500+ US municipalities
- Automatic data pull from CRM, project files, and contracts
- Permit expiry and renewal reminders
- Multi-project portfolio tracking

**Differentiating features**
- Contractor-side (applicant-side) tool, not a government-side platform — uniquely optimised for construction companies filing permits across many jurisdictions
- AI Research Agent that searches AHJ portals for jurisdiction-specific requirements, fees, and timelines
- Claims 50%+ reduction in permit approval times through streamlined preparation and follow-up

**UX patterns**
- Project dashboard with permit status timeline per jurisdiction
- Automated form-fill reducing manual data entry
- Stakeholder notification and status sharing

**Integration points**
- CRM integrations for automatic data pull
- AHJ portal filing (jurisdiction-specific connectors)
- 1,500+ jurisdiction database

**Known gaps**
- Not suitable for the government agency side of permitting administration
- Limited to US jurisdictions
- Does not provide government workflow, inspection, or code enforcement tools

**Licence / IP notes**
- Commercial proprietary; subscription and per-permit pricing

---

### Salesforce Public Sector (Permitting)

**Core features**
- Online permit and licence application submission with smart dynamic forms
- Permit tracking and status management
- Licence lifecycle management (application, issuance, renewal)
- Constituent self-service portal with chatbot support
- Configurable workflow automation using Salesforce Flow
- Fee management and payment processing integration
- Analytics dashboards (Salesforce Analytics) for permit volumes, geography, type, and status
- CRM integration across all constituent interactions

**Differentiating features**
- Built on Salesforce CRM — unifies constituent permitting, service requests, and engagement history in a single platform
- Salesforce Einstein AI capabilities available for conversational intake and analytics
- Broadest integration ecosystem of any platform via Salesforce AppExchange and MuleSoft

**UX patterns**
- Single portal for residents and businesses covering applications, payments, renewals, and licence updates
- Dynamic intake questionnaires reducing unnecessary fields
- Chatbot and resource articles for self-service resolution

**Integration points**
- MuleSoft for enterprise API integration
- Salesforce AppExchange (thousands of third-party apps)
- DocuSign for e-signature
- Payment processors
- GIS integrations

**Known gaps**
- Less purpose-built for government permitting depth (plan review, inspection routing) — requires more configuration
- Higher total cost of ownership with Salesforce licensing plus implementation
- Requires Salesforce expertise for configuration and maintenance

**Licence / IP notes**
- Commercial proprietary; Salesforce enterprise licensing

---

### iWorQ

**Core features**
- Permit and licence tracking across building, land use, zoning, variances, and encroachments
- Inspection scheduling and results recording
- Code enforcement case tracking
- Public works work orders and asset management
- Citizen/contractor self-service portal with web forms
- TextMyGov citizen communication integration
- Real-time updates accessible from any device
- Fee tracking and parcel/owner/contractor information management

**Differentiating features**
- Affordable pricing specifically targeting small and mid-sized municipalities
- Broad public works and asset management scope alongside permitting
- TextMyGov integration for SMS-based constituent communication

**UX patterns**
- Simple web-based staff interface designed for smaller department teams
- Contractor/citizen portal with web forms
- Mobile access for field staff

**Integration points**
- TextMyGov (citizen communication)
- Payment gateways
- GIS (basic parcel lookup)

**Known gaps**
- Lacks advanced automated plan review features
- Limited complex multi-department workflow automation compared to enterprise solutions
- Analytics and reporting are basic
- Not suited to large or complex permit workloads

**Licence / IP notes**
- Commercial proprietary; subscription pricing

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Online permit and licence application intake with electronic form submission
- Workflow routing with step assignment, branching, and status tracking
- Fee calculation engine with configurable schedules and online payment processing
- Constituent self-service portal with real-time status visibility
- Inspection scheduling, dispatch, and results recording
- Document upload and management
- Email/SMS automated status notifications
- GIS parcel lookup integration
- Mobile inspector access (iOS/Android)
- Reporting dashboards for permit volumes and processing times

### Differentiating Features
- AI-powered plan review automation with code compliance checking (Clariti/CivCheck)
- No-code drag-and-drop form and workflow builder (OpenGov)
- Contractor-side multi-jurisdiction filing and research (PermitFlow)
- Best-in-class offline mobile inspection with integrated plan review markup (Cloudpermit)
- AI-assisted intake completeness checking and intelligent document routing
- Deep Esri GIS integration with dimensional map visualisation (Tyler)
- Comprehensive CRM integration unifying permitting with constituent service history (Salesforce)
- Virtual inspection capability for remote project assessment
- AI-generated form creation to accelerate jurisdiction configuration

### Underserved Areas / Opportunities
- **Natural-language eligibility assistant**: Most platforms require applicants to know which permit type they need before applying; conversational AI guidance remains thin
- **Predictive processing time estimates**: Applicants universally lack reliable estimates of how long their specific application will take given current workloads and complexity
- **Cross-jurisdiction interoperability**: No platform provides easy data exchange between adjacent jurisdictions sharing projects or contractors
- **Open data publishing**: Automated BLDS-compliant permit data publication for transparency and developer access is not standard
- **Analytics depth**: Advanced operational analytics (inspector productivity, bottleneck identification, cycle-time root-cause analysis) is absent from most mid-market platforms
- **Integrated BIM/IFC review**: Digital building permit review using IFC model data (as required by Singapore's CORENET X and emerging EU standards) is an emerging gap
- **Language accessibility**: Multi-language applicant portals and AI-assisted translation of permit requirements remain limited
- **Predictive compliance risk scoring**: AI assessment of application characteristics correlating with future code violations

### AI-Augmentation Candidates
- Plan review pre-screening: Rule-based code compliance checking is manual and time-consuming — ideal for AI automation
- Application completeness validation: Checklist-based document review is mechanical and well-suited to AI
- Inspection scheduling optimisation: Route and workload optimisation across inspector schedules is a classic ML problem
- Natural-language permit eligibility guidance: Conversational intake that maps plain-language project descriptions to required permits and fees
- Fee calculation verification: Complex, jurisdiction-specific fee formula application is error-prone manually but tractable for AI
- Anomaly detection for licence compliance: Pattern-based detection of lapsed or fraudulently obtained licences from operational data

---

## Legal & IP Summary

All major platforms in this category are commercial proprietary products with enterprise government licensing. No open-source permitting and licensing platform with production-ready capabilities was identified. The BLDS open data standard (permitdata.org) is the main open IP artefact — a collaborative data specification with contributions from Accela, Socrata, and others, published under an open standard. Open311 is an LGPL-related open standard for government service request APIs. CityGML 3.0 is an OGC/ISO open standard. No patent concerns affecting an open-source AI-native implementation were identified, provided the implementation does not reproduce proprietary form logic, fee calculation engines, or workflow rule sets from existing vendors.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Online permit and licence application intake with configurable form fields and conditional logic
- Workflow engine with step assignment, reviewer queues, status tracking, and SLA thresholds
- Fee calculation with configurable schedules and integrated online payment processing
- Constituent self-service portal: application submission, status tracking, document upload, and payment
- Inspection scheduling, assignment, and results recording (with mobile access)
- GIS parcel lookup using an open or pluggable GIS layer
- AI-powered completeness checker: validate uploaded documents against permit type requirements before human review
- Email/SMS automated notifications at each workflow milestone

**Should-have (v1.1)**
- AI natural-language eligibility assistant guiding applicants to the correct permit pathway
- Digital plan review with markup tooling (Bluebeam-compatible or native)
- Code enforcement case management with violation notice generation
- Business and contractor licence lifecycle management (application, renewal, suspension)
- Analytics dashboard: processing times, workload by department, application volume trends
- BLDS-compliant open permit data export API

**Nice-to-have (backlog)**
- AI plan review automation for code compliance checking (IBC/fire code rules)
- Predictive inspection scheduling optimisation (route and workload ML)
- BIM/IFC model intake for digital building permit review
- Multi-language applicant portal with AI-assisted translation
- Multi-jurisdiction data interoperability and shared parcel data federation
- Virtual inspection tooling with video review capability
