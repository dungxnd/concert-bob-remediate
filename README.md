# Concert + Bob Vulnerability Remediation

This repo demonstrates an automated CVE remediation workflow using Concert + Bob, featuring a vulnerable Java application that replicates the 2017 Equifax data breach scenario.

## 📦 Demo Application

### CVE-2017-5638 - Equifax Breach Scenario
**Location**: `app/credit-app/`

A vulnerable credit reporting application built with Apache Struts 2.3.31, demonstrating **CVE-2017-5638** - the critical Remote Code Execution vulnerability exploited in the 2017 Equifax data breach that compromised 147 million records.

- **Vulnerability**: Apache Struts 2.3.31 (CVSS 10.0 - Critical)
- **Attack Vector**: OGNL injection via Content-Type header
- **Impact**: Remote Code Execution (RCE)
- **Real-World Impact**: 2017 Equifax breach (147M records)
- **Documentation**: See [app/credit-app/README.md](app/credit-app/README.md)

---

## Workflow Overview

We showcase an automated workflow for vulnerability remediation using Concert + Bob

![Workflow Diagram](./workflow_diagram.png)

## Workflow Overview
This automated [workflow](./.github/workflows/bob-shell-v2.yml) orchestrates CVE remediation through four key stages:

1. **DevOps Phase** - Bob Shell researches the CVE using Concert API and external sources (NVD, GitHub Advisory), performs false positive detection, then creates a PR with package upgrades (if needed)
2. **DevOps Approval** - DevOps engineer reviews and approves the package changes on GitHub UI
3. **Unit Testing** - Automated tests run against the upgraded dependencies to verify compatibility
4. **Code Remediation** - If tests fail, Bob Shell automatically fixes application code to work with the new package versions
5. **Merge & Deploy** - Software Lead assigned to perform code review before changes are merged and deployed to production

The workflow uses Bob Shell in two specialized modes: `devops-concert-shell-v2` for CVE research with false positive detection and package upgrades, while using `advanced` mode for code fixes. Each stage includes automated PR comments, reviewer assignments, and label management to track progress.

## False Positive Detection

The workflow includes intelligent false positive detection to prevent unnecessary package upgrades and potential breaking changes. Before creating a remediation PR, Bob Shell analyzes the codebase to determine if the CVE actually affects your application.

### Detection Process

1. **Extract CVE Context** - Parse the vulnerability details including CVE ID, package name, vulnerable functions, attack vectors, and severity
2. **Search Codebase** - Find all locations where the vulnerable package is imported or used
3. **Analyze Function Calls** - Determine if specific vulnerable functions are actually called in the code
4. **Evaluate Usage Context** - Trace data flow to understand if user input reaches vulnerable code paths
5. **Assess Deployment** - Consider if the service is exposed to untrusted users and what security controls exist
6. **Make Determination** - Decide if the CVE is a false positive or true positive based on evidence
7. **Document Analysis** - Create comprehensive documentation with file paths, code snippets, and reasoning
8. **Take Action** - Either comment on the issue (false positive) or proceed with remediation (true positive)

### Common False Positive Scenarios

