# Tutorial: from an empty machine to a running app

Stands up the stack on a fresh single-node cluster: k3s, the SealedSecrets
controller, Argo CD, then the app and its data services synced from your fork.
Ends at a `200` from `/healthz`.

Needs a Linux machine with `sudo` and a GitHub account.

## 1. Fork and clone

Fork this repo, then clone your fork:

```sh
git clone https://github.com/<your-user>/<your-fork>.git
cd <your-fork>
```

## 2. Install k3s

On the machine:

```sh
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes        # node shows Ready
```

k3s ships Traefik as the ingress controller.

Copy `/etc/rancher/k3s/k3s.yaml` to `~/.kube/config` on your workstation,
replace `127.0.0.1` with the machine's IP, then:

```sh
kubectl get nodes
```

## 3. Install the SealedSecrets controller

```sh
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml
kubectl rollout status -n kube-system deployment/sealed-secrets-controller
brew install kubeseal        # see the releases page for Linux
```

## 4. Install Argo CD

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status -n argocd deployment/argocd-server
```

Get the admin password and open the UI:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

Log in at `https://localhost:8080` as `admin`. Leave the port-forward running.

## 5. Seal the secrets

Set real values in each `secret.example.yaml`, then seal:

```sh
cd manifests/app
# edit secret.example.yaml, postgres/secret.example.yaml,
# postgres/backup-secret.example.yaml, meilisearch/secret.example.yaml
./seal-secrets.sh
./seal-minio.sh
```

Each script writes an encrypted `sealedsecret.yaml`. Commit them:

```sh
cd ../..
git add manifests/app/**/sealedsecret.yaml manifests/app/sealedsecret.yaml
git commit -m "Add sealed secrets"
git push
```

## 6. Point the manifests at your fork

Replace `https://github.com/your-org/your-repo.git` in:

```text
apps/app.yaml
apps/infrastructure.yaml
infrastructure/argocd/appproject.yaml
```

```sh
git commit -am "Point Argo at this fork"
git push
```

## 7. Apply the Applications

```sh
kubectl apply -f infrastructure/argocd/appproject.yaml
kubectl apply -f apps/
kubectl get applications -n argocd        # app, infrastructure -> Synced/Healthy
```

## 8. Verify

```sh
kubectl get pods -n app
kubectl port-forward -n app svc/app 8088:80
curl -i http://localhost:8088/healthz     # 200
```

## Next steps

- Hostname: set it in `manifests/app/ingress.yaml` and
  `infrastructure/cloudflared/configmap.yaml`, create a Cloudflare Tunnel, seal
  its credentials as in step 5.
- Backups: build an image with `pg_dump`, `gzip`, `gpg`, `rclone`, set it in
  `manifests/app/postgres/backup-cronjob.yaml`, seal the backup secret.
