# Module 07 — Security & User Management

## Learning Objectives
- Configure Jenkins authentication (local, LDAP, SSO/OAuth).
- Apply Matrix and Role-Based authorization strategies.
- Manage credentials safely.
- Harden the controller against common attacks.
- Integrate external secret stores like Vault.

## 1. Authentication

Authentication answers *who are you?*

### Local users (Jenkins' own user database)
- Simple, works out of the box.
- Best for small teams or sandbox installs.
- Users are stored under `JENKINS_HOME/users/`.
- Configure under **Manage Jenkins → Security → Security Realm → Jenkins' own user database**.
- Enable or disable user sign-up. Disable for any non-personal install.

### LDAP / Active Directory
- Use your existing corporate directory.
- Install **LDAP** or **Active Directory** plugin.
- Configure server URL, bind DN/password, search filter, group membership filter.
- Test the configuration *before* clicking save — locking yourself out is painful.

### SSO / OAuth (recommended for teams)
Common options:
- **GitHub Authentication** plugin — log in with GitHub.
- **Google Login** plugin — log in with Google.
- **OpenID Connect** / **SAML** plugins — for enterprise SSO via Okta, Keycloak, Azure AD, etc.

When using OAuth/OIDC:
- Restrict by org/group claim (don't allow any GitHub user to log in).
- Map external groups to internal authorization roles.

### Lockout recovery
If you misconfigure auth and can't log in:
1. Stop Jenkins.
2. Edit `JENKINS_HOME/config.xml`:
   - Set `<useSecurity>false</useSecurity>`.
3. Start Jenkins. The UI is unauthenticated — fix the security realm and turn security back on.

This is also why the controller filesystem must be tightly access-controlled.

## 2. Authorization

Authorization answers *what are you allowed to do?*

**Manage Jenkins → Security → Authorization** offers:

### Anyone can do anything
Default before setup. **Never** leave a real install like this.

### Logged-in users can do anything
Easy and weak — every authenticated user is effectively an admin.

### Matrix-Based Security
A grid: rows = users/groups, columns = permissions (Overall/Read, Job/Build, Credentials/Update, etc.). Tick what each principal can do.

### Project-Based Matrix Authorization Strategy
Same matrix, but each job can also have its own matrix overriding the global one. Use this for shared Jenkins where teams own their jobs.

### Role-Based Strategy (plugin)
Define **roles** with permission sets, then assign users/groups to roles. Scales much better than the raw matrix.

Typical roles:
- **admin** — everything.
- **developer** — read everything, build their team's jobs.
- **reader** — read-only.
- **integrator** — release and deploy.

Use **Project roles** to scope permissions to job-name patterns (e.g., `team-payments/.*`).

### Principle of least privilege
- New users start as readers.
- Promote only when needed.
- Audit periodically — remove permissions that aren't used.

## 3. Credentials Store

Already introduced in Module 04. Deeper points:

### Domains
A credential domain restricts where a credential is valid (host, scheme). Use domains to prevent a token meant for `nexus.example.com` from being usable elsewhere.

### Folder credentials
Credentials added to a folder are only visible to jobs inside that folder. Good for multi-team isolation.

### Credential binding in pipelines
```groovy
withCredentials([usernamePassword(credentialsId: 'nexus-deploy',
                                  usernameVariable: 'NEXUS_USER',
                                  passwordVariable: 'NEXUS_PASS')]) {
  sh 'curl -u $NEXUS_USER:$NEXUS_PASS ...'
}
```
Jenkins automatically masks the secret in console output.

### Don't
- Don't put secrets in job descriptions or `Jenkinsfile`.
- Don't echo secrets to logs (`set -x`, `env | grep PASS`).
- Don't store secrets in environment variables on agents at the OS level.

## 4. HashiCorp Vault Integration

For teams that already run Vault, the **HashiCorp Vault plugin** lets Jenkins fetch secrets at build time instead of storing them in the Jenkins credentials store.

### Setup
1. Configure a Vault server URL and an authentication method (AppRole is typical).
2. Add a Vault credential (Role ID + Secret ID).
3. In a pipeline:
   ```groovy
   withVault([vaultSecrets: [[path: 'secret/data/jenkins/nexus',
                              secretValues: [[envVar: 'NEXUS_PASS', vaultKey: 'password']]]]]) {
     sh 'echo $NEXUS_PASS | sha256sum'
   }
   ```
4. The secret never lives on the Jenkins filesystem — it's fetched per build.

### Benefits
- Centralized secret rotation.
- Audit trail of which job accessed which secret.
- Short-lived credentials.

## 5. Securing the Controller

### TLS everywhere
- Reverse proxy with valid certificates (Module 02).
- Redirect HTTP → HTTPS.
- Modern cipher suites only.

### CSRF protection
On by default. Don't disable. The CSRF crumb is required for API write operations:
```bash
CRUMB=$(curl -s -u user:token "https://jenkins/crumbIssuer/api/json" | jq -r .crumb)
curl -X POST -H "Jenkins-Crumb: $CRUMB" -u user:token "https://jenkins/job/foo/build"
```

### Agent-to-controller security
Treat agents as untrusted. The **Agent → Controller Access Control** subsystem restricts what agents can do back to the controller. Keep it enabled.

### Script approval
The **Script Security** plugin gates Groovy scripts that touch internals. Approve scripts deliberately; an approved unsafe script is a controller takeover waiting to happen.

### Markup formatter
**Manage Jenkins → System → Markup Formatter → Safe HTML** (OWASP Markup Formatter). Otherwise job descriptions can host stored XSS.

### Hide build version
Add to the JVM args:
```
-Dhudson.model.DirectoryBrowserSupport.CSP="sandbox; default-src 'none';"
```
This locks down what the `/static/...` and workspace browser endpoints can do.

### Restrict signup and anonymous access
- Disable **Allow users to sign up**.
- Set **Anonymous** to no permissions in the authorization matrix.

### Reverse-proxy headers
Make sure your proxy sets `X-Forwarded-For` and `X-Forwarded-Proto`; configure Jenkins to trust them so audit logs show real client IPs.

## 6. Audit Logging

Install the **Audit Trail** plugin to log:
- Logins and login failures.
- Job configuration changes.
- Build triggers.
- Permission changes.

Send the audit log to your central log system (syslog, Elastic, Splunk). Reviewing audit logs catches stale accounts, misuse, and suspicious script approvals.

The **Job Configuration History** plugin keeps a versioned history of every job config change, with diffs and revert.

## 7. Account Hygiene

- Use a **break-glass admin** account stored offline; the day-to-day admins use SSO.
- Remove leavers promptly (or rely on SSO to do it automatically).
- Use **service accounts**, not personal tokens, for API automation.
- Rotate credentials at least annually; immediately on any suspicion of compromise.

## 8. CVE & Update Discipline

- Subscribe to the Jenkins **security advisory mailing list**.
- Apply LTS security updates within a defined window (e.g., 30 days, or 7 days for critical).
- The Jenkins UI shows a banner when there's a security advisory affecting your installed plugins.

## 9. Hands-On Exercise

1. Enable a real authentication strategy (your team's SSO if available, else LDAP).
2. Install the **Role-based Authorization Strategy** plugin.
3. Create roles: `admin`, `developer`, `reader`. Assign yourself to `developer`; create another test user as `reader`.
4. Verify the test user can read jobs but cannot trigger builds or edit config.
5. Add a credential of type "Secret text" and write a pipeline that uses it via `withCredentials`. Confirm the value is masked in the log.
6. Enable the **Audit Trail** plugin and review the resulting log.

## 10. Knowledge Check
1. How would you recover if you locked yourself out of Jenkins by misconfiguring auth?
2. What's the difference between Matrix-Based and Role-Based authorization?
3. Why scope credentials to a folder?
4. What does `withCredentials` do that simply setting an env var does not?
5. Name three controller-hardening practices.

## What's Next
**Module 08** is about day-2 operations: logs, backups, JVM tuning, monitoring, and disaster recovery.
