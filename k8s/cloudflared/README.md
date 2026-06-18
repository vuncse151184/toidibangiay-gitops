# cloudflared — Named Tunnel cho `api.toidibangiay.shop`

Phơi API Gateway (`gateway` Service, port 4000) ra Internet qua một **named**
Cloudflare Tunnel với hostname **cố định** `api.toidibangiay.shop`.

> Thay thế quick tunnel cũ (`cloudflared tunnel --url ...`) vốn sinh hostname
> `*.trycloudflare.com` ngẫu nhiên, đổi mỗi lần pod restart → FE gọi API lỗi.

Manifest ở repo GitOps này được **ArgoCD auto-sync**. Riêng credentials là Secret,
tạo thủ công trong cluster, **không commit**.

## Yêu cầu

- `cloudflared` CLI cài ở máy local
- Quyền vào tài khoản Cloudflare đang quản `toidibangiay.shop`
- `kubectl` trỏ đúng cluster EKS (`toidibangiay-prod`)

## Các bước (chạy 1 lần)

```bash
# 1) Đăng nhập + tạo named tunnel
cloudflared tunnel login
cloudflared tunnel create toidibangiay-api
#   → in ra <TUNNEL_ID> và file creds: ~/.cloudflared/<TUNNEL_ID>.json

# 2) Điền <TUNNEL_ID> vào configmap.yaml  (dòng `tunnel: REPLACE_WITH_TUNNEL_ID`)

# 3) Tạo Secret credentials trong cluster (KHÔNG commit)
kubectl create secret generic cloudflared-credentials \
  --from-file=credentials.json="$HOME/.cloudflared/<TUNNEL_ID>.json" \
  -n toidibangiay-prod

# 4) Trỏ DNS — tự tạo CNAME `api` → <TUNNEL_ID>.cfargotunnel.com (proxied) trên Cloudflare
cloudflared tunnel route dns toidibangiay-api api.toidibangiay.shop

# 5) Deploy: commit + push repo GitOps này → ArgoCD sync.
#    (hoặc apply tay từ gốc repo:  kubectl apply -k k8s/)
```

## Kiểm tra

```bash
kubectl -n toidibangiay-prod rollout status deploy/cloudflared-tunnel
curl -i https://api.toidibangiay.shop/api/docs        # Swagger → 200 là OK
curl -i https://api.toidibangiay.shop/api/collections # phải trả JSON, không 5xx
```

## Cập nhật Frontend (Vercel)

Đặt env cho cả **Production** và **Preview**, rồi redeploy FE:

```
BACKEND_API_URL=https://api.toidibangiay.shop/api
NEXT_PUBLIC_BACKEND_API_URL=https://api.toidibangiay.shop/api
```

CORS ở gateway đã allow sẵn `https://toidibangiay.shop` và
`https://toidibangiay.vercel.app` (`backend/apps/gateway/src/main.ts`).

## Ghi chú

- Deployment tên `cloudflared-tunnel` — trùng tên quick tunnel cũ nên **thay thế**
  nó, không tạo trùng. Nếu quick tunnel cũ apply tay từ file khác, `kubectl apply`
  sẽ cập nhật in-place.
- Đổi `tunnel:` trong [configmap.yaml](configmap.yaml) là cách duy nhất gắn tunnel ID;
  credentials đến từ Secret `cloudflared-credentials` (mount tại `/etc/cloudflared/creds`).
- Cloudflare lo TLS đầu vào — không cần ACM cert như hướng ALB Ingress.
