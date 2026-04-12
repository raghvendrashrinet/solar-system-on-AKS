# AKS Ingress & Networking Troubleshooting Report

## 📝 Problem Statement
The application was successfully deployed to Azure Kubernetes Service (AKS), and internal pods were healthy. However, the application was inaccessible via the public Ingress URL, resulting in `Connection Timed Out` errors despite the Azure Load Balancer being active and the NSG rules being open.

---

## 🧪 Phase 1: Isolation Strategy (The "Local" Test)
Before changing any infrastructure, we used **Port-Forwarding** to isolate the root cause. This step determines if the failure is **Inside Kubernetes** or **Outside (Azure Network)**.

1.  **Bypass the Public Network:**
    We mapped the NGINX controller service directly to the local runner/machine:
    ```powershell
    kubectl port-forward -n ingress-basic svc/nginx-ingress-ingress-nginx-controller 8888:80
    ```

2.  **Simulate a Real Request:**
    While the port-forward was active, we sent a request including the expected `Host` header:
    ```powershell
    curl.exe -I -H "Host: solar-system-development.4.224.119.179.nip.io" http://127.0.0.1:8888
    ```

3.  **The Discovery:**
    *   The command returned **`HTTP 200 OK`**.
    *   **Conclusion:** This proved that the NGINX Controller, the Kubernetes Service, and the Backend Pods were all working perfectly. The "Connection Timeout" on the Public IP was confirmed to be an **Azure Load Balancer / Health Probe** issue.

---

## 🛠 Phase 2: Critical Fixes Applied

### 1. Linking the Ingress Class
*   **Issue:** `kubectl get ingress` showed an empty `CLASS` column.
*   **Cause:** Modern Kubernetes versions require an explicit `ingressClassName` to tell the NGINX controller to "claim" and manage that specific Ingress resource.
*   **Fix:** Added the class name to the manifest `spec`.
    ```yaml
    spec:
      ingressClassName: nginx
    ```

### 2. Resolving SSL Certificate Mismatches
*   **Issue:** NGINX logs reported errors looking for a non-existent `ingress-local-tls` secret.
*   **Fix:** Removed the `tls:` section to allow stable **HTTP** traffic during the initial deployment phase.

### 3. Azure Load Balancer Health Probe Fix
*   **Issue:** The Public IP timed out despite healthy pods.
*   **Cause:** Azure's Load Balancer health probes were failing due to internal "traffic hopping" (the default `Cluster` policy).
*   **Fix:** Patched the NGINX Service to use a `Local` traffic policy.
    ```bash
    kubectl patch svc nginx-ingress-ingress-nginx-controller -n ingress-basic -p '{"spec":{"externalTrafficPolicy":"Local"}}'
    ```

---

## 🔍 Verification & Health Checks

Use these commands to verify the health of the entire stack:


| Component | Command | Expected Result |
| :--- | :--- | :--- |
| **Pod Status** | `kubectl get pods -n development -l app=solar-system` | **STATUS** is `Running` and **READY** is `1/1`. |
| **Ingress Link** | `kubectl get ingress -n development` | **CLASS** must be `nginx`. |
| **Backend Sync** | `kubectl describe ingress solar-system` | **Backends** must list active Pod IPs (not `unknown`). |
| **Application** | `curl.exe -I http://<full-nip-io-url>` | Returns **`200 OK`**. |

### Detailed Pod Health Check
To see exactly why a pod might be failing or to check its startup logs:
```powershell
# Check for restarts or specific errors in pod status
kubectl get pods -n development -o wide

# View live application logs to see incoming requests
kubectl logs -f -n development -l app=solar-system --tail=20
 The Application Pod Logs (The "Core")These logs show the internal startup and any code-level errors.Note: If your Node.js   code does not have a logging middleware (like morgan), you will only see the "Server running" message here and no browser hits.

### 📝 Monitoring Real-Time Traffic

To fully verify the traffic flow, you must monitor two different log layers. 

#### 1. The Ingress Controller Logs (The "Front Door")
These logs show every request hitting the cluster from the internet. If you see lines here when you refresh your browser, your Azure Networking is 100% correct.
```powershell
# Watch real-time traffic hitting the NGINX Ingress
kubectl logs -f -n ingress-basic -l app.kubernetes.io/name=ingress-nginx --tail=20
