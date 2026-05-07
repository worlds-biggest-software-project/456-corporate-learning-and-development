# Standards & API Reference

> Project: Corporate Learning & Development Platform · Generated: 2026-05-07

---

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 19796-1:2005 — Information Technology for Learning, Education and Training: Quality Management and Quality Assurance**
- URL: https://www.iso.org/standard/33934.html
- Defines a reference framework and process model for the quality management of learning, education, and training (LET) technology products and services. Directly relevant to defining quality criteria for AI-generated content and learning pathway recommendations.

**ISO/IEC 40180:2017 — Quality for Learning, Education and Training**
- URL: https://www.iso.org/standard/64720.html
- Provides guidance on the fundamental concepts and principles for the quality of learning, education, and training, including eLearning content. Relevant to content quality scoring and curation model design.

**ISO/IEC 23988:2007 — A Code of Practice for the Use of Information Technology (IT) in the Delivery of Assessments**
- URL: https://www.iso.org/standard/41571.html
- Covers best practices for computerised assessment delivery, directly relevant to the adaptive assessment and compliance quiz modules.

**ISO/IEC 25010:2011 — Systems and Software Quality Requirements and Evaluation (SQuaRE)**
- URL: https://www.iso.org/standard/35733.html
- Defines the quality model for software product quality; useful as a benchmark for platform quality attributes (reliability, security, usability, maintainability).

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The principal standard for information security management. Essential for enterprise LMS deployments handling sensitive employee HR and performance data; certification is a common enterprise procurement requirement.

---

### W3C & IETF Standards

**W3C Verifiable Credentials Data Model 2.0**
- URL: https://www.w3.org/TR/vc-data-model-2.0/
- Defines a standard for expressing digital credentials (including learning badges and certifications) in a cryptographically verifiable format. Enables cross-platform skills portability and tamper-proof certification records — a key gap identified in the feature analysis.

**W3C ActivityStreams 2.0 / Activity Vocabulary**
- URL: https://www.w3.org/TR/activitystreams-core/
- Provides a JSON-LD vocabulary for describing social and activity data. Forms the conceptual foundation of xAPI (Experience API) statements; relevant for designing the learning event schema.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard for delegated authorisation used by every major LMS API (Docebo, Cornerstone, Degreed, Absorb). All platform API endpoints must implement OAuth 2.0 client credentials and authorisation code flows.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Defines the JWT format used for API authentication tokens. Used alongside OAuth 2.0 in all major LMS and HRIS API integrations.

**RFC 7643 / RFC 7644 — SCIM 2.0 (System for Cross-domain Identity Management)**
- URLs: https://datatracker.ietf.org/doc/html/rfc7643, https://datatracker.ietf.org/doc/html/rfc7644
- Defines a RESTful protocol for user provisioning and de-provisioning. The standard mechanism for syncing user accounts, roles, and groups between HRIS platforms (Workday, BambooHR, SAP SuccessFactors) and the learning platform. Essential for automated onboarding/offboarding workflows.

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Core HTTP semantics standard; the foundation for REST API design compliance.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the Link header for web resource relationships; relevant for pagination in REST API responses.

---

### eLearning Content & Tracking Standards

**SCORM 1.2 and SCORM 2004 (4th Edition) — Sharable Content Object Reference Model**
- Maintained by: ADL (Advanced Distributed Learning Initiative)
- URL: https://adlnet.gov/projects/scorm/
- The most widely deployed eLearning packaging and communication standard. SCORM 1.2 remains the dominant content format in corporate training despite its age; SCORM 2004 adds sequencing and navigation capabilities. Must be supported for backward compatibility with existing content libraries.

**xAPI (Experience API / Tin Can) v1.0.3**
- Maintained by: ADL Initiative
- URL: https://github.com/adlnet/xAPI-Spec / https://xapi.com
- A modern eLearning standard that records any learning experience (online, offline, mobile, simulation) as JSON statements to a Learning Record Store (LRS). Enables richer learning analytics than SCORM and is the data format underlying cmi5. Required for tracking informal learning, mobile learning, and live operational skill signals.

**cmi5 (xAPI Profile for LMS–Content Communication)**
- Maintained by: ADL Initiative (originally AICC)
- URL: https://xapi.com/cmi5/
- Extends xAPI with standardised launch, authorisation, session management, and reporting protocols between an LMS and xAPI-enabled content. The designated successor to SCORM for new content development; DoD-mandated for US government deployments. Adoption is accelerating and new content should target cmi5 compliance.

