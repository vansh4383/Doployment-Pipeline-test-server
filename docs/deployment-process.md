Deployment Process
1. Purpose and scope
This document explains the Jenkins CI/CD workflow for the Golobiz Customer API and Business Owner API. The pipeline builds a candidate release, tests it over HTTP, and then updates the production PM2 process.

The two APIs use separate source repositories and different deployment paths/ports. Run and troubleshoot them independently.
2. Service-specific values
Setting	Customer API	Business Owner API
GitLab repository	golobiz-backend-customer-app-api	golobiz-backend-business-owner-api
Server deployment directory	/opt/pipeline-pro-customer	/opt/pipeline-pro
Candidate port	5002	5001
Production port	4002	4000
Smoke route	/v1/categories	/v1/health (verify)


Do not copy the Customer API port or endpoint into the Business Owner pipeline, or vice versa.
3. Before triggering a deployment
Confirm the following:
- The intended change is pushed to the correct GitLab repository and branch.
- The Jenkins job is configured for the correct service and branch.
- Jenkins can access GitLab using the configured Git credential (the pipeline notes refer to credential ID access -token; verify the exact ID in Jenkins).
- The server has the required Node.js/npm and PM2 versions.
- The deployment directory is writable by the deployment account.
- The service's runtime .env file exists at the path expected by the Jenkinsfile.
- Required database and external-service configuration is available.
- Candidate and production ports are not occupied by unrelated processes.
- Slack credential slack-webhook and Jenkins Email Extension settings are configured if notifications are expected.
Never put secret values into Jenkinsfile source code or commit them to Git.
4. Pipeline workflow
Step 1 — Trigger and validate
A pipeline run starts through the configured Jenkins job trigger. Depending on job configuration, this may be a webhook or a manual build. The Validate stage checks required inputs and configuration.

If the job is not starting automatically, inspect the Jenkins job's build triggers and GitLab webhook configuration rather than assuming the Jenkinsfile alone enables webhooks.
Step 2 — Checkout source
Jenkins checks out the selected GitLab repository and branch using the configured credential.

Check the build log for:
- Correct repository URL.
- Correct branch or commit.
- Successful authentication.
- Successful checkout of the expected commit.
Do not paste personal access tokens into the repository URL or console log.
Step 3 — Install dependencies
The pipeline runs:
npm ci
npm ci installs dependencies from the lockfile and is appropriate for repeatable CI builds. If it fails, check that package-lock.json is committed and compatible with package.json, and review the first npm error in the Jenkins log.
Step 4 — Build the application
The pipeline runs:
npm run build
This executes the build script defined by the repository's package.json. For TypeScript services, this commonly compiles TypeScript into the configured output directory, but the actual output path must be confirmed in the project configuration.

A successful build is not proof that the application can connect to its database or external services. Those checks happen at runtime.
Step 5 — Prepare a release
The pipeline copies the required application files into a release directory on the server and makes the runtime environment configuration available to that release.

The .env file is a server-side secret-bearing configuration file. It must not be copied into the Git repository or printed in logs. Confirm the source and destination paths in the Jenkinsfile before manually copying or changing permissions.
Step 6 — Start the candidate process
Jenkins starts or restarts a candidate PM2 process using the candidate port:
- Customer API: 5002
- Business Owner API: 5001
The candidate process is intended to run the new release without immediately replacing the production process.

Inspect candidate startup with:
pm2 list
pm2 logs --lines 100
Use the actual candidate process name from the Jenkinsfile/PM2 list.
Step 7 — Run the smoke test
The pipeline sends an HTTP request to the candidate service and retries according to the Jenkinsfile's retry logic.

Recorded endpoints:
Customer API:       http://127.0.0.1:5002/v1/categories
Business Owner API: http://127.0.0.1:5001/v1/health
The Business Owner /v1/health route must be verified against the current application. A route that does not exist or requires authentication will cause a false deployment failure.

