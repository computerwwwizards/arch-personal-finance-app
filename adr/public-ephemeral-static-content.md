---
status: "live"
initial_date: "2026-10-28"
---

# ADR: Adopt Cloudflare Pages for Ephemeral Environments of Static Public Pages

## Context and Problem Statement
Every pull request and feature branch targeting our public-facing static web properties requires an isolated, production-parity ephemeral preview environment for automated E2E testing, visual regression verification, and stakeholder review. Because these artifacts are strictly pre-rendered static assets (HTML, CSS, JS bundles, and static media) with zero server-side Node.js runtime requirements, provisioning infrastructure that bills per repository contributor seat, charges for CDN egress bandwidth, or bills per active staging environment introduces severe financial scaling bottlenecks. Furthermore, because ephemeral preview URLs are publicly routable, automated scrapers, AI crawlers, and Layer 7 DDoS floods can exhaust bandwidth allowances on traditional cloud platforms. We must select an edge hosting provider that automates ephemeral branch deployments, enforces perimeter bot mitigation and Zero Trust access gateways at the network edge, and decouples operational costs from engineering headcount and public network traffic.

## Decision Drivers
* **Zero Seat-Based Licensing Scaling:** Provisioning preview URLs on pull requests must not require purchasing a monthly platform seat ($20+/user/month) for every git committer or reviewer in the organization.
* **Uncapped Egress and Bandwidth Predictability:** Public preview URLs and static assets must be insulated from data transfer overages ($150–$400/TB) caused by automated E2E browser runners, heavy asset bundles, or unrestricted external traffic.
* **Edge Bot Mitigation and Perimeter WAF:** Publicly exposed ephemeral URLs must sit behind an edge proxy capable of dropping automated scrapers, headless browsers, and L3/L4/L7 DDoS traffic before requests trigger billing events or origin egress.
* **Native Zero Trust and API/AI Gateway Integration:** The platform must provide identity-aware access gateways (SSO/OIDC perimeter policies) and reverse-proxy routing without requiring dedicated NAT gateways, per-ACL fees, or external enterprise security appliances.
* **Unlimited Ephemeral Environment Concurrency:** The target platform must support arbitrary concurrent branch and PR deployments without hard staging environment caps or per-environment persistent storage fees.

## Considered Options
* **Cloudflare Pages**
* **Vercel (Pro / Enterprise)**
* **AWS Amplify Hosting (with AWS WAF and Bot Control)**
* **Azure Static Web Apps (with Azure Front Door and WAF)**

## Decision Outcome
Chosen option: "**Cloudflare Pages**," because it combines uncapped static bandwidth ($0.00/GB egress) and zero per-committer seat licensing with native edge perimeter security—including flat-rate L3/L4/L7 DDoS mitigation, Bot Fight Mode and Turnstile anti-bot challenges, Web Application Firewall (WAF) rulesets, and Cloudflare Zero Trust Access gateways (free for up to 50 users)—at a $0 to $20/month flat baseline.

### Benefits
* Good, because static asset egress across all 330+ global edge locations is completely free ($0.00/GB), eliminating financial exposure from high-frequency CI/CD test suites or traffic spikes on public preview links.
* Good, because edge bot mitigation (Bot Fight Mode, AI crawler blocking, and Turnstile CAPTCHA-free challenges) and DDoS protection execute natively at the global routing layer at $0 extra cost, stopping malicious automated traffic immediately.
* Good, because Cloudflare Zero Trust (Access and Gateway) natively wraps `*.<project>.pages.dev` preview subdomains behind identity-aware OIDC/SAML policies at $0/month for up to 50 seats (and $7/user/month thereafter), while allowing explicit service-token bypass rules for automated CI/CD E2E runners.
* Good, because git committers trigger automated PR preview deployments without incurring $20/user/month seat licensing charges.

### Trade-offs
* Bad, because native Cloudflare Pages build pipelines have soft concurrency limits (1 concurrent build on Free, 5 on Pro) and a 500–5,000 monthly build cap unless build execution is offloaded to external CI runners via `wrangler pages deploy`.
* Bad, because enabling Bot Fight Mode on preview domains can inadvertently block headless Chromium and Playwright CI runners unless custom WAF Skip rules or Cloudflare Access Service Tokens (`CF-Access-Client-Id` and `CF-Access-Client-Secret`) are explicitly configured for the CI pipeline.

