Troubleshooting Guide
1. How to investigate a failed deployment
Use this order. It narrows the failure without changing production unnecessarily.
1. Open the failed Jenkins build and identify the first failing stage.
2. Read the first meaningful error above the final stack trace; later errors may be consequences.
3. Confirm the service and commit being deployed.
4. Check server process state with pm2 list.
5. Check candidate/production logs with pm2 logs --lines 100.
6. Test the relevant local HTTP endpoint with curl.
7. Check the runtime configuration path without printing secret values.
8. Make one correction, then rerun the pipeline.
Do not respond to every failure by restarting PM2 or deleting files. That can destroy evidence and make a recoverable issue worse.
2. Service reference
Setting	Customer API	Business Owner API
Deployment directory	/opt/pipeline-pro-customer	/opt/pipeline-pro
Candidate port	5002	5001
Production port	4002	4000
Candidate smoke route	/v1/categories	/v1/health (verify)


3. Jenkins job does not start
Likely causes
- The job trigger is not configured.
- GitLab webhook URL or secret is incorrect.
- Jenkins is not reachable from GitLab.
- The webhook targets the wrong job or branch.
Checks
- Review Jenkins job configuration and build triggers.
- Review GitLab webhook delivery history.
- Confirm the webhook URL and secret match the Jenkins configuration.
- Check Jenkins system logs for incoming webhook or authentication errors.
Correction
Fix the trigger configuration and send a test event from GitLab. Do not assume that adding a Jenkinsfile automatically configures a webhook.
4. GitLab checkout or authentication fails
Typical symptoms
- Authentication failed
- Repository not found
- Permission denied
- Jenkins cannot fetch the selected branch.
Likely causes
- Wrong repository URL or branch.
- Expired/revoked token.
- Credential ID mismatch.
- Token lacks repository read permission.
- Jenkins credential is unavailable to the job/folder.
Checks
- Verify the GitLab repository URL and branch in the Jenkinsfile/job.
- In Jenkins Credentials, verify the credential ID referenced by the pipeline. The recorded ID is access -token; confirm the exact spelling, including spaces.
- Confirm the token is active and has the minimum required repository access.
- Check whether the repository was renamed or moved.
Correction
Update the Jenkins credential securely and reference its credential ID. Never hard-code a personal access token in the Jenkinsfile or print it in the build log.
5. npm ci fails
Typical symptoms
- Lockfile/package mismatch.
- Dependency not found.
- Registry authentication or network error.
- Node/npm version incompatibility.
Checks
node --version
npm --version
Review the first npm error in the Jenkins console. Confirm package-lock.json is committed and synchronized with package.json.

Correction
Use the Node/npm version supported by the project, regenerate and commit the lockfile only when dependency definitions intentionally changed, and fix registry credentials/network access if applicable. Do not replace npm ci with npm install just to hide a lockfile problem.
6. npm run build fails
Likely causes
- TypeScript compilation error.
- Missing dependency or build script.
- Incorrect Node.js version.
- Build-time configuration unexpectedly required.
- Code change introduced a compile error.
Checks
cat package.json
node --version
npm --version
Run the same build locally or in a clean workspace using the same Node version. Inspect the first compiler error.

Correction
Fix the source/configuration issue and commit the change. Do not bypass the build stage or deploy an old build artifact as if it were the new commit.
7. Release copy or permissions fail
Typical symptoms
- Permission denied
- No such file or directory
- Copy command exits non-zero.
- current symlink cannot be updated.
Likely causes
- Jenkins deployment user cannot write to the target directory.
- Release path is incorrect.
- Expected build output does not exist.
- Disk space is exhausted.
- Another deployment is changing the same path.
Checks
df -h
df -i
ls -ld /opt/pipeline-pro
ls -ld /opt/pipeline-pro-customer
Check the actual target path and build output path in the Jenkinsfile. Avoid displaying .env contents.

Correction
Grant the deployment account only the required directory permissions, correct the path, or free disk space using an approved cleanup process. Do not use chmod -R 777.
8. Missing or invalid environment configuration
Typical symptoms
- Application exits during startup.
- Missing environment-variable error.
- Database/authentication/external API connection fails.
Known configuration examples
- DATABASE_URL
- JWT/OTP secrets
- R2 configuration
- Google Maps configuration
- Groq configuration
The exact required variables differ by service. This list is not a complete environment template.

Checks
# Check whether the file exists; do not print its contents.
test -f /path/to/service/.env && echo "env file exists" || echo "env file missing"
Check the .env source/destination path in the Jenkinsfile and the application logs. If checking a variable, print only whether it is set, never its value.

