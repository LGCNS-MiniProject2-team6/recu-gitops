# Argo CD Bootstrap

이 디렉터리는 K3s 클러스터에 설치할 Argo CD 표준 비고가용성 구성을 관리한다.

- 버전: `v3.5.7`
- 설치 Namespace: `argocd`
- 배치 노드: `workload=platform`, `environment=prod`
- UI 공개 방식: 외부 Ingress 없이 SSH 터널 또는 `kubectl port-forward`만 사용
- GitOps 배포 범위: `recu` AppProject가 허용한 dev/prod 앱 6개, ConfigMap, Service, Deployment

설치 후에는 아래 순서로 적용한다.

```bash
kubectl apply -k argocd/install
kubectl apply -k argocd/applications
```

`argocd/applications`는 Argo CD CRD가 먼저 설치된 뒤 적용해야 한다.
