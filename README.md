# FlowShip — Ops Repository

> Kubernetes 매니페스트 + ArgoCD GitOps 자동 배포

**FlowShip**은 Django 웹 서비스의 코드 변경부터 Kubernetes 배포까지 전 과정을 자동화하는 CI/CD 프로젝트입니다.  
이 레포는 Kubernetes 배포 매니페스트(YAML)를 관리하며, ArgoCD가 이 레포를 감시하여 변경 사항을 클러스터에 자동 반영합니다. 애플리케이션 소스코드는 [FlowShip-app](https://github.com/jeegle16-alt/FlowShip-app) 레포에서 관리됩니다.

---

## Architecture

### CI/CD 전체 흐름
<img width="933" height="248" alt="image" src="https://github.com/user-attachments/assets/20ddd5c0-fdd3-411c-b936-1e2fbba2789a" />

```
 Developer
    │ git push (코드 변경)
    ▼
 FlowShip-app ──▶ Jenkins ──▶ Docker Hub
                     │                │
                     │ image tag 업데이트   │ image pull
                     ▼                ▼
              FlowShip-ops ──▶ Kubernetes Cluster
              (이 레포)    ArgoCD
                           auto sync
```

### Kubernetes 애플리케이션 구조

<img width="986" height="170" alt="image" src="https://github.com/user-attachments/assets/82f25758-ca1b-4ca6-9af2-d341597a6bbb" />


---

## Manifest Structure

```
FlowShip-ops/
└── k8s/
    └── mysite/
        ├── namespace.yaml        # mysite 네임스페이스
        ├── mysql-secret.yaml     # DB 인증 정보 (Secret)
        ├── mysql-pv.yaml         # PersistentVolume (5Gi, hostPath)
        ├── mysql-pvc.yaml        # PersistentVolumeClaim
        ├── mysql-deploy.yaml     # MySQL Deployment + Headless Service
        ├── django-deploy.yaml    # Django Deployment + ClusterIP Service
        └── ingress.yaml          # Ingress (NGINX, pybo.mysite.local)
```

---

## Cluster Configuration

| Node | Role | IP | Workload |
|------|------|----|----------|
| `kube-control1` | Control Plane | 192.168.56.11 | Jenkins, ArgoCD |
| `kube-node1` | Worker (web) | 192.168.56.21 | Django Pods |
| `kube-node2` | Worker (web) | 192.168.56.22 | Django Pods |
| `kube-node3` | Worker (db) | 192.168.56.23 | MySQL Pod |

---

## Tech Stack

| 구분 | 기술 |
|------|------|
| Orchestration | Kubernetes 1.31 (kubespray) |
| GitOps CD | ArgoCD 2.11+ (Auto Sync, Self-Heal, Prune) |
| Ingress | NGINX Ingress Controller |
| Load Balancer | MetalLB (L2, 192.168.56.200–209) |
| Storage | PV/PVC (hostPath, 5Gi) |
| VM | Vagrant + VirtualBox (Ubuntu 22.04) |

---

## GitOps Workflow

이 레포의 YAML이 변경되면 ArgoCD가 자동으로 클러스터에 반영합니다.

### 자동 배포 (Jenkins → Ops Repo → ArgoCD)

1. 개발자가 FlowShip-app에 코드 push
2. Jenkins가 Docker 이미지 빌드 & Docker Hub push
3. Jenkins가 이 레포의 `django-deploy.yaml` 이미지 태그를 `sed`로 업데이트 후 push
4. ArgoCD가 변경 감지 → Kubernetes에 자동 배포

### 수동 스케일링

```bash
# Ops Repo에서 django-deploy.yaml 수정
spec:
  replicas: 3  # 2 → 3으로 변경

# git commit & push → ArgoCD 자동 반영
```

---

## Verified Tests

| 테스트 | 결과 |
|--------|------|
| GitOps 자동 배포 (Replica 2→3) | ArgoCD 자동 Sync, Pod 3개 정상 Running |
| End-to-End CI/CD (navbar 문구 변경) | Push → Jenkins → Docker Hub → Ops Repo → ArgoCD → 브라우저 반영 |
| Self-Healing (Pod 강제 삭제) | ~5초 내 새 Pod 자동 생성, Replica 수 유지 |

---

## ArgoCD Application 설정

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mysite-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/<your-username>/FlowShip-ops.git'
    targetRevision: main
    path: k8s/mysite
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: mysite
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## Related Repository

| Repo | 역할 |
|------|------|
| [**FlowShip-app**](https://github.com/jeegle16-alt/FlowShip-app) | 애플리케이션 소스코드 + CI (Jenkins) |
| **FlowShip-ops** (이 레포) | Kubernetes 매니페스트 + CD (ArgoCD) |

