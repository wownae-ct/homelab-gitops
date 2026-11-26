> 공개용으로 정리한 저장소입니다. AWS 계정 ID, 사설 IP, 개인 도메인은 문서용 값으로 바꿨습니다(123456789012, 192.0.2.x 등, example.internal). 실제 운영 설정은 비공개 저장소에서 관리합니다.

# homelab-gitops

홈랩 Kubernetes HA 클러스터를 **코드로 만들고(IaC), Git 커밋만으로 배포·복구(GitOps)** 하는 구성 저장소입니다. 서버 구축은 Ansible, 앱 배포는 ArgoCD, 빌드는 Jenkins, 관측은 Prometheus·Loki·Grafana가 맡습니다.

## 요약

| 영역 | 내용 | 위치 |
|---|---|---|
| IaC | 서버·클러스터를 21개 단계별 플레이북으로 처음부터 재구축. VM 이미지를 복사하지 않음 | `ansible/` |
| GitOps | `root-app.yaml` 하나로 App of Apps 시작, 하위 앱은 `prune`·`selfHeal` 자동 동기화 | `root-app.yaml`, `apps/*-app.yaml` |
| CI/CD | Jenkins(Kaniko)가 이미지를 빌드해 이 저장소의 이미지 태그를 커밋하면 ArgoCD가 배포 | `apps/jenkins/`, `apps/portfolio-web/` |
| 무중단 배포·롤백 | 롤링 업데이트(`maxUnavailable: 0`, `maxSurge: 1`), HPA 2~5개(CPU 50%), 헬스 체크 실패 시 해당 커밋을 `git revert`해 이전 이미지로 복귀 | `apps/portfolio-web/` |
| 모니터링·알림 | Prometheus 메트릭, Loki·Promtail 로그, Grafana 대시보드 4종, `PortfolioWeb5xxHigh` 규칙, Alertmanager → SNS 메일 | `apps/prometheus/`, `apps/loki/`, `apps/promtail/`, `apps/grafana/` |
| 고가용성 | control-plane HA(keepalived·HAProxy), Cilium, MetalLB, Istio 게이트웨이 | `ansible/roles/k8s`, `metallb`, `istio` |
| 스토리지·백업 | Longhorn(원격 오브젝트 스토리지 백업), MinIO | `apps/longhorn/`, `apps/minio/` |
| 검증 자동화 | 인프라 검증 25개(`verify-phase.yml`), 최종 E2E·장애 시나리오·모니터링 검증(`verify-final.yml`) | `ansible/playbooks/` |
| 시크릿 | 저장소에는 암호화된 Sealed Secrets와 ansible-vault 변수만 두는 방식 | 아래 참고 |

## 동작 방식

1. 서버·클러스터: `ansible/playbooks/`의 00~20번을 순서대로 실행하면 bastion, storage, DB, Kubernetes HA, Cilium, Sealed Secrets, MetalLB, Longhorn, Istio, CoreDNS, ArgoCD, kube-prometheus-stack까지 구성됩니다.
2. 18번 플레이북이 `root-app.yaml`만 적용하면 ArgoCD가 `apps/`를 읽어 나머지 앱을 만듭니다.
3. 앱 배포: Jenkins 파이프라인이 이미지를 빌드해 `apps/portfolio-web/values.yaml`의 태그를 커밋하고, ArgoCD가 변경을 동기화합니다. 배포 대상 앱과 파이프라인은 [jiong-minihome](https://github.com/wownae-ct/jiong-minihome)에 있습니다.
4. 장애 시: 헬스 체크가 실패하면 태그 커밋을 `git revert`해 이전 이미지로 되돌립니다. 서비스 경로 전환(컷오버) 롤백 절차는 `docs/cutover/`에 있습니다.

## 구조

```
root-app.yaml                         # App of Apps 진입점
apps/
  argocd, istio-gateway, kiali        # 진입·서비스 메시
  portfolio-web                       # 배포 차트(replica 2, 롤링 업데이트, HPA)
  jenkins                             # CI(Kaniko 빌드)
  prometheus, loki, promtail, grafana # 메트릭·로그·대시보드, Alertmanager 규칙
  longhorn, minio                     # 스토리지
ansible/                              # 클러스터·서버 구성 플레이북(ansible/README.md 참고)
docs/                                 # 컷오버 검증·롤백 문서
```

## 시크릿

- 비밀번호·토큰·키는 저장소에 평문으로 두지 않습니다. 클러스터 쪽은 Sealed Secrets, Ansible 쪽은 `vault_*` 변수를 ansible-vault로 암호화해 씁니다.
- 공개본에서는 Sealed Secret 5개 파일을 제외했습니다. 그대로 배포할 수 없고, 사용하려면 자신의 값과 시크릿을 채워야 합니다.
- `*.secret.yaml`, `*-credentials.yaml`, `.env*`는 `.gitignore`로 막아 둡니다.

## 관련 저장소

- [homelab-terraform](https://github.com/wownae-ct/homelab-terraform): Proxmox·AWS·Cloudflare를 정의한 Terraform
- [jiong-minihome](https://github.com/wownae-ct/jiong-minihome): 배포 대상 앱과 Jenkins 파이프라인
- [k3s-repo](https://github.com/wownae-ct/k3s-repo): cortex-agent용 k3s 클러스터의 GitOps 매니페스트
