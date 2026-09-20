# argocd-lab

Lab học ArgoCD (GitOps) trên cluster `docker-desktop`.

## Cấu trúc

```
apps/guestbook/   # manifest YAML thuần (Deployment + Service)
apps/web/         # nginx + ConfigMap chứa index.html
argocd/           # các Application CR
```

## Bước 1: Cho ArgoCD truy cập repo private

Repo dùng SSH URL, nên cần một deploy key:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/argocd-lab -N "" -C argocd-lab
# Thêm ~/.ssh/argocd-lab.pub vào GitHub: Settings > Deploy keys (read-only)

argocd login localhost:8080 --username admin --insecure \
  --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)"

argocd repo add git@github.com:ducpv-1208/argocd-lab.git \
  --ssh-private-key-path ~/.ssh/argocd-lab
```

(Cần `kubectl port-forward svc/argocd-server -n argocd 8080:443` chạy ở terminal khác.)

## Bước 2: Deploy app guestbook

```bash
kubectl apply -f argocd/guestbook.yaml
argocd app sync guestbook
kubectl get all -n guestbook
kubectl port-forward -n guestbook svc/guestbook 9898:80
```

## Bước 3: Thử GitOps

1. Đổi `replicas` trong `apps/guestbook/deployment.yaml`, commit, push, rồi xem app chuyển `OutOfSync`.
2. Bỏ comment `automated` trong `argocd/guestbook.yaml` và apply lại.
3. Chạy `kubectl scale deploy guestbook -n guestbook --replicas=5` và xem `selfHeal` revert.

## App 2: web (nginx + ConfigMap)

App này bật sẵn `automated` (prune + selfHeal), nên không cần sync tay.

```bash
kubectl apply -f argocd/web.yaml
kubectl port-forward -n web svc/web 8081:80   # mở http://localhost:8081
```

Thử: đổi `Version: v1` thành `v2` trong `apps/web/configmap.yaml`, commit và push.
ArgoCD tự sync ConfigMap. Kubelet cập nhật file trong pod sau khoảng 1 phút, không cần restart pod.

## Các bước tiếp theo

Helm, Kustomize (overlay dev/prod), App of Apps / ApplicationSet.
