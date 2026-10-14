# my-devops-config

[my-devops](https://github.com/Nakyungwon/my-devops) 의 배포 설정 리포. ArgoCD 는 이 리포만 봅니다.
코드·Dockerfile·CI 는 my-devops 에 있고, 그쪽 CI 가 이 리포의 `image.tag` 를 커밋합니다.

## 구조

```
chart/        Helm 차트. 차트는 한 벌, 환경별로 values 만 다름
  values.yaml       공통 (image.repo, resources)
  values-dev.yaml   my-devops ci-dev 가 image.tag 를 자동 갱신 → 자동 sync
  values-prod.yaml  my-devops ci-prod 가 dev 태그를 복사 → 수동 sync
argocd/       ArgoCD Application CR. Helm 차트가 아닌 일반 매니페스트
  app-dev.yaml   chart/ + values-dev  → ns dev,  automated
  app-prod.yaml  chart/ + values-prod → ns prod, 수동 sync
  istio-*.yaml   Istio 공식 Helm 차트(base/istiod/gateway)를 가리킴
bootstrap/    ArgoCD 설치 + argocd/ 등록 스크립트 (클러스터당 한 번)
```

환경은 브랜치가 아니라 values 파일로 나눕니다. dev/prod 모두 `main` 을 봅니다.

`argocd/` 가 `chart/` 밖에 있는 이유: 안에 두면 helm 이 이 파일까지
렌더링해서 ArgoCD가 자기 자신을 만들려다 깨집니다.

## 흐름

```
my-devops develop push (app/ 변경)
  → ci-dev: 이미지 빌드 → GHCR → 이 리포 values-dev.yaml tag 커밋
  → ArgoCD dev 자동 sync

my-devops develop → main merge
  → ci-prod: values-dev.yaml 의 tag 를 values-prod.yaml 에 복사 커밋
  → ArgoCD prod OutOfSync (여기서 멈춤)
  → 사람이 SYNC
```

my-devops CI 는 deploy key(이 리포에만 쓰기 권한)로 push 합니다.
키는 이 리포 Settings → Deploy keys, 비밀키는 my-devops 의 `CONFIG_REPO_DEPLOY_KEY` 시크릿.

## 자주 쓰는 명령

```bash
# prod 배포 (automated 가 없어서 직접 눌러야 합니다)
# syncOptions 를 operation 에 같이 안 넣으면 namespaces "prod" not found 로 실패합니다.
kubectl patch app web-prod-metadata -n argocd --type merge \
  -p '{"operation":{"sync":{"syncOptions":["CreateNamespace=true"]}}}'

# 롤백: values-prod.yaml 의 tag 를 이전 값으로 되돌리는 커밋 → SYNC
git log -p -- chart/values-prod.yaml

kubectl get app -n argocd
kubectl config current-context   # docker-desktop 이어야 함
```

## 알아둘 것

- **chart/templates 변경은 dev·prod 공통**입니다. main 에 올리면 dev 는 바로 나가고,
  prod 는 OutOfSync 로 대기했다가 다음 SYNC 때 태그 변경과 함께 나갑니다.
- **NodePort 는 클러스터 전역**: dev 30080 / prod 30081 이 겹치면 나중에 뜨는 쪽이 실패합니다.
- **valueFiles 쓰면 values.yaml 기본 포함이 꺼집니다**: Application 에 `values.yaml` 을 같이 나열해야 합니다.
- **selfHeal 은 dev 만**: `kubectl edit` 로 바꿔도 git 상태로 되돌아갑니다.

## 재구축

```bash
./bootstrap/install.sh
```
