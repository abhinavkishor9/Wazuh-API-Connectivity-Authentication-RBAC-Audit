# Wazuh API Connectivity, Authentication and RBAC Audit

## Overview

This lab investigates Wazuh API connectivity from a Windows host to a Wazuh Manager running in a Linux virtual machine. The work focuses on validating network access, troubleshooting authentication failures, inspecting API account metadata, and verifying authenticated access to selected read-only API endpoints.

The investigation follows an evidence-based approach: a reachable TCP port does not prove successful authentication, and an HTTP `401 Unauthorized` response does not necessarily indicate a network failure.

## Objectives

- Verify TCP connectivity to the Wazuh dashboard and API ports.
- Inspect the Linux firewall configuration and apply a source-restricted runtime rule.
- Distinguish Wazuh API credentials from Wazuh indexer credentials.
- Investigate API authentication failures using API logs and the RBAC database schema.
- Identify API accounts without exposing password hashes or secrets.
- Authenticate to the API and validate read-only endpoints.
- Document findings, troubleshooting steps, and the investigation timeline.

## Lab Environment

| Component | Details |
|---|---|
| Client | Windows host with PowerShell 7 |
| Server | Wazuh Manager on a Linux VM |
| Windows VMnet8 address | `192.168.203.1` |
| Wazuh VM address | `192.168.203.129` |
| Dashboard port | TCP `443` |
| Wazuh API port | TCP `55000` |
| Firewall | `firewalld` |
| API RBAC database | `/var/ossec/api/configuration/security/rbac.db` |
| API log | `/var/ossec/logs/api.log` |

## Investigation Workflow

1. Tested TCP connectivity to ports `443` and `55000`.
2. Confirmed that the API service was listening on port `55000`.
3. Inspected the active `firewalld` zone and added a runtime rule restricted to the Windows host IP.
4. Tested unauthenticated API access and observed HTTP `401 Unauthorized`.
5. Reviewed API logs to distinguish unauthenticated requests from failed login attempts.
6. Inspected the RBAC database schema and listed account metadata without retrieving password hashes.
7. Examined the installation password archive and distinguished API credentials from indexer credentials.
8. Authenticated successfully using the appropriate Wazuh API account.
9. Validated selected read-only API endpoints and security-related endpoints.

## Key Findings

- The API was listening on all IPv4 and IPv6 interfaces on port `55000`.
- The initial Windows TCP test failed before the firewall rule was added; subsequent testing confirmed TCP connectivity.
- The API returned HTTP `401` for requests without an authorization token and for unsuccessful login attempts.
- The RBAC database contained `wazuh` and `wazuh-wui` user records; `admin` was not present in that database.
- The installation password archive distinguished `api_username` and `api_password` entries from `indexer_username` and `indexer_password` entries.
- Authentication subsequently succeeded with HTTP `200`.
- Authenticated requests to `/`, `/agents?limit=5`, `/agents/summary/status`, and `/manager/info` returned HTTP `200`.
- Requests to `/security/users`, `/security/roles`, and `/security/config` also returned HTTP `200`.

## Security Considerations

- The firewall rule was restricted to source address `192.168.203.1/32` and applied at runtime.
- API credentials, passwords, JWTs, password hashes, and private keys must not be committed to the repository.
- `-SkipCertificateCheck` was used as a temporary lab diagnostic. It bypasses TLS certificate validation and is not a production recommendation.
- The RBAC database was inspected in read-only mode; it was not modified directly.
- HTTP `200` confirms a successful response, not necessarily that every permission or role assignment is correct.
- The runtime firewall rule may not survive a firewall reload or system restart. Its persistence was not established by this investigation.

## Conclusion

The investigation established a working path from the Windows host to the Wazuh API, resolved the initial authentication mismatch, and verified access to selected API endpoints. It also demonstrated the importance of separating network connectivity, authentication, and authorization into distinct troubleshooting stages.

The findings apply to the tested lab environment and the endpoints checked during this investigation.
