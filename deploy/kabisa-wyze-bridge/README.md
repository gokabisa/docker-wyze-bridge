# kabisa-wyze-bridge Helm chart

Deploys [docker-wyze-bridge](https://github.com/mrlt8/docker-wyze-bridge) to the
Kabisa Kubernetes cluster (arm64 Kapsule). Image is built from this fork via
`.github/workflows/kabisa-deploy.yml` and pushed to
`ghcr.io/gokabisa/wyze-bridge`.

## Prerequisites

- `kubectl` pointed at the target cluster (`admin@k8s-infrastructure`)
- `helm` 3.x
- Namespace `tools` exists
- Secrets in `tools`:
  - `wyze-bridge-credentials` — `WYZE_EMAIL`, `WYZE_PASSWORD`, `API_ID`, `API_KEY`
  - `wyze-bridge-webauth` — `WB_USERNAME`, `WB_PASSWORD`, `WB_IP` (Traefik LB IP, used as WebRTC ICE host)
  - `regcred` — GHCR pull secret (already provisioned cluster-wide; no action needed)
- DNS `cameras.k8s.gokabisa.com` resolves to the Traefik LoadBalancer
- Image `ghcr.io/gokabisa/wyze-bridge:<tag>` published (run the build workflow)

## One-time secret creation

```bash
kubectl create secret generic wyze-bridge-credentials \
  -n tools \
  --from-literal=WYZE_EMAIL=<email> \
  --from-literal=WYZE_PASSWORD=<password> \
  --from-literal=API_ID=<api-id> \
  --from-literal=API_KEY=<api-key>

kubectl create secret generic wyze-bridge-webauth \
  -n tools \
  --from-literal=WB_USERNAME=<webui-user> \
  --from-literal=WB_PASSWORD=<webui-password> \
  --from-literal=WB_IP=51.159.74.192
```

API ID and key come from the [Wyze developer portal](https://developer-api-console.wyze.com/#/apikey/view).

## Deploy

```bash
cd deploy/kabisa-wyze-bridge
helm upgrade --install wyze-bridge . \
  -n tools \
  -f ../values/prod/kabisa-wyze-bridge.values.yaml \
  --set image.tag=<tag> \
  --timeout 5m
```

A 1Gi RWO PersistentVolumeClaim is provisioned on the cluster default
StorageClass to persist `/tokens` (Wyze auth cookies) and `/img` (camera
thumbnails) across pod restarts.

## Verify

```bash
kubectl get pods -n tools -l app.kubernetes.io/name=kabisa-wyze-bridge
kubectl logs -n tools -l app.kubernetes.io/name=kabisa-wyze-bridge --tail=100
curl -sI https://cameras.k8s.gokabisa.com/
```

The Traefik IngressRoute (`cameras.k8s.gokabisa.com`) lives in the
`gokabisa/ingress-traefik` repo and is applied separately.

## Upgrade

1. Bump `appVersion` in `Chart.yaml`
2. Run the build workflow with the new tag (or push to `main`)
3. `helm upgrade --install wyze-bridge . -n tools -f ../values/prod/kabisa-wyze-bridge.values.yaml --set image.tag=<new-tag>`

## Notes

- WebUI (port 5000) is the only port exposed via the Service. RTSP (8554),
  RTMP (1935), and WebRTC (8889/8189) are unreachable outside the pod. Browser
  playback uses HLS over the same Flask port — works through Traefik.
- WebRTC low-latency playback will not work externally (no UDP LB, no `WB_IP`).
- `LLHLS=false` keeps HLS in classic mode (avoids cert plumbing for
  Low-Latency HLS).
- The container runs as root because mediamtx, ffmpeg, and the Flask app all
  write to in-image paths (`/app`, `/tokens`, `/img`). Capability `ALL` is
  dropped and privilege escalation disabled.
