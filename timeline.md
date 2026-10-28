
# Timeline — Wazuh API Connectivity, Authentication and RBAC Audit

## Investigation Timeline

| Stage | Activity | Observation | Outcome |
|---|---|---|---|
| 1 | Tested Windows connectivity to TCP `443` and `55000` | Dashboard reachable; API port initially unreachable | Network-level investigation required |
| 2 | Inspected listening sockets on the Linux VM | Python process listening on `0.0.0.0:55000` and `[::]:55000` | API port had a listener |
| 3 | Checked Linux firewall configuration | `firewalld` active; active zone was `public` on `ens160` | Firewall configuration identified |
| 4 | Added a source-restricted runtime firewall rule | Rule permitted TCP `55000` from `192.168.203.1/32` | Firewall accepted the rule |
| 5 | Retested TCP connectivity from Windows | `TcpTestSucceeded : True` | API port became reachable |
| 6 | Sent an unauthenticated request to the API root | HTTP `401 Unauthorized` | API required authentication |
| 7 | Attempted API authentication with `admin` | HTTP `401` | Credentials were not accepted |
| 8 | Reviewed `/var/ossec/logs/api.log` | Failed `admin` authentication attempts were recorded | Failure confirmed in API logs |
| 9 | Inspected the API RBAC database schema | Identified user, role, policy, and token-blacklist tables | RBAC structure documented |
| 10 | Listed API user metadata in read-only mode | `wazuh` and `wazuh-wui` present; `admin` absent | API and indexer account distinction established |
| 11 | Inspected the installation password archive | API credentials were listed separately from indexer credentials | Correct credential type identified |
| 12 | Authenticated using the appropriate API credentials | HTTP `200`; JWT received | API authentication succeeded |
| 13 | Tested general and agent-related endpoints | `/`, `/agents?limit=5`, and `/agents/summary/status` returned HTTP `200` | Read-only endpoint access verified |
| 14 | Tested manager information endpoint | `/manager/info` returned HTTP `200` | Manager endpoint access verified |
| 15 | Tested security endpoints | `/security/users`, `/security/roles`, and `/security/config` returned HTTP `200` | Selected security endpoints accessible |

## Final State

- **TCP connectivity:** Verified working from Windows to Wazuh API port `55000`.
- **Firewall:** Source-restricted runtime rule added.
- **API authentication:** Successful with an appropriate API account.
- **Read-only API access:** Selected general, agent, manager, and security endpoints returned HTTP `200`.
- **RBAC database:** Inspected in read-only mode; no direct database modification performed.
- **TLS validation:** Certificate verification was bypassed during lab requests and still requires proper validation.
- **Firewall persistence:** Not verified.
- **Full RBAC authorization review:** Not completed.

## Investigation Conclusion

The investigation progressed from an initial TCP connectivity failure to successful authenticated API access. The evidence showed that network reachability, authentication, and authorization are separate stages and should be tested independently.

The lab concluded with successful authentication and endpoint validation. TLS hardening, firewall persistence, and a deeper review of role-policy enforcement remain potential follow-up activities.