- **Transitive Dependency** - Package is a dependency of a dependency, never directly used
- **Unused Feature** - Vulnerable feature of the package is not used (e.g., Flask app doesn't use cookies but CVE affects cookie parsing)
- **Unreachable Code** - Vulnerable functions exist but code paths are unreachable
- **Internal Only** - Service is internal-only, not exposed to untrusted users
- **Security Controls** - Input validation, WAF, or other controls prevent exploitation
- **Disabled Feature** - Vulnerable feature is disabled in configuration

### False Positive Workflow

When a false positive is detected:

1. Bob Shell creates a detailed comment on the GitHub issue with:
   - Summary of why it's a false positive
   - Code analysis with file paths and line numbers
   - Evidence showing the vulnerable code is not reachable
   - Exploitability assessment
   - Recommendation (no action required)

2. The DevOps engineer is tagged for review: `@devops-engineer`

3. **No branch or PR is created** - avoiding unnecessary work and potential breaking changes

4. The DevOps engineer can review the analysis and either:
   - Agree and close the issue
   - Disagree and manually trigger remediation

### True Positive Workflow

When a true positive is confirmed:

1. Bob Shell proceeds with the normal CVE remediation workflow
2. Creates a remediation branch
3. Updates package manifests with the fixed version
4. Commits and pushes changes
5. Creates a pull request with detailed CVE information
6. Workflow continues to DevOps approval stage

### Safety First Approach

When the analysis is inconclusive or uncertain, the workflow defaults to **TRUE POSITIVE** and proceeds with remediation. This ensures that potential vulnerabilities are not missed, even if it means some unnecessary upgrades.

### Initial Setup:
#### Configure the following Github Action Secrets
`https://github.com/highorbit25/concert-bob-remediate/settings/secrets/actions`

```
BOBSHELL_API_KEY="XXX"

BOBSHELL_MCP_CONFIG="{sample in .bob/mcp.json}"

CONCERT_API_KEY="C_API_KEY XXX"
```


#### Configure the following Github Action Variables
`https://github.com/highorbit25/concert-bob-remediate/settings/variables/actions`

```
CONCERT_HOSTNAME="https://162.133.134.XXX:12443"

CONCERT_INSTANCE_ID="0000-0000-0000-0000"

# To disable telemetry
BOBSHELL_SETTINGS={ "security": { "ibmTelemetry": { "enabled": false } } }

DEVOPS_REVIEWERS=['devops-user']

INGESTION_JOB_ID=92073c05-xxxx-xxxx-8408-89ed1341a592

LEAD_REVIEWERS=['swe-lead-user']

SWE_REVIEWERS=['swe-user']
```

#### Configure `devops-approval` Environments for DevOps Approval

`https://github.com/highorbit25/concert-bob-remediate/settings/environments`

1. Create `devops-approval` environment
2. Add `Deployment protection rules` > `Required reviewers` > Select `devops-user`

#### Reset scenario
To simulate clearing of vulnerabilities for the credit-app, use `pom-reset.xml`
```
cp pom-reset.xml pom.xml
```

To re-introduce vulnerabilities for the credit-app, use `pom-vuln.xml`
```
cp pom-vuln.xml pom.xml
```

### Upload credit-app SBOM to Concert
The `Upload credit-app SBOM to Concert` workflow (`.github/workflows/upload-sbom-to-concert.yml`) runs on pushes to `main` that change `app/credit-app/pom.xml` or `app/credit-app/src/**`, and on manual dispatch. It:

1. Generates a CycloneDX 1.6 package SBOM with `cyclonedx-maven-plugin` (no change to `pom.xml`).
2. Generates a ConcertDef 2.0.0 application SBOM (`credit-app`, version read from the POM, code component = this repository and branch).
3. Scans the package SBOM with Trivy (`aquasecurity/setup-trivy` + `trivy sbom --scanners vuln --format cyclonedx`) and writes the CVE findings as a CycloneDX VDR file. Concert does not match package SBOMs against advisories on its own, so this scan is what produces the CVE findings (for example for Struts 2.3.31). The output is left unedited because Concert requires `code_scan` files to match the structure of its sample, which is a Trivy CycloneDX file. Findings never fail the job.
4. Uploads to `POST {CONCERT_HOSTNAME}/concert/ingestion/api/v1/upload_files`, in this order: `application_sbom`, `package_sbom` (metadata: `repo_url`, `repo_name`, `branch_name`, `application_name`, `application_version`; a new `serialNumber` is generated on every run because Concert rejects a repeated serialNumber/version pair) and `code_scan` (the Trivy scan; metadata: `scanner_name`, `repo_url`, `branch_name`, `application_name`, `application_version`). The `/concert` prefix can be changed with the variable `CONCERT_API_PREFIX` (`none` for no prefix).

`CONCERT_HOSTNAME` must be set as a repository **variable** (Settings > Secrets and variables > Actions > Variables) in the repository that runs the workflow, including forks.

It uses the existing `CONCERT_API_KEY` secret and `CONCERT_HOSTNAME` / `CONCERT_INSTANCE_ID` settings. TLS verification is skipped by default (self-signed certificate); set the variable `CONCERT_INSECURE_TLS=false` to enforce it. For OpenShift deployments the `CONCERT_API_KEY` value must start with `ZenApiKey` instead of `C_API_KEY`.

#### Let Concert open the GitHub issue that starts Bob Shell
The `Bob Shell V2` workflow (`.github/workflows/bob-shell-v2.yml`) starts when an issue receives the label `cve-remediation`. Concert can create that issue itself with an automation rule ([IBM docs](https://www.ibm.com/docs/en/SSQNYH_3.1.x/doc/automation_rule_cve_tracking.html)). Both parts are configured in the Concert UI:

1. **Connection**: Administration > Integrations > Connections. Integration type `GitHub`, with a personal access token that has write access to this repository's issues.
2. **Automation rule**: Administration > Integrations > Automation rules > Create automation rule.
   - Condition: `CVE tracking`, applied to the application `credit-app`.
   - Action: `Open a GitHub issue`, with this repository's organization and name.
   - Using this connection: the connection from step 1.
   - Assign to: a GitHub user with access to the repository.
   - Labels: `cve-remediation` (use the refresh icon to load existing labels, so create the label in the repository first).
   - Threshold: the minimum IBM risk score, CVSS score or priority that should trigger an issue.

Concert creates one issue per CVE group (CVEs on the same package are grouped) once the CVEs have been ingested for `credit-app` by the upload workflow above.

### Bob shell
For development purposes, Bob shell is currently installed on the Github Actions runner when the Github Action is triggered, followed by the configuration of Github MCP server. The custom 'devops-concert-shell-v2' mode is also configured in the `.bob` directory in the repository. 

For an actual production setup, a dedicated Github runner with Bob shell installed and settings pre-configured can be used.

## Watch it in action (Click through Demo)
Enable voiceover to hear the narration. [Link to Demo](https://demo-now.techzone.ibm.com/psl/fmx0mz7)