Correction
Restore the correct environment file from the approved secret/configuration source and set restrictive permissions. Never commit .env or paste secret values into tickets or chat.
9. Candidate PM2 process does not start
Typical symptoms
- Candidate process is errored or repeatedly restarts.
- Candidate port does not respond.
- Jenkins cannot find the process.
Checks
pm2 list
pm2 logs --lines 100
Check the candidate port for a listener:
sudo ss -ltnp | grep -E ':5001|:5002'
Use the port for the service being deployed. Verify the process name, working directory, start script, runtime environment, and port in the Jenkinsfile.

Likely causes
- Wrong start command or build output path.
- Missing environment file.
- Port already in use.
- Application startup exception.
- PM2 process name mismatch.
Correction
Fix the actual startup/configuration issue. Do not kill an unknown process just because it occupies a port; identify its owner first.
10. Smoke test fails
Customer API candidate
curl -i --max-time 10 http://127.0.0.1:5002/v1/categories
Business Owner candidate (route must be verified)
curl -i --max-time 10 http://127.0.0.1:5001/v1/health
Common causes
- Candidate process is not ready before the request.
- Wrong port or route.
- Endpoint requires authentication.
- Application returned 4xx/5xx.
- Database or downstream service is unavailable.
- Jenkins smoke-test expected status/body does not match the API response.
Correction
Check PM2 logs and call the endpoint manually from the server. Confirm the endpoint's intended unauthenticated behavior and expected response. Adjust retry timing only if startup time is genuinely the issue; retries should not conceal a broken service.
11. Production process fails after candidate passes
A candidate smoke test is only a basic check. Production can still fail due to a different port, environment, process definition, or deployment path.

Checks
pm2 list
pm2 logs --lines 100
curl -i --max-time 10 http://127.0.0.1:<production-port>/<route>
Confirm the current symlink:
readlink -f /opt/pipeline-pro/current
readlink -f /opt/pipeline-pro-customer/current
Correction
Identify which release PM2 is running and which release current points to. Follow the rollback procedure in the current Jenkinsfile. Verify the production endpoint after recovery.
The pipeline's rollback-related logic is not a guarantee of zero downtime. A PM2 restart or process replacement can interrupt requests.

12. Port already in use
Typical symptoms
- EADDRINUSE
- Application cannot bind to candidate or production port.
Checks
sudo ss -ltnp | grep ':5001'
sudo ss -ltnp | grep ':5002'
sudo ss -ltnp | grep ':4000'
sudo ss -ltnp | grep ':4002'
Replace the port with the one reported in the error. Identify the owning process before taking action.

Correction
Resolve the duplicate process or correct the configured port. Do not blindly kill the PID; it may be the live production process.
13. Slack or email notification fails
Likely causes
- Missing or inaccessible Jenkins credential.
- Slack webhook expired or revoked.
- Email Extension plugin/configuration missing.
- SMTP settings or sender configuration incorrect.
- Network restrictions.
Checks
- Confirm Jenkins credential ID slack-webhook exists and is accessible to the job.
- Confirm Email Extension (emailext) is installed and global SMTP settings are configured.
- Review the notification stage logs without exposing webhook URLs or credentials.
Correction
Update the credential/configuration securely and test notification delivery. Treat notification failure separately from application deployment status unless the pipeline explicitly makes notifications a release gate.
14. PM2 process does not survive a server reboot
pm2 save stores the process list; it does not alone guarantee that PM2 is configured to start at boot.

Checks
pm2 list
pm2 startup
pm2 startup prints a command that must be run with the appropriate privileges. Follow the server's operational procedure; do not execute arbitrary commands from an untrusted source.

Correction
Configure the PM2 startup service for the correct deployment user, save the intended process list, and validate recovery during an approved maintenance window.
15. Safe recovery checklist
Before manually recovering a failed deployment:
- [ ] Confirm which service failed (Customer or Business Owner).
- [ ] Record the failed Jenkins build number and commit.
- [ ] Identify candidate and production PM2 processes.
- [ ] Identify the active release and current symlink target.
- [ ] Check logs and endpoint responses.
- [ ] Confirm the previous known-good release.
- [ ] Use the rollback steps from the current Jenkinsfile.
- [ ] Verify the production endpoint after recovery.
- [ ] Record the root cause and corrective action.
16. Known gaps and future improvements
The following should be treated as improvement work, not existing functionality unless independently verified:
- A tested, deterministic rollback procedure.
- A post-deployment production health check.
- Deployment locking for concurrent builds.
- Automated cleanup with release-retention safeguards.
- Integration/API tests beyond the current smoke check.
- Monitoring and alerting for post-deployment regressions.
- A documented and tested server-reboot recovery procedure.