**IMS Global Learning Tools Interoperability (LTI) v1.3 / LTI Advantage**
- Maintained by: 1EdTech Consortium (formerly IMS Global)
- URL: https://www.imsglobal.org/spec/lti/v1p3
- Enables secure embedding of third-party learning tools within the platform (e.g., virtual labs, simulation tools, assessment engines) using OAuth 2.0 and JSON Web Token authentication. LTI Advantage adds Assignment & Grade Services (AGS) for result reporting back to the platform gradebook.

**IMS Global Open Badges v3.0**
- Maintained by: 1EdTech Consortium
- URL: https://www.imsglobal.org/spec/ob/v3p0/
- Defines a portable, verifiable digital credential format for skills, achievements, and certifications. Aligns with W3C Verifiable Credentials; enables learner-owned, cross-platform skills portability. Critical for the recommended backlog feature of cross-platform skills credential export.

**IMS Global Caliper Analytics v1.2**
- Maintained by: 1EdTech Consortium
- URL: https://www.imsglobal.org/spec/caliper/v1p2
- Defines a sensor API and event data model for measuring and communicating learning activity metrics. Complements xAPI for structured learning analytics pipeline design.

**IMS Global Common Cartridge v1.3**
- Maintained by: 1EdTech Consortium
- URL: https://www.imsglobal.org/cc/index.html
- A packaging standard for learning content interoperability across platforms, extending SCORM with richer metadata and modern web content support.

---

### Data Model & API Specifications

**IEEE 1484.12.1-2020 — Standard for Learning Object Metadata (LOM)**
- URL: https://www.ieeeltsc.org/working-groups/wg12LOM/lomDescription/
- Defines a metadata schema for describing learning objects (courses, modules, assessments). Relevant for the content catalogue data model and content discovery/recommendation engine.

**IEEE 1484.20.1 — Standard for Reusable Competency Definitions**
- URL: https://standards.ieee.org/ieee/1484.20.1/4012/
- Defines a data model for describing, referencing, and sharing competency and skill definitions across systems. The foundational standard for the skills taxonomy data model.

**HR Open Standards — Skills and Competency Data Models**
- URL: https://www.hropenstandards.org/
- Maintained by HR Open Standards Consortium (founded 1999); provides XML and JSON schemas for HR data exchange including competency frameworks, candidate assessments, and employee records. Directly relevant for HRIS integration data contracts.

**ESCO (European Skills, Competences, Qualifications and Occupations)**
- Maintained by: European Commission
- URL: https://esco.ec.europa.eu/en/use-esco/download
- An open, multilingual classification of European skills, competences, qualifications, and occupations with REST API access. Provides a freely licensed, comprehensive skills taxonomy as an alternative to proprietary vendor taxonomies. Recommended as the foundation for the open-source skills ontology.

**O*NET (Occupational Information Network)**
- Maintained by: US Department of Labor
- URL: https://www.onetcenter.org/developers.html
- A comprehensive database of occupational requirements and worker attributes, freely available via API. Contains skills, knowledge, abilities, and work activities for 900+ occupations. Complements ESCO for North American market coverage.

**OpenAPI Specification v3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The current standard for machine-readable REST API documentation. All platform API endpoints should be documented using OAS 3.1 to support automatic SDK generation, API testing tools, and third-party integrations.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OAuth 2.1 (draft)**
- URL: https://oauth.net/2/
- The standard for API authorisation. All LMS APIs in the market (Docebo, Cornerstone, Degreed, Absorb) use OAuth 2.0. The platform should implement both the authorisation code flow (for user-delegated access) and client credentials flow (for machine-to-machine HRIS/LRS integrations).

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0; the standard for SSO integration. Required for enterprise deployments integrating with identity providers (Okta, Azure AD, Google Workspace).

**SAML 2.0 (Security Assertion Markup Language)**
- URL: https://docs.oasis-open.org/security/saml/v2.0/
- The dominant SSO federation standard in enterprise HR environments, especially for SAP and Workday integrations. Must be supported alongside OpenID Connect for full enterprise SSO coverage.

**SCIM 2.0 (RFC 7643/7644)**
- See W3C & IETF section above. Essential for automated user lifecycle management from HRIS.