Manual checks from the server:
curl -i --max-time 10 http://127.0.0.1:5002/v1/categories
curl -i --max-time 10 http://127.0.0.1:5001/v1/health
Run only the command for the service being deployed. A smoke test should verify a meaningful successful response, not merely that a TCP port is open. Check the expected HTTP status and response body logic in the Jenkinsfile.

If the smoke test fails, the pipeline should stop before the production update. Inspect candidate logs and runtime configuration before retrying.
Step 8 — Update production
After the candidate smoke check passes, the pipeline updates the production PM2 process:
- Customer API production port: 4002
- Business Owner API production port: 4000
The pipeline then updates the current release symlink and runs:
pm2 save
pm2 save saves the current PM2 process list. It does not prove that the service is healthy or that PM2 startup on reboot is configured correctly.
Step 9 — Verify production
After deployment, check:
pm2 list
pm2 logs --lines 100
Then call the production endpoint:
# Customer API
curl -i --max-time 10 http://127.0.0.1:4002/v1/categories

# Business Owner API — confirm the route first
curl -i --max-time 10 http://127.0.0.1:4000/v1/health
Confirm that:
- The intended production process is online.
- The production endpoint returns the expected status.
- Logs do not show repeated crashes, database connection failures, or missing environment variables.
- The active current symlink points to the intended release.
5. Notifications
The pipeline notes identify two notification mechanisms:
- Slack webhook credential: slack-webhook
- Jenkins Email Extension: emailext
Notifications should communicate the service, build result, and build/job reference. Do not include secrets, full environment files, or sensitive log output.

Notification delivery depends on Jenkins credential availability and global email/Slack configuration. A notification failure should be distinguished from an application deployment failure.
6. Failure and rollback behavior
The recorded Jenkinsfiles include rollback-related logic, but this should not be described as guaranteed zero-downtime deployment. The current process updates PM2 and the release pointer; service interruption is possible during a restart or process replacement.

If a production deployment fails:
1. Stop making additional changes until the active release and PM2 state are understood.
2. Inspect pm2 list, process logs, the current symlink, and the latest release directory.
3. Identify the last known-good release.
4. Follow the rollback commands implemented in the current Jenkinsfile. Do not improvise a symlink or PM2 command without confirming the actual process names and paths.
5. Verify the production endpoint after rollback.
6. Record the failing build, commit, error, and recovery action.
A rollback is successful only after the previous application is running and its production smoke check passes.
7. Operational commands
# PM2 process overview
pm2 list
pm2 status

# Inspect logs
pm2 logs --lines 100

# Inspect a specific process (replace with the real name or ID)
pm2 describe <process-name-or-id>

# Check candidate/production HTTP responses
curl -i --max-time 10 http://127.0.0.1:<port>/<health-route>

# Inspect the active release pointer (adjust path for the service)
readlink -f /opt/pipeline-pro/current
readlink -f /opt/pipeline-pro-customer/current
Do not run destructive commands such as pm2 delete all, broad directory deletion, or blanket permission changes on a shared server.
8. Current implementation and planned work
Recorded current pipeline behavior
- Validate, checkout, npm ci, and npm run build.
- Prepare a release and include the service's runtime environment configuration.
- Start a candidate PM2 process on a service-specific test port.
- Retry an HTTP smoke test.
- Update the production PM2 process after the candidate check.
- Update the current symlink and run pm2 save.
- Attempt Slack and email notifications.
Items requiring verification
- Webhook versus manual job trigger.
- Exact release directory naming and environment-file paths.
- Exact PM2 process names and rollback commands.
- Business Owner health endpoint and expected response.
- Whether production deployment causes a brief interruption.
- Whether notification delivery has been tested successfully.
Future improvements — not currently claimed as implemented
- Add automated API/integration tests before production deployment.
- Add a deployment lock to prevent concurrent runs for the same service.
- Make rollback deterministic and test it regularly.
- Add release retention and cleanup with safeguards.
- Add a production post-deployment health check and monitoring window.
- Add a manual approval gate if production changes require approval.