# Investigation Notes 

## 1. Environment and Evidence

| Item | Observed Value |
|---|---|
| Windows source address | `192.168.203.1` |
| Wazuh VM address | `192.168.203.129` |
| Dashboard TCP port | `443` |
| API TCP port | `55000` |
| Firewall active zone | `public` |
| Linux network interface | `ens160` |
| Wazuh Manager service | Active and running |
| API process | Python process listening on port `55000` |
| API log location | `/var/ossec/logs/api.log` |
| RBAC database | `/var/ossec/api/configuration/security/rbac.db` |

## 2. Network Connectivity

The initial connectivity tests showed that TCP port `443` was reachable, while port `55000` was not reachable from Windows.

The Linux socket check showed listeners on all IPv4 and IPv6 interfaces:

    LISTEN 0 2048 0.0.0.0:55000 0.0.0.0:* users:(("python3",pid=3082,fd=42))
    LISTEN 0 2048 [::]:55000 [::]:* users:(("python3",pid=3082,fd=44))

This established that a Python process was listening on the expected API port. The process listing and subsequent successful API responses supported the conclusion that the Wazuh API was operating.

The active firewall zone was `public`, attached to interface `ens160`. A runtime rich rule was added to permit TCP port `55000` only from `192.168.203.1/32`.

The subsequent Windows TCP test returned:

    TcpTestSucceeded : True

**Finding:** Network connectivity to the API was established after the firewall change. The initial failure was consistent with the port not being permitted through the firewall.

## 3. Authentication Investigation

An unauthenticated request to the API root returned HTTP `401 Unauthorized`. The response body stated that no authorization token had been provided.

This response showed that the API was reachable and enforcing authentication.

Initial login attempts using the username `admin` also returned HTTP `401`. Review of the installation password archive showed that `admin` was listed as an indexer account, while the API credentials were listed separately under `api_username` and `api_password`.

The RBAC database inspection showed these account records:

| ID | Username | `allow_run_as` |
|---:|---|---:|
| 1 | `wazuh` | 1 |
| 2 | `wazuh-wui` | 1 |

The database did not contain an `admin` record. The API logs recorded failed authentication attempts for `admin` from `192.168.203.1`.

The installation archive contained separate credential entries for `wazuh` and `wazuh-wui`. After using the appropriate API credentials, the authentication request returned HTTP `200`, and a JWT was received.

**Finding:** The failed `admin` login was consistent with using an indexer account against the Wazuh API. Successful authentication was later confirmed using an API account.

## 4. RBAC Database Inspection

The database was opened using SQLite in read-only mode. The inspection identified the following tables:

- `users`
- `roles`
- `policies`
- `user_roles`
- `roles_policies`
- `roles_rules`
- `rules`
- `roles_token_blacklist`
- `users_token_blacklist`
- `runas_token_blacklist`

The `users` table contained `id`, `username`, `password`, `allow_run_as`, and `created_at` columns.

Only account metadata was queried. Password hashes and credential values were not extracted from the RBAC database.

**Finding:** The database structure and account records were inspected without directly modifying the RBAC database.

## 5. Authenticated API Validation

After obtaining a JWT, the token was supplied in the HTTP `Authorization` header using the Bearer scheme.

The following read-only requests returned HTTP `200`:

| Endpoint | Result |
|---|---|
| `/` | HTTP `200` |
| `/agents?limit=5` | HTTP `200` |
| `/agents/summary/status` | HTTP `200` |
| `/manager/info` | HTTP `200` |
| `/security/users` | HTTP `200` |
| `/security/roles` | HTTP `200` |
| `/security/config` | HTTP `200` |

These results confirmed that the authenticated session could access the tested endpoints.

The status codes alone do not establish the exact permissions assigned to every user or prove that access controls prevent every unauthorized operation.

## 6. Security Assessment

### Positive observations

- The API required authentication for protected requests.
- API and indexer credentials were distinguishable in the installation archive.
- The firewall rule limited inbound API access to the Windows host address.
- API authentication and endpoint responses were validated with explicit HTTP status codes.
- The RBAC database was inspected without direct modification.

### Limitations and outstanding checks

- The firewall rule was added at runtime; persistence across reloads or restarts was not verified.
- TLS certificate validation was bypassed during the PowerShell tests using `-SkipCertificateCheck`.
- The exact role and policy permissions assigned to each account were not fully audited.
- No unauthorized-operation test was performed.
- No claim is made that the API is secure against every access-control or configuration weakness.