**OWASP Top 10 (2021)**
- URL: https://owasp.org/www-project-top-ten/
- The authoritative reference for web application security risks. Employee training and HR data are sensitive; the platform must be audited against OWASP Top 10 prior to enterprise deployment.

**GDPR (EU Regulation 2016/679)**
- URL: https://gdpr-info.eu/
- Applies to all European employee data, including training records, assessment results, and skill profiles. Requires lawful basis for processing, right to erasure (including learning history), data minimisation, and audit log retention. Article 39 specifically addresses staff training record-keeping obligations for DPOs.

**FedRAMP (Federal Risk and Authorization Management Program)**
- URL: https://www.fedramp.gov/
- US federal government cloud security authorisation programme. Cornerstone OnDemand's FedRAMP certification is a key competitive differentiator for the US public sector. Pursuing FedRAMP authorisation would open the government training market.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- Maintained by: Anthropic
- URL: https://modelcontextprotocol.io/
- An open protocol enabling AI systems (Claude, ChatGPT, Microsoft Copilot, etc.) to access external data sources and tools via a standardised server interface. Docebo has implemented an MCP Server exposing learning catalog, learner progress, and certification data to LLM tools. Implementing an MCP Server endpoint is a high-priority differentiator for positioning the platform as an AI-era learning infrastructure layer.

---

## Similar Products — Developer Documentation & APIs

### Docebo
- **Description:** AI-first enterprise LMS/LXP with agentic AI, enterprise knowledge management, and MCP Server integration. One of the most API-complete platforms in the market.
- **API Documentation:** https://developer.docebo.com/
- **API Reference / Browser:** https://help.docebo.com/hc/en-us/articles/23173336663826-Get-started-with-the-Docebo-API-browser
- **SDKs/Libraries:** No official SDK; third-party REST client libraries available via community
- **Developer Guide:** https://developer.docebo.com/docs/get-started-with-the-docebo-api-browser
- **Standards:** REST/JSON, OpenAPI, xAPI/LRS, LTI 1.3, SCORM, cmi5, OAuth 2.0, SAML 2.0, SCIM 2.0, MCP Server
- **Authentication:** OAuth 2.0 (client credentials and authorization code)

### Cornerstone OnDemand
- **Description:** Enterprise talent suite combining LMS, LXP, performance management, and succession planning. FedRAMP-certified. Dominant in large enterprise and regulated industries.
- **API Documentation:** https://csod.dev
- **Additional Docs:** https://help.csod.com/help/csod_0/Content/User/Edge/Overview_Topics_-_Edge/APIs_Overview.htm
- **SDKs/Libraries:** Third-party: https://github.com/digitregroup/cornerstone-client (Node.js)
- **Developer Guide:** https://csod.dev/guides/getting-started/
- **Standards:** REST/JSON, OData (Reporting API), xAPI, SCORM, LTI 1.3, OAuth 2.0, SAML 2.0, OpenID Connect, FedRAMP
- **Authentication:** OAuth 2.0 client credentials (client ID + secret provisioned by platform admin)

### Degreed
- **Description:** Skills-first LXP that aggregates learning across all sources and builds a unified skill record per employee. Strong API for skills data and content integration.
- **API Documentation:** https://developer.degreed.com/reference/overview
- **Integration Docs:** https://developer.degreed.com/docs/degreed-integrations
- **SDKs/Libraries:** No official SDK; REST API with xAPI endpoint support
- **Developer Guide:** https://developer.degreed.com/docs/getting-started-with-the-api
- **Standards:** REST/JSON, xAPI, SCORM, OAuth 2.0, SAML 2.0
- **Authentication:** OAuth 2.0 (API key + client credentials)

### 360Learning
- **Description:** Collaborative LMS/LXP that enables subject matter experts to author and iterate on courses rapidly. Strong HRIS integrations and AI authoring assistance.
- **API Documentation:** https://360learning.com/partner-marketplace/ (partner integration docs)
- **Webhooks:** Available for course completion and user events
- **Standards:** REST/JSON, xAPI, SCORM, LTI, OAuth 2.0, SAML 2.0, OpenID Connect
- **Authentication:** OAuth 2.0

