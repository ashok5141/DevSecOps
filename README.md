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

## Jenkins Pipeline Implementation

This section provides instructions on how to run the DevSecOps pipeline using Jenkins.

### Jenkins Setup

To run the Jenkins pipeline, you first need to set up a Jenkins instance. The provided `docker-compose.yml` file makes this process straightforward.

**Prerequisites:**
*   Docker
*   Docker Compose

**Steps:**

1.  **Start Jenkins:**
    Run the following command from the root of the project to start the Jenkins controller and agent:
    ```bash
    docker-compose up --build -d
    ```
    This will build the custom Jenkins agent image and start both containers in the background.

2.  **Get Initial Admin Password:**
    You will need the initial admin password to unlock Jenkins for the first time. You can get it from the logs of the `jenkins-controller` container:
    ```bash
    docker-compose logs jenkins-controller | grep -A 5 "Please use the following password to proceed to installation"
    ```
    Copy the password that is displayed in the terminal.

3.  **Access Jenkins:**
    Open your web browser and navigate to `http://localhost:8080/jenkins`. Paste the admin password and follow the instructions to complete the initial setup. It is recommended to install the suggested plugins.

4.  **Create the Pipeline Job:**
    *   On the Jenkins dashboard, click on **New Item**.
    *   Enter a name for your pipeline (e.g., `dvwa-devsecops-pipeline`).
    *   Select **Pipeline** and click **OK**.
    *   In the pipeline configuration page, scroll down to the **Pipeline** section.
    *   Select **Pipeline script from SCM** from the **Definition** dropdown.
    *   Select **Git** as the **SCM**.
    *   Enter the repository URL. If you are running this locally, you can use the path to your local repository (e.g., `/path/to/your/dvwa-repo`).
    *   Ensure the **Script Path** is set to `Jenkinsfile` (which is the default).
    *   Click **Save**.

Now you can run the pipeline by clicking on **Build Now** from the job's page. The pipeline will execute the stages defined in the `Jenkinsfile`.

### Pipeline Stages

The `Jenkinsfile` is designed to be self-contained and defines the following stages:

1.  **Checkout SCM:**
    *   **Purpose:** This first stage cleans the workspace and checks out the source code of *this* repository, which contains the Jenkins pipeline configuration.
    *   **Command:** `checkout scm`
    *   **Output:** The project files (`Jenkinsfile`, `docker-compose.yml`, etc.) are available in the Jenkins workspace.

2.  **Clone DVWA Source:**
    *   **Purpose:** To ensure the pipeline has the application code to work with, this stage clones the official DVWA repository from GitHub into a subdirectory (`dvwa-src`).
    *   **Command:** `git url: 'https://github.com/digininja/DVWA.git', ...`
    *   **Output:** The full DVWA source code is now present in the `dvwa-src/` directory within the Jenkins workspace.

3.  **Build DVWA Image:**
    *   **Purpose:** This stage builds the Docker image for the DVWA application using the `Dockerfile` from the cloned source code.
    *   **Command:** `docker.build(DVWA_IMAGE, './dvwa-src')`
    *   **Output:** A new Docker image is created with the tag `dvwa-app:<build-id>`.

4.  **SAST Scan (Semgrep):**
    *   **Purpose:** This stage performs a Static Application Security Test on the cloned PHP source code using Semgrep.
    *   **Command:** `semgrep --config="p/php" --error --verbose .` (run inside the `dvwa-src` directory).
    *   **Output:** The pipeline will fail if Semgrep finds any vulnerabilities. The output will list the findings.

5.  **SCA Scan (Composer Audit):**
    *   **Purpose:** This stage checks for known vulnerabilities in the PHP dependencies. It first installs the dependencies using `composer install` and then runs the audit.
    *   **Commands:** `composer install`, `composer audit` (run inside the `dvwa-src` directory).
    *   **Output:** The pipeline will fail if any vulnerable packages are found. The output will detail the vulnerabilities.

6.  **Container Scan (Trivy):**
    *   **Purpose:** This stage scans the newly built DVWA Docker image for operating system and package vulnerabilities.
    *   **Command:** `trivy image --exit-code 1 --severity HIGH,CRITICAL ${DVWA_IMAGE}`
    *   **Output:** The pipeline will fail if Trivy finds any `HIGH` or `CRITICAL` severity vulnerabilities. A table of findings will be printed to the console.

7.  **DAST Scan (OWASP ZAP):**
    *   **Purpose:** This stage performs a Dynamic Application Security Test against a running instance of the DVWA application.
    *   **Commands:**
        1.  `docker-compose up -d`: Starts the DVWA application using the `compose.yml` file from the cloned DVWA repository.
        2.  `docker run ... owasp/zap2docker-stable zap-baseline.py ...`: Runs the ZAP baseline scan against the application.
        3.  `docker-compose down`: Tears down the application after the scan.
    *   **Output:** A `zap-report.html` file is generated with the DAST scan results. This report is archived as a build artifact in Jenkins, so you can view it from the build's page.
