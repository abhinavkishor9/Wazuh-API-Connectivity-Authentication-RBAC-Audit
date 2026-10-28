# Troubleshooting Notes

## 1. TCP Port 55000 Initially Unreachable

### Symptom

The Windows host could reach the Wazuh dashboard on port `443`, but the API port `55000` initially failed the TCP connection test.

### Checks performed

From Windows PowerShell:

    Test-NetConnection 192.168.203.129 -Port 443
    Test-NetConnection 192.168.203.129 -Port 55000

The dashboard port was reachable, while the API port initially returned `TcpTestSucceeded : False`.

From the Linux VM:

    sudo ss -lntp | grep ':55000'

The output showed a Python process listening on port `55000` on IPv4 and IPv6 wildcard addresses.

### Resolution

The active firewall zone was identified as `public`, attached to interface `ens160`. A source-restricted runtime rule was added:

    sudo firewall-cmd --zone=public \
      --add-rich-rule='rule family="ipv4" source address="192.168.203.1/32" port port="55000" protocol="tcp" accept'

The firewall returned:

    success

A subsequent Windows test returned `TcpTestSucceeded : True`.

### Lesson

A service listening on a port does not guarantee that remote clients can reach it. Check the listener, routing, and active firewall zone before changing firewall rules.

The rule was runtime-only; persistence was not tested.

## 2. HTTP 401 from an Unauthenticated Request

### Symptom

A request to the API root returned:

    HTTP/1.1 401 Unauthorized

The response body indicated that no authorization token had been provided.

### Interpretation

The TCP connection and HTTP exchange succeeded, but the API rejected the request because it was not authenticated.

### Resolution

The API authentication endpoint was used to obtain a JWT. Subsequent requests included the token in the Bearer authorization header.

### Lesson

An HTTP `401` response does not, by itself, indicate a firewall or network failure. Distinguish transport connectivity from application authentication.

## 3. Login Attempts Using `admin` Failed

### Symptom

Authentication attempts using `admin` returned HTTP `401`.

### Investigation

The installation password archive was inspected with password values redacted. It listed indexer credentials separately from API credentials.

The RBAC database was inspected in read-only mode. The `users` table contained `wazuh` and `wazuh-wui`, but not `admin`.

The API logs also recorded failed `admin` authentication attempts from the Windows host.

### Resolution

The appropriate API account and its associated API password were used. Authentication subsequently returned HTTP `200`, and a JWT was received.

### Lesson

Do not assume that dashboard or indexer credentials are valid for the Wazuh API. Identify the intended account type before attempting authentication or resetting passwords.

## 4. Password Archive Search Returned No Output

### Symptom

A `grep` expression intended to find the `wazuh` account entry returned no output.

### Cause

The search pattern did not match the structure of the password file. The file used separate `api_username` and `api_password` fields rather than the expected `username` field format.

### Resolution

The archive was listed first to verify the file path, then its content was inspected with password values redacted.

### Lesson

When a search returns no matches, verify the file structure and field names instead of assuming the file is missing or empty.

## 5. Authenticated API Requests Returned HTTP 200

### Symptom

After authentication, requests to the API root, agent endpoints, manager information, and security endpoints returned HTTP `200`.

### Validation

The token was supplied using:

    Authorization = "Bearer $Jwt"

The tested endpoints included:

    /
    /agents?limit=5
    /agents/summary/status
    /manager/info
    /security/users
    /security/roles
    /security/config

### Interpretation

The responses confirmed that the authenticated session could access the tested endpoints.

### Remaining checks

- Review role and policy assignments if a full RBAC audit is required.
- Validate TLS certificates instead of permanently bypassing certificate checks.
- Confirm whether the firewall rule should persist across reloads or restarts.
- Test denied access only through safe, authorized, non-destructive requests.

