# Standards & API Reference

> Project: Permitting & Licensing Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### Building and Construction Domain Standards

**International Building Code (IBC)**
- Organisation: International Code Council (ICC)
- URL: https://www.iccsafe.org/products-and-services/i-codes/2021-i-codes/ibc/
- Relevance: Model building code adopted by the majority of US jurisdictions. Drives permit type taxonomy, occupancy classifications, inspection checkpoint definitions, and code compliance rules that a permitting platform must model and reference.

**International Code Council (ICC) Digital Codes**
- Organisation: International Code Council
- URL: https://codes.iccsafe.org/
- Relevance: Digital publication of building, fire, plumbing, electrical, and mechanical codes. Permitting platforms need code references aligned to inspection checklists and plan review workflows. Machine-readable code content is an emerging resource for AI-assisted compliance checking.

**BLDS — Building and Land Development Specification**
- Organisation: Open data standards collaborative (Accela, Socrata, Buildingeye, Civic Insight, others)
- URL: https://permitdata.org/ · GitHub: https://github.com/open-data-standards/permitdata.org
- Relevance: Open data standard for sharing building and construction permit data issued by municipal governments. Defines required and optional fields including permits.csv, inspections.csv, and contractors.csv schemas. Adopted by San Diego County, Seattle, Tampa, Chattanooga, and others. An AI-native permitting platform should publish data in BLDS format to enable open-data transparency and third-party analytics.

**IFC — Industry Foundation Classes (ISO 16739)**
- Organisation: buildingSMART International / ISO
- URL: https://www.buildingsmart.org/standards/bsi-standards/industry-foundation-classes/
- Relevance: Open BIM data standard defining a neutral, vendor-agnostic file format for building information models. Increasingly required for digital building permit submission (Singapore CORENET X mandate; EU CHEK DBP project). An AI-native permitting platform should be capable of ingesting IFC files for automated plan review and compliance checking.

**BCF — BIM Collaboration Format**
- Organisation: buildingSMART International
- URL: https://www.buildingsmart.org/standards/bsi-standards/bim-collaboration-format-bcf/
- Relevance: Standard for communicating model-based issues between BIM applications. Relevant for digital plan review workflows where reviewers need to annotate 3D building models and share comments back to designers.

---

### Geospatial Standards

**GML — Geography Markup Language (ISO 19136 / OGC)**
- Organisation: Open Geospatial Consortium / ISO TC211
- URL: https://www.ogc.org/standards/gml/
- Relevance: XML-based encoding standard for geographic information including parcel boundaries, land use zones, and infrastructure layers. Permitting platforms relying on parcel data and zoning lookups consume GML or GML-derived formats from government GIS systems.

**CityGML 3.0 (OGC Standard)**
- Organisation: Open Geospatial Consortium
- URL: https://www.ogc.org/standards/citygml/ · Documentation: https://docs.ogc.org/is/21-006r2/21-006r2.html
- Relevance: Open standard for 3D city models representing terrain, buildings, infrastructure, and urban objects. Relevant for permitting platforms integrating with smart city and urban digital twin systems, and for AI-assisted zoning analysis incorporating 3D building context.

**INSPIRE Directive (EU)**
- Organisation: European Commission
- URL: https://inspire.ec.europa.eu/
- Relevance: European spatial data infrastructure directive establishing interoperability standards for geospatial data including cadastral parcels, land use, and buildings. Relevant for any permitting platform deployed in EU jurisdictions.

---

### Government Service & API Standards

**Open311**
- Organisation: Open311 community (civic technology open standard)
- URL: https://www.open311.org/
- Relevance: Standard API for government service requests and non-emergency reporting. Permits permitting status and code enforcement cases to be surfaced within broader 311 constituent service systems. Enables integration between permitting platforms and CRM tools like SeeClickFix.

**NEPA and Permitting Data and Technology Standard**
- Organisation: US Federal Permitting Improvement Steering Council (PISC)
- URL: https://permitting.innovation.gov/resources/data-standard/
- Relevance: Federal standard for environmental review and permitting data exchange, using JSON, YAML, SQL, and OpenAPI. Defines interoperability requirements for federal and state permitting systems and provides developer tools for implementers.

**OpenAPI Specification 3.1**
- Organisation: OpenAPI Initiative / Linux Foundation
- URL: https://spec.openapis.org/oas/v3.1.0
- Relevance: De facto standard for describing REST APIs. Government permitting platforms (Accela, OpenGov) publish API documentation using OpenAPI-compatible formats. An AI-native open-source platform should expose its APIs as an OpenAPI 3.1 specification.

**JSON Schema (Draft 2020-12)**
- Organisation: JSON Schema organisation / IETF
- URL: https://json-schema.org/
- Relevance: Standard for validating and describing JSON data structures. Used for defining permit application data models, fee schedule configurations, and workflow definitions in modern permitting platforms.

