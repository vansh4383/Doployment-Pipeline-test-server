# GoloBiz Backend – CI/CD Pipeline Automation

Automated CI/CD pipelines for GoloBiz backend applications using **Jenkins, GitLab, PM2, Slack, and Email Notifications**.

### Key Features
- Automated builds triggered by GitLab push events.
- Branch-specific deployment from `test/server`.
- Automated dependency installation using `npm ci`.
- TypeScript application build and validation.
- Release management with build-number-based versioning.
- Candidate deployment on isolated testing ports.
- Automated smoke testing before deployment.
- Production deployment after successful validation.
- PM2 process management.
- Slack notifications for successful and failed deployments.
- Email notifications to developers and testers.
- Previous production process remains active if the pipeline fails before switching.

### Configured Applications

| Application | Testing Port | Production Port |
|---|---:|---:|
| Customer API | 5002 | 4002 |
| Business Owner API | 4000 | 5001 |

### Technology Stack
Jenkins · GitLab · Node.js · TypeScript · PM2 · Linux · Slack Webhooks · Email Extension Plugin

**Objective:** Establish a reliable, automated deployment workflow that reduces manual deployment effort and enables developers and QA teams to validate new builds efficiently.
