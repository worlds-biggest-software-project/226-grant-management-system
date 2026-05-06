# Grant Management System

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for grant discovery, application, reporting, and compliance tracking across nonprofits, foundations, and government agencies.

Grant Management System is a candidate open-source platform that supports the full grant lifecycle — from prospecting and proposal drafting through review, award, post-award compliance, and reporting. It targets grantseekers (nonprofits, universities) and grantmakers (foundations, community funders, government agencies) who today rely on costly, vertically-fragmented commercial SaaS.

---

## Why Grant Management System?

- Incumbent platforms are fragmented by buyer type: Instrumentl skews to grantseekers, Fluxx and Foundant to grantmakers, Euna and eCivis to government — leaving organisations operating in multiple roles to stitch together several systems.
- Enterprise government-grade platforms (Euna, eCivis) run $50,000–$300,000/year, pricing smaller agencies and nonprofits out of compliance-grade tooling.
- Federal compliance is a moving target: FY2026 Uniform Guidance updates (single audit threshold raised to $1M, de minimis indirect rate raised to 15%) require platform changes that proprietary vendors release on their own timelines.
- Grant prospecting on most platforms still relies on keyword search rather than ML matching against an organisation's mission and programmatic focus.
- Subrecipient monitoring, automated impact narrative generation, and real-time regulatory change alerts remain underserved across the incumbent landscape.

---

## Key Features

### Application & Intake

- Online grant application submission with customisable form builder
- Applicant portal with application status tracking
- Email notifications for applicants and reviewers
- Document management and uploading

### Review & Award

- Review workflow with approval routing
- Multi-stage blind review workflows
- Custom scoring rubrics and evaluation criteria
- Award notification and tracking
- Audit trails for all decisions and communications

### Financial & Compliance

- Budget tracking with line-item detail
- Financial reporting with budget variance analysis
- Federal compliance reporting (FFATA, Data Act format)
- OMB 2 CFR Part 200 compliance checklist
- Subrecipient tracking and monitoring

### Reporting & Integrations

- Reporting with basic analytics and data export
- REST API for integrations
- Integration with accounting software
- Grant prospecting tools (opportunity database)
- Grantee CRM for post-award relationship tracking

---

## AI-Native Advantage

AI is applied where it removes the highest-cost manual work in grant operations: ML-powered prospecting that matches an organisation's mission against grant databases (Grants.gov, foundation directories) with higher relevance than keyword search, LLM-assisted first-draft proposal generation from project descriptions and budget data, and AI agents that continuously monitor expenditure against budget categories and generate OMB 2 CFR Part 200-compliant documentation. Generative AI also transforms structured outcome data into grantmaker-ready narrative reports, and ML risk scoring assesses subrecipient financial health and audit history to drive risk-based monitoring plans.

---

## Tech Stack & Deployment

The platform is expected to expose REST APIs for integrations, with connectors for Grants.gov, USASpending.gov / Data Act reporting, SAM.gov, and UEI (Unique Entity Identifier) tracking. XBRL and Data Act compliance require standardised, machine-readable reporting outputs. Cloud-based deployment is the dominant delivery model in this market (over 60% share); a self-hostable option would differentiate against enterprise SaaS incumbents.

---

## Market Context

The grant management software market was valued at approximately $3.22 billion in 2026 and is projected to reach $4.98 billion by 2030 at an 11.5% CAGR, with longer-range forecasts of $8.09 billion by 2035 (Precedence Research; National Law Review). Pricing ranges from ~$179/month for small nonprofit tools (Instrumentl) up to $50,000–$300,000/year for enterprise government platforms (Euna, eCivis). Primary buyers are nonprofit grant writers, community foundations, government agencies administering federal and state grants, universities managing research grant compliance, and corporate foundations.

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
