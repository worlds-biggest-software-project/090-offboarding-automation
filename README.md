# Offboarding Automation

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native platform for structured offboarding checklists, access revocation, and knowledge transfer when employees leave.

Offboarding Automation orchestrates every task that needs to happen when someone leaves an organisation: revoking SaaS access, finalising payroll and benefits, collecting documents, capturing institutional knowledge, and producing the audit trail compliance teams require. It is built for security-conscious engineering organisations and HR operations teams that today manage offboarding through spreadsheets, email threads, and ad hoc scripts.

---

## Why Offboarding Automation?

- **No credible open-source option exists.** Every major offboarding platform (Rippling, BetterCloud, Zluri, BambooHR, Offboard.ai) is fully proprietary; engineering-led organisations that self-manage their SaaS stack have no off-the-shelf alternative.
- **Incumbents force a choice between HR depth and IT depth.** BambooHR and HR Cloud cover HR workflow but require third-party tools for SaaS deprovisioning; BetterCloud and Zluri cover SaaS access but are not standalone HR platforms; only Rippling unifies both, and only by stacking expensive modules.
- **Shadow IT and personal OAuth grants go unrevoked.** No incumbent fully discovers apps IT does not officially provision, leaving former employees with lingering access — 50% of ex-employee accounts remain active beyond one day of departure (BetterCloud, 2026).
- **Knowledge transfer is the most-cited unmet need.** No tool comprehensively identifies what knowledge a departing employee holds and facilitates its capture before their last day.
- **Exit interview data is wasted.** Qualitative exit-survey data typically goes unread; AI synthesis of exit themes across cohorts is not available in any incumbent product.

---

## Key Features

### Offboarding Workflow Orchestration

- Configurable multi-department offboarding checklist with task assignment to HR, IT, Finance, and Facilities
- Departure-event triggers via webhook from common HRIS systems (BambooHR, Workday, Rippling, HR Cloud)
- Status tracker showing completion state of each step across departments
- Customisable workflows by role, department, tenure, and reason for departure

### SaaS Access Revocation

- Automated deprovisioning across pre-built connectors for Google Workspace, Microsoft 365, Slack, GitHub, Jira, AWS IAM, and Notion
- Shadow IT discovery using SSO log analysis to surface apps not officially provisioned by IT
- Group membership and access-level revocation, not just user removal
- Equipment return workflow with shipping-label generation and asset tracking

### Compliance and Audit

- Complete audit log of all access revocation and task completion events, exportable as compliance evidence
- Jurisdiction-aware checklist adaptation for GDPR data deletion, state-specific final pay rules, and notice periods
- Evidence aligned with ISO/IEC 27001 A.9, NIST SP 800-53 AC-2, SOC 2, and SOX/FINRA audit requirements
- Electronic document collection and signature for separation paperwork

### Knowledge Transfer and Exit Insights

- Exit interview collection with structured question templates
- AI-powered exit interview synthesis: aggregate themes across surveys to surface retention-risk patterns
- Knowledge transfer assistant prompting departing employees to document key processes, hand over project ownership, and identify successors
- Optional alumni record for potential rehire or contractor engagement

---

## AI-Native Advantage

AI is used to close the gaps incumbents leave open. Intelligent SaaS access discovery scans email headers, calendar invites, directory logs, and browser activity to build a complete map of every app the employee can reach — including personal OAuth grants and shadow IT — before the workflow runs. AI-driven knowledge extraction analyses communication patterns, document authorship, and project history to surface institutional knowledge that would otherwise leave with the person. Natural-language synthesis turns exit interviews into cohort-level retention insights, and offboarding workflows are personalised by tenure, role, access level, project involvement, and jurisdiction rather than a one-size-fits-all checklist.

---

## Tech Stack & Deployment

The project is designed to be self-hosted by security-conscious organisations, with a managed-cloud option available. Integration is connector-based: pre-built adapters for the major productivity, identity, and code-collaboration SaaS platforms; SSO integration with Okta and Azure AD for identity-linked app access tracking; webhook and REST API hooks for HRIS departure events and custom internal systems. MDM integration with Apple Business Manager and Microsoft Intune is targeted for device lock and wipe. The platform produces machine-readable audit evidence aligned with SOC 2, ISO/IEC 27001, and NIST SP 800-53 controls.

---

## Market Context

Offboarding sits within the HR technology market (~$35B globally in 2025) and the SaaS management market (~$8B in 2025, growing at 19% CAGR). Demand is security-driven: 20% of businesses report data breaches linked to former employees and 76% of IT leaders consider offboarding a significant security threat (BetterCloud, 2026). Incumbent pricing ranges from ~$5.40 PEPM (HR Cloud Essential) and ~$17 PEPM (BambooHR Pro) for HRIS-bundled offboarding, to $8–$15/user/month for IT-focused SaaS deprovisioning (BetterCloud, Zluri), with full HR+IT unified platforms (Rippling) starting at $8+/user/month before module stacking. Primary buyers are IT Security / InfoSec managers, HR Operations managers, General Counsel, and Finance Controllers.

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
