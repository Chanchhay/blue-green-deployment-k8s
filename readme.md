Your setup works like a relay race—each piece passes your request one step closer to the Headlamp pod inside your Kubernetes cluster. Here is what every step we configured actually does in simple terms:

### 1. DNS (`headlamp.kagea.site` → `34.2.137.230`)

* **What it does:** Acts like a phonebook for the internet.
* **How it works:** When you type `[https://headlamp.kagea.site](https://headlamp.kagea.site)` into your browser, DNS translates that domain name into your GCP Load Balancer's public IP address (`34.2.137.230`) so your browser knows where to send the request.

---

### 2. Multi-Zone Unmanaged Instance Groups (`ig-worker-a` and `ig-worker-b`)

* **What they do:** Group your worker VMs so the Load Balancer knows which servers belong to your cluster.
* **How it works:** Because `worker01` lives in `asia-southeast1-a` and `worker02` lives in `asia-southeast1-b` (two separate data center buildings in Singapore), GCP requires one unmanaged instance group per zone. Keeping them in two zones means if one building has an outage, the other worker still stays online.

---

### 3. GCP Load Balancer (`gcp-lb-k8s` on `34.2.137.230`)

* **What it does:** Acts as the public front door to your cluster and splits incoming traffic between `worker01` and `worker02`.
* **How it works:** It listens on ports `80` (HTTP) and `443` (HTTPS) and passes those packets straight through to your worker VMs. Every 5 seconds, it also pings port `8080` (`/ping`) on both workers—if a worker stops replying, the Load Balancer automatically stops sending users to that broken node.

---

### 4. VPC Firewall Rules (The Security Guards)

* **What they do:** Block all incoming connections by default except the two exact types of traffic you allowed.
* **How they work:**
* **Health-check rule (Port `8080`):** Lets **only Google's monitoring servers** (`35.191.0.0/16`, `209.85.152.0/22`, `209.85.204.0/22`, `130.211.0.0/22`) check `/ping` on port `8080`, keeping random internet scanners out of Traefik's internal admin port.
* **Public traffic rule (Ports `80` and `443`):** Lets **anyone on the internet** (`0.0.0.0/0`)—including your browser and Let's Encrypt—reach your web apps over HTTP and HTTPS.



---

### 5. Traefik Ingress Controller (`DaemonSet` + `hostPort`)

* **What it does:** Acts as the smart traffic router (reverse proxy) inside your cluster.
* **How it works:**
* Running Traefik as a **`DaemonSet`** forces Kubernetes to place **one Traefik pod on every worker node** (`worker01` and `worker02`).
* Setting **`hostPort: 80` and `443**` opens ports `80` and `443` directly on each worker VM's network card so the GCP Load Balancer can hand traffic straight to Traefik.
* Once Traefik receives a request, it reads the domain name (`headlamp.kagea.site`), unlocks the HTTPS encryption using your TLS certificate, and forwards the request to the right app inside Kubernetes.



---

### 6. `cert-manager` and `ClusterIssuer` (`letsencrypt-prod`)

* **What they do:** Act as an automated SSL/TLS certificate robot.
* **How it works:**
* **`ClusterIssuer` (`letsencrypt-prod`):** Holds your account registration with Let's Encrypt (a free Certificate Authority) and tells `cert-manager` to prove domain ownership over HTTP (`http01` via Traefik).
* **The verification check:** When you created your Ingress, `cert-manager` temporarily placed a secret verification file at `[http://headlamp.kagea.site/.well-known/acme-challenge/](http://headlamp.kagea.site/.well-known/acme-challenge/)...`. Let's Encrypt downloaded that file over port `80` to verify that you control `headlamp.kagea.site`, issued a trusted SSL/TLS certificate, and saved it inside a Kubernetes Secret named `headlamp-tls-cert`. It will also automatically renew this certificate before it expires.



---

### 7. `Ingress` and Headlamp `Service`

