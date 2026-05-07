# Corporate Learning & Development Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native platform that replaces static course catalogues with continuous skills intelligence, closing the gap between workforce capability and business strategy.

Corporate Learning & Development Platform is an open-source system that combines training needs analysis, AI-powered content curation, personalised learning pathways, and a skills intelligence layer. It is built for L&D teams, engineering leaders, and HR departments who need to map workforce skills against strategic capability targets and demonstrate measurable business impact from learning programmes.

---

## Why Corporate Learning & Development Platform?

- **Incumbents are locked behind enterprise pricing.** Platforms like Cornerstone OnDemand, SAP SuccessFactors Learning, and Workday Learning are only available as part of expensive HCM suite subscriptions, putting skills-driven learning out of reach for mid-market organisations.
- **Skills intelligence is siloed.** Degreed's skill inference lags when HRIS and performance data are not well-connected. SAP's 30,000-skill taxonomy is proprietary. No platform offers a portable, open skills record that works across vendors.
- **Course completion is a poor proxy for capability.** Most platforms still measure learning by SCORM completion rates and self-assessments rather than inferring skills from actual work artefacts and operational signals.
- **ROI attribution is unsolved.** No incumbent reliably correlates learning activity with downstream business KPIs (sales performance, retention, promotion velocity) without heavy custom data engineering.
- **Only 9% of companies use AI in training delivery** (Training Industry, 2026), representing a significant first-mover opportunity for a platform that embeds AI deeply into content curation, pathway generation, and skill gap detection.

---

## Key Features

### Skills Taxonomy & Gap Analysis

- Skills taxonomy builder with import support for ESCO and O*NET open datasets
- Role-to-skill mapping with personalised upskilling pathways
- Individual and team skill gap analysis dashboards
- Manager dashboard with team skill gap heatmap and recommended learning actions

### AI-Powered Content Curation & Pathways

- AI-generated learning pathway recommendations based on identified skill gaps
- Content quality scoring weighted by freshness, learner ratings, and source credibility
- Integration connectors for LinkedIn Learning and Coursera content libraries
- AI course authoring tool generating course structure and assessment questions from topic or document input

### LMS Core

- SCORM 1.2, xAPI, and cmi5 content hosting, enrolment, and completion tracking
- Compliance training module with automated assignment rules and audit trail export
- Certifications and progress tracking
- Mobile application (iOS and Android)

### Integrations & Interoperability

- HRIS user sync via SCIM 2.0 (Workday and BambooHR as initial targets)
- Slack and Microsoft Teams learning nudge notifications
- REST API with OpenAPI 3.1 specification and OAuth 2.0 authentication
- SSO via SAML 2.0 and OpenID Connect

### Analytics & ROI Reporting

- Basic ROI reporting linking completion rates to performance review cycle data
- Learning impact attribution connecting interventions to measurable performance changes
- Compliance question generation from policy documents
- Feedback loop detection: auto-flag learning pathways that did not improve targeted business metrics

---

## AI-Native Advantage

The platform replaces periodic surveys and manual curriculum design with continuous, AI-driven skill gap detection drawn from live operational signals -- CRM data, support tickets, code activity, and quality dashboards. Generative AI produces role-specific, contextually current learning assets and auto-flags stale content for update. A manager coaching assistant surfaces actionable suggestions on how to address specific team member skill gaps, moving beyond passive dashboards to active intervention. Learning pathway optimisation is driven by AI rather than manual L&D programme design, targeting measurable role-readiness outcomes.

---

## Tech Stack & Deployment

The platform targets self-hosted, cloud, and hybrid deployment models. It is built on open standards: xAPI and SCORM for content interoperability, SCIM 2.0 for user provisioning, LTI v1.3 for external tool embedding, and OAuth 2.0 / SAML 2.0 for authentication. The skills taxonomy layer is grounded in freely available open datasets (ESCO, O*NET) rather than proprietary ontologies. A REST API documented with OpenAPI 3.1 provides the integration surface. A backlog item includes an MCP Server endpoint for exposing the platform as a knowledge source inside enterprise LLM tools.

---

## Market Context

The corporate L&D market is dominated by commercial SaaS platforms spanning LMS, LXP, and skills intelligence categories. Enterprise incumbents (SAP SuccessFactors, Workday Learning, Cornerstone) bundle learning as part of broader HCM suites with significant licensing costs, while standalone platforms (Degreed, Docebo, 360Learning) charge per-user SaaS fees that scale steeply. Moodle is the only major open-source LMS but lacks built-in skills intelligence or AI-driven personalisation. With only 9% of companies currently using AI in training delivery (Training Industry, 2026), there is substantial room for an AI-native open-source alternative to capture demand from organisations that need skills intelligence without enterprise-tier pricing.

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