### Canvas LMS (Instructure)
- **Description:** Widely adopted open-core LMS in higher education with a well-documented REST API and OpenAPI 3.0 specification — a useful reference implementation for LMS API design patterns.
- **API Documentation:** https://developerdocs.instructure.com/services/canvas
- **REST API Reference:** https://www.canvas.instructure.com/doc/api/
- **SDKs/Libraries:** Python: canvasapi (https://github.com/ucfopen/canvasapi); Ruby: canvas-lms (official); Node.js community libraries
- **Standards:** REST/JSON, OpenAPI 3.0, LTI 1.3 (LTI Advantage), SCORM, xAPI, OAuth 2.0, SAML 2.0, SCIM 2.0
- **Authentication:** OAuth 2.0; API token (developer key)

### Moodle
- **Description:** The world's most widely deployed open-source LMS (GPLv3). Self-hosted or Moodle.net SaaS. Extensive plugin ecosystem and REST Web Services API.
- **API Documentation:** https://moodledev.io/docs/apis/subsystems/external
- **GitHub:** https://github.com/moodle/moodle
- **SDKs/Libraries:** PHP (core); Python: moodlepy; JavaScript: moodle-rest (community)
- **Developer Guide:** https://moodledev.io/
- **Standards:** REST/JSON (Web Services), LTI 1.3, SCORM 1.2/2004, xAPI, IMS Common Cartridge, SAML 2.0, OAuth 2.0, WCAG 2.1 AA, GDPR
- **Authentication:** Token-based (user token or web service token); external API supports OAuth 2.0

### LinkedIn Learning API
- **Description:** Content delivery and progress tracking API for organisations integrating LinkedIn Learning courses into internal LMS/LXP platforms.
- **API Documentation:** https://learn.microsoft.com/en-us/linkedin/learning/
- **SDKs/Libraries:** No official SDK; REST/JSON via LinkedIn Partner Program membership
- **Developer Guide:** https://learn.microsoft.com/en-us/linkedin/learning/
- **Standards:** REST/JSON, xAPI (completion data), OAuth 2.0
- **Authentication:** OAuth 2.0 (LinkedIn Partner Program membership required)

### Workday Learning API
- **Description:** Learning management functionality embedded in the Workday HCM suite. API access shares authentication with the broader Workday platform.
- **API Documentation:** https://community.workday.com/sites/default/files/file-hosting/productionapi/index.html
- **SDKs/Libraries:** Workday Studio (proprietary integration builder); third-party connectors via Workday Marketplace
- **Standards:** REST/JSON, SOAP/XML (legacy), SCORM, xAPI, SAML 2.0, OAuth 2.0, SCIM 2.0
- **Authentication:** OAuth 2.0 (ISU — Integration System User credentials); SAML for SSO

### ESCO REST API (European Commission)
- **Description:** Free, multilingual REST API providing access to the ESCO skills and occupations taxonomy — the recommended open-source foundation for the platform skills ontology.
- **API Documentation:** https://esco.ec.europa.eu/en/use-esco/esco-api
- **Data Download:** https://esco.ec.europa.eu/en/use-esco/download
- **Standards:** REST/JSON, SPARQL endpoint available for linked data queries
- **Authentication:** No authentication required (open public API)

### O*NET Web Services (US Department of Labor)
- **Description:** REST API for the O*NET occupational database — comprehensive skills, knowledge, abilities, and work activities for 900+ US occupations. Complements ESCO for North American deployments.
- **API Documentation:** https://www.onetcenter.org/developers.html
- **Standards:** REST/JSON, XML
- **Authentication:** API key (free, self-service registration)

---

## Notes

- **Skills portability gap**: No current standard covers the full cross-platform portability of a learner's composite skill record (aggregated from multiple learning platforms). W3C Verifiable Credentials and IMS Open Badges v3.0 address credential portability for individual achievements, but a holistic "skills wallet" standard is still emerging. Watching the 1EdTech Consortium's work on Comprehensive Learner Records (CLR) is recommended.
- **MCP as emerging standard**: The Model Context Protocol is very new (Anthropic, 2024) but Docebo's early adoption (announced April 2026) signals it as a key integration target for AI-era L&D platforms. The MCP specification is open and should be implemented as part of the AI-native differentiation strategy.
- **cmi5 adoption trajectory**: US DoD institutional backing is driving content publisher adoption of cmi5. New content developed for the platform should target cmi5 as the primary standard, with SCORM 1.2 maintained for backward compatibility.
- **GDPR and employee data**: Training completion records, assessment scores, and skill profiles constitute personal data under GDPR. The platform data model must support right-to-erasure requests without corrupting aggregate analytics; anonymisation pipelines for reporting are a requirement for EU deployments.