---

### Accessibility Standards

**ADA Title II (Americans with Disabilities Act)**
- Organisation: US Department of Justice
- URL: https://www.ada.gov/resources/web-guidance/
- Relevance: Requires state and local government digital services, including online permitting portals, to be accessible to people with disabilities. Title II final rule (effective 2026 for most local governments) mandates WCAG 2.1 Level AA conformance.

**Section 508 (Rehabilitation Act)**
- Organisation: US Access Board / GSA
- URL: https://www.section508.gov/
- Relevance: US federal accessibility mandate applicable to government technology procurement. Permitting platforms sold to federal and federally funded agencies must meet Section 508 / WCAG 2.1 Level AA requirements.

**WCAG 2.1 Level AA**
- Organisation: W3C
- URL: https://www.w3.org/TR/WCAG21/
- Relevance: Web Content Accessibility Guidelines providing the technical conformance criteria cited by ADA Title II and Section 508. Applicant-facing permit portals and staff interfaces must meet AA level criteria.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- Organisation: IETF
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Relevance: Standard authorisation framework for delegated API access. Used by Accela Construct API and OpenGov APIs for third-party application authentication. An open-source permitting platform should implement OAuth 2.0 for its API and constituent portal authentication.

**OpenID Connect 1.0**
- Organisation: OpenID Foundation
- URL: https://openid.net/connect/
- Relevance: Identity layer on top of OAuth 2.0 for federated authentication. Government platforms use OIDC for single sign-on with agency identity providers and Login.gov (the US federal identity service for constituent authentication).

**NIST SP 800-63-3 (Digital Identity Guidelines)**
- Organisation: National Institute of Standards and Technology
- URL: https://pages.nist.gov/800-63-3/sp800-63-3.html
- Relevance: Defines Identity Assurance Levels (IAL), Authenticator Assurance Levels (AAL), and Federation Assurance Levels (FAL) for government digital services. Permitting platforms handling contractor licences and business registrations should target IAL2/AAL2 for identity-sensitive transactions.

**NIST SP 800-53 (Security Controls)**
- Organisation: National Institute of Standards and Technology
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- Relevance: Comprehensive security control catalogue used for FISMA compliance. Required for permitting platforms deployed in federal or federally funded state/local government environments.

**FedRAMP**
- Organisation: US General Services Administration
- URL: https://www.fedramp.gov/
- Relevance: Federal cloud security authorisation programme. Commercial permitting SaaS platforms (Oracle, Accela) targeting federal contracts seek FedRAMP authorisation. An open-source platform targeting US government deployment should document its FedRAMP control implementation.

---

### Payment Standards

**PCI DSS (Payment Card Industry Data Security Standard)**
- Organisation: PCI Security Standards Council
- URL: https://www.pcisecuritystandards.org/
- Relevance: Required for any permitting platform that processes, stores, or transmits cardholder data for permit fee payments. Most platforms achieve compliance through integration with PCI-compliant payment processors rather than direct card data handling.

---

## Similar Products — Developer Documentation & APIs

### Accela Civic Platform

