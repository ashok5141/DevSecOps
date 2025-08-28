# DevSecOps Pipeline for DVWA

This document outlines a plan for a DevSecOps pipeline for the Damn Vulnerable Web Application (DVWA). The goal of this pipeline is not to fix the vulnerabilities within DVWA, but to implement a security-focused workflow that can detect and report on these vulnerabilities. This serves as a practical example of how to integrate security into a CI/CD process.

The pipeline is designed to be automated and will be triggered on every code commit. It consists of the following stages:

1.  [Static Application Security Testing (SAST)](#static-application-security-testing-sast)
2.  [Dependency Scanning](#dependency-scanning)
3.  [Dynamic Application Security Testing (DAST)](#dynamic-application-security-testing-dast)
4.  [Container Security](#container-security)

---

## Static Application Security Testing (SAST)

**Tool:** [Semgrep](https://semgrep.dev/)

**Description:**
Semgrep is a fast, open-source, polyglot static analysis tool. It will be used to scan the PHP source code for security vulnerabilities. Semgrep's community-driven ruleset is excellent for identifying common security flaws, such as those intentionally included in DVWA.

**Implementation:**
In a CI/CD job, we will run Semgrep against the entire codebase. The command would look something like this:
```bash
semgrep --config="p/php" .
```
This command uses a curated ruleset for PHP (`p/php`) to scan the current directory. The pipeline would be configured to fail if any high-severity issues are found.

---

## Dependency Scanning

**Tool:** [Composer Audit](https://getcomposer.org/doc/08-dependencies.md#auditing-your-dependencies-for-security-vulnerabilities)

**Description:**
`composer audit` is the official command-line tool from Composer for checking for known vulnerabilities in your project's dependencies. Since DVWA uses Composer to manage its PHP packages, this is the most direct and effective way to ensure that the third-party libraries are secure.

**Implementation:**
A step in the pipeline will be dedicated to running the Composer audit. This is a simple, one-line command:
```bash
composer audit
```
The job will fail if any vulnerabilities are found in the dependencies, preventing insecure packages from being used.

---

## Dynamic Application Security Testing (DAST)

**Tool:** [OWASP ZAP (Zed Attack Proxy)](https://www.zaproxy.org/)

**Description:**
OWASP ZAP is a widely-used, open-source web application security scanner. It will be used to perform a dynamic scan of the running DVWA application. This will help identify vulnerabilities that can only be found at runtime, such as misconfigurations or certain types of injection flaws.

**Implementation:**
For a CI/CD pipeline, a ZAP baseline scan is a good starting point. It's a fast, passive scan that doesn't perform any attacks.
1.  The DVWA application will be started using Docker Compose.
2.  The ZAP Docker container will then be run to perform a baseline scan against the DVWA container.
```bash
docker run -v $(pwd):/zap/wrk/:rw -t owasp/zap2docker-stable zap-baseline.py \
    -t http://<dvwa-container-ip>:4280/ -g gen.conf -r report.html
```
The pipeline would be configured to fail based on the severity of the alerts in the ZAP report.

---

## Container Security

**Tool:** [Trivy](https://github.com/aquasecurity/trivy)

**Description:**
Trivy is a simple, fast, and comprehensive open-source vulnerability scanner for container images. Since DVWA is designed to be run in a Docker container, it's crucial to scan the container image for known vulnerabilities in the operating system packages and other dependencies.

**Implementation:**
After building the DVWA Docker image, Trivy will be used to scan it. This can be done with a single command:
```bash
trivy image <your-dvwa-image-name>
```
The pipeline will be configured to fail if Trivy detects any high or critical severity vulnerabilities in the Docker image. This ensures that the production environment is not exposed to known exploits.

---

## GitHub Actions DevSecOps Pipeline

This project includes a fully automated DevSecOps pipeline using GitHub Actions. The pipeline is defined in the `.github/workflows/security.yml` file.

### How it Works

*   **Trigger:** The pipeline is automatically triggered on every `push` or `pull_request` to the `main` or `master` branches.
*   **Execution:** It runs on a GitHub-hosted `ubuntu-latest` runner. You can view the progress and results of each run in the "Actions" tab of this GitHub repository.

### Pipeline Steps

The pipeline consists of a single job, `security-pipeline`, which executes the following steps in order:

1.  **Checkout This Repository:**
    *   **Purpose:** Checks out the source code of this repository, so the workflow can access the DVWA source code that is cloned in the next step.
    *   **Action:** `actions/checkout@v3`

2.  **Clone DVWA Source Code:**
    *   **Purpose:** Clones the official DVWA repository from GitHub into a `dvwa-src` subdirectory. This provides the application source code for the subsequent build and scan steps.
    *   **Action:** `actions/checkout@v3` with `repository: 'digininja/DVWA'`

3.  **Build DVWA Docker Image:**
    *   **Purpose:** Builds the Docker image for the DVWA application using the `Dockerfile` from the cloned source.
    *   **Command:** `docker build ...`
    *   **Output:** A new Docker image tagged with the Git SHA of the commit.

4.  **SAST Scan (Semgrep):**
    *   **Purpose:** Performs a Static Application Security Test on the cloned PHP source code.
    *   **Action:** `semgrep/semgrep-action@v1`
    *   **Output:** The workflow will fail if any vulnerabilities are found. Findings are listed in the step's log, and can also be viewed in the "Security" tab under "Code scanning alerts".

5.  **Set up PHP:**
    *   **Purpose:** Installs PHP, which is required for the Composer dependency audit.
    *   **Action:** `shivammathur/setup-php@v2`

6.  **SCA Scan (Composer Audit):**
    *   **Purpose:** Performs a Software Composition Analysis by checking for known vulnerabilities in the PHP dependencies.
    *   **Commands:** `composer install`, `composer audit`
    *   **Output:** The workflow will fail if any vulnerable packages are found.

7.  **Container Scan (Trivy):**
    *   **Purpose:** Scans the newly built DVWA Docker image for vulnerabilities in the OS packages and other libraries.
    *   **Action:** `aquasecurity/trivy-action@master`
    *   **Output:** The workflow will fail if any `HIGH` or `CRITICAL` severity vulnerabilities are found. A table of findings is printed in the step's log.

8.  **DAST Scan (OWASP ZAP):**
    *   **Purpose:** Performs a Dynamic Application Security Test against a running instance of the DVWA application.
    *   **Commands:**
        1.  `docker-compose up -d`: Starts the DVWA application using its `compose.yml`.
        2.  `docker run ... owasp/zap2docker-stable zap-baseline.py ...`: Runs the ZAP baseline scan.
        3.  `docker-compose down`: Tears down the application.
    *   **Output:** A `zap-report.html` file is generated with the scan results.

9.  **Upload DAST Report:**
    *   **Purpose:** Uploads the ZAP report as a build artifact.
    *   **Action:** `actions/upload-artifact@v3`
    *   **Output:** The `zap-dast-report.zip` will be available for download in the "Summary" section of the workflow run.