### Confirmation
Compliance is enforced at the repository, infrastructure-as-code, and CI/CD pipeline levels:
1. **Repository and Pipeline Gate:** GitHub Actions workflows enforce deployment exclusively via `cloudflare/wrangler-action` using `wrangler pages deploy <dist-dir> --branch=<branch-name>`.
2. **Perimeter Security Gate:** Terraform and OpenTofu Cloudflare provider manifests (`cloudflare_pages_project`, `cloudflare_ruleset`, and `cloudflare_zero_trust_access_application`) enforce WAF custom rules, bot mitigation, and Access policies on preview subdomains.
3. **Configuration Lint Check:** Workspace root and package directories forbid `.vercel/`, `vercel.json`, `amplify.yml`, and `staticwebapp.config.json` configuration artifacts via pre-commit file-ban hooks.

## Pros and Cons of the Options

### Cloudflare Pages
Edge-first static hosting native to Cloudflare's global edge network, serving static build artifacts directly from edge locations behind Cloudflare's unified WAF, Bot Management, and Zero Trust gateway stack.

* Good, because static bandwidth and HTTP request volumes have no usage caps across all tiers ($0.00/GB egress and unlimited static requests).
* Good, because Bot Fight Mode (Free) and Super Bot Fight Mode (Pro at $20/month) inspect TLS fingerprints and user-agent heuristics to block automated scrapers and headless bots at the edge without per-request inspection fees.
* Good, because native gateway capabilities—including Cloudflare Zero Trust Access (50 free seats), Custom WAF Rules (5 free, 20 on Pro), and AI Gateway—are integrated into the same control plane without spinning up separate cloud load balancers.
* Neutral, because bypassing the monthly native build quota (500 builds/month on Free) requires executing the static build step inside GitHub Actions and pushing the compiled artifact directory via `wrangler pages deploy`.
* Bad, because any optional edge middleware (`_middleware.js`) must execute inside the Cloudflare `workerd` V8 isolate runtime rather than a full Node.js container, restricting native C++ addons.

### Vercel (Pro / Enterprise)
Frontend cloud platform optimized for Next.js and serverless framework abstractions, running edge and serverless compute over underlying public cloud infrastructure.

* Good, because it provides out-of-the-box PR comment bots and an integrated visual feedback toolbar overlaid directly onto preview deployments.
* Bad, because the Pro tier enforces a **$20/user/month** seat tax for every team member who authors a git commit that triggers a preview build.
* Bad, because bandwidth beyond the 1 TB Pro baseline is billed at **$150/TB ($0.15/GB)**, meaning unblocked bot scraping against public preview URLs directly translates into egress overage charges.
* Bad, because while basic L3/L4 DDoS mitigation is included, advanced managed WAF rulesets, enterprise bot management, and SSO/SAML preview protection gates require either paid add-ons or upgrading to custom Enterprise contracts ($2,000–$5,000+/month).

### AWS Amplify Hosting (with AWS WAF and Bot Control)
Managed AWS static hosting service backed by Amazon S3 object storage and Amazon CloudFront CDN distribution, paired with AWS WAF for perimeter protection.

* Good, because it eliminates per-seat licensing ($0/user/month), charging strictly for underlying resource consumption.
* Bad, because CDN data transfer out (egress) is billed at **$0.15/GB** beyond a 15 GB free tier ($150/TB), alongside **$0.023/GB/month** for retained ephemeral build artifacts.
* Bad, because attaching an edge gateway with anti-bot protection requires provisioning **AWS WAF** ($5.00/month per Web ACL + $1.00/month per rule + $0.60 per million requests) plus **AWS WAF Bot Control** ($10.00/month base fee + $1.00 per million requests inspected after 10 million), and **AWS Shield Advanced** costs **$3,000/month**.

### Azure Static Web Apps (with Azure Front Door and WAF)
Microsoft Azure's managed static hosting service with automated GitHub Actions and Azure DevOps integration and optional Azure Front Door edge security.

* Good, because the Standard plan charges per application resource (**$9/app/month**) rather than per git contributor seat.
* Bad, because bandwidth overages beyond the 100 GB monthly subscription allowance are billed at **$0.20/GB ($200/TB)**.
* Bad, because enabling edge gateway caching, WAF, and bot protection requires provisioning **Azure Front Door** (the "Enterprise-grade Edge" add-on costs **$17.52/app/month** minimum, while full Azure Front Door Premium with Bot Manager Rule Sets costs **$330/month base** plus per-request and egress charges).
* Bad, because the Standard tier enforces a hard limit of **10 concurrent staging or ephemeral environments** per app and caps compiled artifact storage at **0.50 GB per environment**.
