# CI/CD Pipeline Audit & Redesign Analysis

## 1. Executive Summary
The initial workflow configuration lacked explicit job dependencies, environment isolation controls, and proper artifact sharing. As a result, all jobs executed concurrently upon pipeline triggering. This audit documents the structural flaws identified across all six pipeline jobs and outlines the implemented resolution matrix.

---

## 2. Job-by-Job Failure Analysis & Fixes

### Job 1: `lint`
* **Intended Responsibility:** Validate code quality and enforce formatting standards across the codebase.
* **Original Issue:** Lacked a defined timeout limit, allowing hanging processes to consume CI runners indefinitely (up to 360 minutes).
* **Remediation:** Configured `timeout-minutes: 10` to guarantee rapid feedback and runner release.

### Job 2: `unit-tests`
* **Intended Responsibility:** Execute fast, isolated unit tests against component logic.
* **Original Issue:** Ran concurrently with `lint` at pipeline initialization, consuming execution budget even when syntax/linting errors existed.
* **Remediation:** Added `needs: lint` to enforce sequential gating, ensuring tests only execute on syntactically valid code. Configured `timeout-minutes: 15`.

### Job 3: `build`
* **Intended Responsibility:** Compile source code into deployable production assets inside the `dist/` directory.
* **Original Issue:** Ran in parallel without waiting for lint checks and failed to persist build outputs for downstream consumption.
* **Remediation:** Added `needs: lint` to run concurrently with `unit-tests`. Integrated `actions/upload-artifact@v4` to store `dist/` as `app-build`. Configured `timeout-minutes: 20`.

### Job 4: `integration-tests`
* **Intended Responsibility:** Validate application behavior against compiled production build artifacts.
* **Original Issue:** Executed before `build` completed, attempting to test non-existent build outputs on an isolated fresh virtual machine.
* **Remediation:** Added `needs: build` to establish dependency. Integrated `actions/download-artifact@v4` using artifact name `app-build` to extract compiled files to `dist/`. Configured `timeout-minutes: 30`.

### Job 5: `deploy-staging`
* **Intended Responsibility:** Deploy validated builds to the staging environment.
* **Original Issue:** Fired immediately on workflow trigger regardless of test results or source branch, risking unstable deployments.
* **Remediation:** Added `needs: [unit-tests, integration-tests]` to ensure complete test validation. Added `if: success() && github.ref == 'refs/heads/main'` to restrict deployment strictly to the `main` branch. Configured `timeout-minutes: 15`.

### Job 6: `deploy-production`
* **Intended Responsibility:** Promote verified staging builds to the production environment.
* **Original Issue:** Triggered in parallel with all other jobs, deploying unverified code from any branch directly to production.
* **Remediation:** Added `needs: deploy-staging` to enforce sequential deployment promotion. Restricted execution using `if: success() && github.ref == 'refs/heads/main'`. Configured `timeout-minutes: 15`.

---

## 3. Supplementary Orchestration Controls
* **Notification System (`notify`):** Added a post-pipeline execution job configured with `if: always()` to provide operational status updates regardless of upstream success, failure, or cancellation.