* **What they do:** Connect the public domain name to the actual Headlamp application pod.
* **How it works:**
* The **`Ingress`** rule tells Traefik: *"Whenever someone visits `headlamp.kagea.site`, use the `headlamp-tls-cert` certificate for HTTPS and send the traffic to the `headlamp` Service on port `80`."*
* The **`headlamp` `ClusterIP` Service** then delivers that request internally over Kubernetes' private network to the running Headlamp container.



---

### Quick Summary of the Full Path

| Step | Component | Simple Role |
| --- | --- | --- |
| **1** | **DNS** | Translates `headlamp.kagea.site` to `34.2.137.230` |
| **2** | **GCP Load Balancer** | Receives public traffic on `34.2.137.230` and forwards it to healthy workers |
| **3** | **VPC Firewall** | Allows Google health checks on `8080` and public web traffic on `80`/`443` |
| **4** | **Traefik (`DaemonSet`)** | Listens on worker ports `80`/`443`, handles HTTPS, and routes by domain name |
| **5** | **`cert-manager`** | Proves domain ownership to Let's Encrypt and stores the free HTTPS certificate |
| **6** | **`Ingress` → `Service**` | Connects `headlamp.kagea.site` to the internal Headlamp pod |

Because **Traefik**, **`cert-manager`**, your **`letsencrypt-prod` `ClusterIssuer**`, and your **GCP Load Balancer (`34.2.137.230`)** are already configured cluster-wide, you do **not** need to reinstall anything or create a new load balancer. You can reuse the exact same `ClusterIssuer` across as many subdomains as you want.

To expose another web app on a new subdomain (for example, `myapp.kagea.site`), follow these three steps:

### 1. Point the New Subdomain's DNS `A` Record to `34.2.137.230`

Log in to your DNS provider for `kagea.site` and add a new **`A` record** pointing your new subdomain to your existing GCP Load Balancer IP:

* **Type:** `A`
* **Name / Host:** `myapp` *(for `myapp.kagea.site`)*
* **Value / IPv4 address:** `34.2.137.230`

**Verification:** Run `dig +short myapp.kagea.site @8.8.8.8` from your terminal and confirm it returns `34.2.137.230`.

---

### 2. Deploy Your App and Its `ClusterIP` `Service`

Traefik routes traffic to a Kubernetes `Service` inside the **same namespace** as the `Ingress` resource. If your app is not deployed yet, create a namespace (or use `default`) and deploy its `Deployment` and `ClusterIP` `Service`:

```bash
kubectl create namespace myapp

tee myapp-deploy.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: nginx:alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
EOF

kubectl apply -f myapp-deploy.yaml

```

*(Note: If your container listens on a different port such as `3000` or `8080`, set `targetPort` to that container port while keeping `port: 80` on the `Service`.)*

**Verification:** Run `kubectl get pods,svc -n myapp` and confirm the `myapp` pods are `1/1 Running` and `svc/myapp` is listed on `80/TCP`.

---

### 3. Create a New `Ingress` for the Subdomain

Create an `Ingress` in the **same namespace** (`myapp`) using the exact same structure as your Headlamp `Ingress`, changing only three fields:

1. **`spec.tls[].hosts` and `spec.rules[].host`:** Set both to your new subdomain (`myapp.kagea.site`).
2. **`spec.tls[].secretName`:** Use a **new unique Secret name** (`myapp-tls-cert`) so it does not collide with `headlamp-tls-cert`.
3. **`backend.service.name` and `port.number`:** Point to your new app's `Service` (`myapp` on port `80`).

```bash
tee myapp-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - myapp.kagea.site
      secretName: myapp-tls-cert
  rules:
    - host: myapp.kagea.site
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
EOF

kubectl apply -f myapp-ingress.yaml

```

**Verification:** Run `kubectl get certificate -n myapp -w` until `myapp-tls-cert` shows `READY: True` (usually within 30–60 seconds), then run `curl -I [https://myapp.kagea.site](https://myapp.kagea.site)` to confirm an `HTTP/2 200` response.
