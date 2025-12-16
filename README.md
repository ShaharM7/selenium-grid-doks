# Selenium Grid on DigitalOcean Kubernetes (DOKS) — GitHub Actions Deployment

This repository demonstrates how to deploy **Selenium Grid** to **DigitalOcean Kubernetes (DOKS)** using **GitHub Actions** for CI/CD. The deployment exposes the Selenium Router externally using a Kubernetes **Service of type `LoadBalancer`**, enabling stable access via a public IP and (optionally) a DNS name.

## What I built

- A Kubernetes cluster on **DigitalOcean (DOKS)**
- **Selenium Grid** deployed via **Helm**
- External access via **Kubernetes LoadBalancer** (router service)
- Optional **DNS mapping** to the LoadBalancer external IP
- Automated deploy/upgrade triggered from **GitHub Actions**

## Architecture (high level)

1. GitHub Actions runs a deployment workflow on push/manual trigger.
2. The workflow authenticates to the DOKS cluster (`kubectl` access).
3. Helm installs/upgrades Selenium Grid in a dedicated namespace.
4. The Selenium Router service is exposed with `type: LoadBalancer`.
5. (Optional) A domain/subdomain points to the LoadBalancer external IP.

## Kubernetes configuration highlights

### Router service exposed via LoadBalancer

In the chart values (router section), the service is configured as:

```yaml
router:
  serviceType: LoadBalancer
  loadBalancerIP: ""
  serviceAnnotations: {}
```

> Note: The external IP is allocated by DigitalOcean when the Service is created/updated. It typically stays stable as long as you do not delete/recreate the Service/Load Balancer resource.

## Deploy with GitHub Actions

The workflow typically performs:

- Checkout repository
- Install/verify `kubectl` and `helm`
- Authenticate to the cluster (kubeconfig or token-based approach)
- `helm upgrade --install` the Selenium Grid release

Conceptual commands:

```bash
kubectl get nodes
helm repo add <repo> <url>
helm repo update
helm upgrade --install selenium-grid <chart> -n selenium-grid --create-namespace -f values.yaml
kubectl -n selenium-grid get svc selenium-router -o wide
```

## Access Selenium Grid

### A) Public IP (LoadBalancer)

Get the external IP:

```bash
kubectl -n selenium-grid get svc selenium-router -o wide
```

Then open:

- `http://<EXTERNAL-IP>:4444/`

### B) DNS name (optional)

Create an **A record** pointing your hostname to the router LoadBalancer external IP.

Example:

- Host: `grid`
- Domain: `selenium-grid-doks-srefaeli.com`
- Points to: `<EXTERNAL-IP>`

Then open:

- `http://grid.selenium-grid-doks-srefaeli.com:4444/`

> DNS maps hostname → IP only. If Selenium is exposed on port `4444`, you must keep `:4444` in the URL unless you add a reverse proxy or expose the service on 80/443.

### C) Local debugging (port-forward)

```bash
kubectl -n selenium-grid port-forward svc/selenium-router 4444:4444
```

Open:

- `http://localhost:4444/`

## Troubleshooting

- **`EXTERNAL-IP` is `<pending>`**
  - Wait and re-check the service.
  - Confirm DigitalOcean can provision a load balancer for your cluster/project.

- **DNS does not resolve**
  - Ensure the domain’s **nameservers** are configured correctly at the registrar (if using DigitalOcean DNS, use DO nameservers).
  - Validate with `nslookup` / `dig`.

- **Port-forward works but public URL does not**
  - DNS may point to the wrong IP
  - Firewall rules may block inbound traffic
  - LoadBalancer may not be exposing the expected port(s)

## Security note

Selenium Grid should not be left publicly accessible without restrictions. Consider:
- Cloud firewall rules (limit source IPs)
- Authentication and TLS via an ingress/reverse proxy
- Short-lived environments for demos/interviews

---

If you want, add your real workflow file under `.github/workflows/` and update this README with exact secret names and commands used in your pipeline.