- **Description:** Enterprise government permitting, licensing, and land management platform used by 600+ agencies worldwide. Provides a rich REST API for building third-party applications on top of the Civic Platform.
- **API Documentation:** https://developer.accela.com/
- **API Reference:** https://developer.accela.com/docs/api_reference/api-index.html
- **Developer Guide:** https://developer.accela.com/docs/construct-gettingStarted.html
- **SDKs/Libraries:** iOS, Android, Windows 8 mobile SDKs; web application samples available via developer portal
- **Standards:** REST/JSON; standard HTTP methods (GET, PUT, POST, DELETE); follows W3C REST guidelines
- **Authentication:** OAuth 2.0 (permission scopes defined at https://developer.accela.com/docs/construct-permissionScopes.html)

---

### OpenGov Permitting & Licensing

- **Description:** Cloud-native government permitting and licensing platform with no-code form builder and workflow automation. Provides APIs for integrating permitting data with third-party systems.
- **API Documentation:** https://developer.opengov.com/catalog/public-service-platform-permitting-and-licensing
- **Developer Portal:** https://developer.opengov.com/
- **API Overview:** https://developer.opengov.com/docs/overview
- **Quickstart Guide:** https://developer.opengov.com/docs/quickstart
- **Standards:** REST/JSON; OpenAPI-compatible documentation
- **Authentication:** API Key (https://developer.opengov.com/docs/app-management/api-key)

---

### Tyler Technologies Enterprise Permitting & Licensing

- **Description:** Market-leading government permitting, licensing, and asset management platform with deep Esri GIS integration, used by 11,000+ government clients.
- **API Documentation:** https://www.tylertech.com/products/enterprise-erp/api-catalog (Tyler API Catalog)
- **Integration Documentation:** https://tylertech.accessgov.com/docs/Forms/Page/docs/api/0
- **Partner Integration Docs:** https://www.civicplus.help/seeclickfix/docs/seeclickfix-connect-for-energov (SeeClickFix/EnerGov integration example)
- **Standards:** REST API; requires EnerGov version 2019.3+ for current integration support
- **Authentication:** Agency-specific credentials; REST-based service calls
- **Notes:** Full API documentation requires client credentials; public documentation is limited

---

### Oracle Permitting and Licensing

- **Description:** Enterprise cloud permitting, licensing, and code enforcement platform for state and large local governments. Part of Oracle Fusion Cloud Public Sector suite.
- **API Documentation:** https://docs.oracle.com/en/cloud/saas/public-sector-compliance-regulation-common/26a/permi/index.html
- **Developer Portal:** https://docs.oracle.com/en/cloud/saas/public-sector-compliance-regulation-common/26b/index.html
- **What's New (26B):** https://docs.oracle.com/en/cloud/saas/readiness/public-sector/26b/pscd26b/26B-pscd-wn-f43558.htm
- **Standards:** REST APIs; Oracle Integration Cloud for connectors
- **Authentication:** Oracle Identity Cloud Service (IDCS); OAuth 2.0

---

### Salesforce Public Sector Solutions (Permitting)

- **Description:** Permitting and licence management built on Salesforce CRM, providing constituent portals, smart dynamic forms, workflow automation, and analytics.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/
- **Public Sector Developer Docs:** https://help.salesforce.com/s/articleView?id=ind.psc_concept_admin_licensingandpermitting.htm
- **SDKs/Libraries:** Salesforce SDKs for JavaScript, Python, Java, .NET, PHP via https://developer.salesforce.com/tools/
- **Developer Guide:** https://developer.salesforce.com/developer-centers/public-sector
- **Standards:** REST and SOAP APIs; Bulk API 2.0; GraphQL API; OpenAPI-compatible specification available
- **Authentication:** OAuth 2.0; Connected Apps; JWT Bearer Flow for server-to-server integration

---

### Clariti (CivCheck AI Plan Review)

- **Description:** Government permitting platform with AI plan review capability via CivCheck acquisition. Provides AI intake validation, AI copilots for code compliance, and planned API integration with third-party permitting platforms.
- **Product Overview:** https://www.claritisoftware.com/products/civcheck-ai-plan-review-software
- **Integration Announcement:** https://www.claritisoftware.com/blog/clariti-eplansoft-announce-permitting-technology-integration
- **Standards:** Integration with e-PlanSoft and Bluebeam for plan review; REST API integration planned for spring 2026
- **Authentication:** Not publicly documented; agency SSO assumed
- **Notes:** Third-party API access is in active development; documentation is not yet publicly available

---

### PermitFlow

- **Description:** Contractor-side construction permitting automation platform covering jurisdiction research, permit preparation, filing, and tracking across 1,500+ US municipalities.
- **Product Overview:** https://www.permitflow.com/product
- **API/Integration:** Integrates with CRM platforms via connector; no public developer API documented
- **Standards:** REST integrations with CRM and project management tools; proprietary AHJ database
- **Authentication:** Not publicly documented
- **Notes:** PermitFlow is a SaaS product optimised for contractor workflows rather than government-side administration; no public API is documented

---

### US Federal Permitting Innovation Center (Developer Tools)

- **Description:** US federal initiative providing open developer tools, data standards, and resources for environmental review and permitting modernisation.
- **Developer Resources:** https://permitting.innovation.gov/resources/developer-tools/
- **Data Standard:** https://permitting.innovation.gov/resources/data-standard/
- **Standards:** JSON, YAML, SQL, and OpenAPI specification for NEPA permitting data exchange
- **Authentication:** Open public resources; agency-specific integrations use OAuth 2.0

---

## Notes

**Emerging area — BIM/IFC-based digital building permits:** Singapore's CORENET X initiative and the EU CHEK Horizon Europe project (concluded September 2025) represent the leading edge of BIM-based digital permitting where IFC model data is submitted in lieu of 2D drawings. The Digital Building Permit Conference series (DBP26, Munich, November 2026) is the primary global forum tracking this evolution. An AI-native platform positioning for this direction should monitor OGC/buildingSMART joint working groups on IFC-for-permitting specifications.

**Login.gov for constituent identity:** US federal and state government permitting portals are increasingly adopting Login.gov (https://login.gov/) as a shared identity provider, implementing OIDC and NIST SP 800-63-3 IAL2 standards. An open-source platform should support Login.gov integration to align with federal constituent identity strategy.

**BLDS adoption gap:** Despite the BLDS standard existing since 2014, adoption among US municipalities remains limited and uneven. An open-source platform that defaults to BLDS-compliant permit data publishing would create meaningful differentiated value in the government open-data ecosystem.
