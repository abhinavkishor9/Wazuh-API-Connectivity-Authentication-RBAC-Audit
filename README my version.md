# Wazuh-API-Connectivity-Authentication-RBAC-Audit
## Overview
The Wazuh API provides a programmatic interface for interacting with Wazuh Manager functionality, retrieving security information, and performing authorized administrative operations. Access depends on API availability, authentication, and the permissions assigned to the authenticated account.

| Concept | Meaning |
|---|---|
| **API availability** | Whether the Wazuh API service is running and accepting connections on the expected address and port. |
| **Authentication** | Verifying the identity of a client requesting API access. |
| **Authorization** | Determining which API operations an authenticated account is permitted to perform. |
| **Least privilege** | Limiting each account to the permissions required for its responsibilities. |
| **API audit** | Recording endpoints, HTTP status codes, responses, and supporting evidence. |
| **Evidence-based conclusion** | Distinguishing confirmed observations from assumptions and tests that remain incomplete. |

This lab investigates Wazuh API connectivity from a Windows host to a Wazuh Manager running in a Linux virtual machine. The work focuses on validating network access, troubleshooting authentication failures, inspecting API account metadata, and verifying authenticated access to selected read-only API endpoints.

The investigation follows an evidence-based approach: a reachable TCP port does not prove successful authentication, and an HTTP `401 Unauthorized` response does not necessarily indicate a network failure.

## Lab Objectives

- Verify network connectivity between the Windows investigation host and the Wazuh API service running on the Wazuh Manager.
- Identify and troubleshoot API connectivity issues involving the listening port, service availability, and firewall access controls.
- Validate API authentication by comparing unauthenticated requests with requests using authorized API credentials.
- Obtain an authentication token and use it to access selected Wazuh API endpoints.
- Perform read-only checks of manager information, agent status, security users, roles, and API security configuration.
- Inspect Wazuh RBAC database metadata to understand how users, roles, and policies relate to access permissions.
- Record endpoint paths, HTTP status codes, command outputs, and relevant logs as investigation evidence.
- Distinguish confirmed findings from unverified assumptions and document limitations, including TLS certificate validation bypasses and firewall rule persistence.

## Lab Scenario

A security analyst is assessing the connectivity, authentication, and access controls of a Wazuh API deployed in a virtual lab environment. The analyst needs to confirm that the API is reachable from the Windows investigation host, determine whether network controls affect access, and verify that authorized credentials permit access to the required management and security information.

During the initial assessment, the Wazuh Dashboard is reachable, but the API port `55000` is not accessible from the Windows host. The analyst investigates the API listener and firewall configuration on the Wazuh Manager, then applies a narrowly scoped firewall rule to allow connections from the designated lab host. Connectivity is retested to establish whether the network-level issue has been resolved.

Once connectivity is available, the analyst examines API authentication behavior. Unauthenticated requests return an HTTP `401 Unauthorized` response, indicating that a valid authorization token is required. The analyst then authenticates with the appropriate API credentials and uses the resulting token to query selected endpoints.

The investigation includes the following activities:

- Validate API connectivity and document the relevant network and firewall findings.
- Compare unauthenticated requests with authenticated API requests.
- Retrieve manager information, agent status, and selected security configuration details using read-only endpoints.
- Inspect user and role metadata in the Wazuh RBAC database to understand the access-control structure.
- Record HTTP status codes, command outputs, and relevant logs to support each conclusion.

The assessment focuses on evidence collection and read-only validation rather than changing user permissions or performing destructive tests. The final findings distinguish confirmed observations from unverified assumptions and identify limitations, including the use of certificate-validation bypass during temporary lab testing and the need to verify firewall rule persistence separately.
  
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

