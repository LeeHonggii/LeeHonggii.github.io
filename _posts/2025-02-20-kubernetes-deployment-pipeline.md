---
title: "Bitbucket + NCloud Kubernetes 배포 파이프라인 구축기"
date: 2025-02-20 09:00:00 +0900
categories: [Study, DevOps]
tags: [kubernetes, docker, bitbucket-pipelines, cicd, ncloud]
---

## 들어가며

회사에서 처음으로 배포 파이프라인을 접했을 때, 신기한 점이 있었다.

**git push 한 번으로 운영 서버에 자동 배포가 됐다.**

```bash
git push origin master  # 이거 하나로 운영 배포 완료
```

처음엔 "어떻게 이게 가능하지?" 싶었는데, 코드를 분석해보니 **Bitbucket Pipelines + Kubernetes** 조합이었다.

---

## 배포 흐름 이해하기

회사 배포 파이프라인을 분석해보니 이런 흐름이었다.

```
1. master/dev에 푸시
      ↓
2. Bitbucket Pipeline 자동 실행
      ↓
3. Docker 이미지 빌드 (Dockerfile)
      ↓
4. NCloud Registry에 Push
      ↓
5. kube-deployment.yml 이미지 태그 치환
      ↓
6. kubectl apply (K8s 배포)
      ↓
7. Rolling Update 완료
```

**마치 컨베이어 벨트처럼**: 코드 푸시하면 자동으로 빌드 → 이미지 생성 → 배포까지 쭉 흘러간다.

---

## 파일 구조

회사 프로젝트의 배포 관련 파일들을 정리해봤다.

| 파일 | 역할 |
|------|------|
| `Dockerfile` | Python 앱을 컨테이너 이미지로 만듦 |
| `bitbucket-pipelines.yml` | CI/CD 파이프라인 정의 |
| `kube-deployment.yml` | 운영 K8s 배포 설정 (replicas: 2) |
| `kube-dev-deployment.yml` | 개발 K8s 배포 설정 (replicas: 1) |
| `config-map.yaml` | 환경변수 정의 (DB, API 키 등) |

---

## Dockerfile 분석

회사 Dockerfile은 놀랍도록 단순했다.

```dockerfile
FROM python:3.13.1-slim-bullseye
USER root

WORKDIR /app/
ADD . /app/

RUN pip install -r /app/requirements.txt

CMD ["python", "app.py"]
```

**핵심 포인트:**
- `slim-bullseye`: 경량 이미지 (용량 절약)
- `ADD . /app/`: 전체 코드 복사
- `CMD`: 컨테이너 시작 시 실행할 명령

**왜 이렇게 단순할까?**
- 환경변수는 ConfigMap으로 분리
- 복잡한 설정은 K8s에서 처리
- Dockerfile은 "이미지 만들기"에만 집중

---

## Bitbucket Pipelines 분석

CI/CD의 핵심인 `bitbucket-pipelines.yml`을 분석해봤다.

### 전체 구조

```yaml
image: atlassian/default-image:4

options:
  docker: true
  size: 2x

pipelines:
  branches:
    master:
      - step: Docker build & push
      - step: Deploy to K8s
    dev:
      - step: Docker build & push
      - step: Deploy to K8s (dev)
```

**브랜치별 자동 분기:**
- `master` 푸시 → 운영 배포
- `dev` 푸시 → 개발 배포

---

### Step 1: Docker 빌드 & 푸시

```yaml
- step:
    name: Docker build & push
    script:
      # 1. 이미지 이름 설정 (커밋 해시로 태그)
      - export IMAGE_NAME=$NCLOUD_CR_URL/$APPLICATION_NAME:$BITBUCKET_COMMIT

      # 2. Docker 빌드
      - docker build -t $APPLICATION_NAME .

      # 3. 태그 지정
      - docker tag $APPLICATION_NAME $IMAGE_NAME

      # 4. Registry 로그인
      - echo "$NCLOUD_KEY" | docker login -u $NCLOUD_ID $NCLOUD_CR_URL --password-stdin

      # 5. 이미지 푸시
      - docker push $IMAGE_NAME
```

**핵심:**
- `$BITBUCKET_COMMIT`: 커밋 해시를 이미지 태그로 사용
- 매번 고유한 이미지 태그 생성 → 롤백 쉬움

---

### Step 2: Kubernetes 배포

```yaml
- step:
    name: Deploy
    deployment: production
    script:
      # 1. 이미지 URL 치환
      - sed -i "s|{{image}}|$NCLOUD_PRI_URL/$APPLICATION_NAME:$BITBUCKET_COMMIT|g" kube-deployment.yml

      # 2. kubectl apply
      - pipe: atlassian/kubectl-run:1.1.2
        variables:
          KUBE_CONFIG: $KUBE_CONFIG
          KUBECTL_COMMAND: 'apply'
          RESOURCE_PATH: 'kube-deployment.yml'
```

**핵심:**
- `sed`로 `{{image}}` 플레이스홀더를 실제 이미지 URL로 치환
- `kubectl apply`로 K8s에 배포

---

## Kubernetes 배포 파일 분석

### Service 정의

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: my-namespace
  name: my-app
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
```

**역할:**
- Pod들을 하나의 엔드포인트로 묶음
- `NodePort`: 외부에서 접근 가능

---

### Deployment 정의

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: my-namespace
  name: my-app
spec:
  replicas: 2  # Pod 2개 운영
  selector:
    matchLabels:
      app: my-app
  template:
    spec:
      containers:
        - name: my-app
          image: {{image}}  # 파이프라인에서 치환됨
          envFrom:
            - configMapRef:
                name: my-app-config
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 60
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 60
      imagePullSecrets:
        - name: regcred
```

**핵심 포인트:**

**1. replicas**
```yaml
replicas: 2  # 운영은 2개, 개발은 1개
```
- 운영: 고가용성을 위해 2개 이상
- 개발: 비용 절약을 위해 1개

**2. envFrom + ConfigMap**
```yaml
envFrom:
  - configMapRef:
      name: my-app-config
```
- 환경변수를 ConfigMap에서 주입
- 코드와 설정 분리

**3. livenessProbe & readinessProbe**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 80
```
- `liveness`: Pod가 살아있는지 체크 (죽으면 재시작)
- `readiness`: 트래픽 받을 준비 됐는지 체크

**4. imagePullSecrets**
```yaml
imagePullSecrets:
  - name: regcred
```
- Private Registry 인증 정보
- 미리 Secret으로 등록해둬야 함

---

## ConfigMap: 환경변수 관리

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  app_mode: "prod"
  app_workers: "5"
  db_host: "your-db-host"
  db_port: "3306"
  db_user: "your-db-user"
  db_passwd: "your-db-password"
  db_name: "your-db-name"
  db_pool_size: "3"
  api_key: "your-api-key"
  TZ: "Asia/Seoul"
```

**왜 ConfigMap?**
- 환경별 설정 분리 (운영/개발)
- 코드 변경 없이 설정 변경 가능
- 민감 정보는 Secret으로 분리 권장

---

## 운영 vs 개발 환경 비교

회사에서는 환경별로 설정을 분리했다.

| 항목 | 운영 (master) | 개발 (dev) |
|------|--------------|-----------|
| 배포 파일 | `kube-deployment.yml` | `kube-dev-deployment.yml` |
| Service 이름 | `my-app` | `dev-my-app` |
| replicas | 2 | 1 |
| ConfigMap | `my-app-config` | `dev-my-app-config` |
| workers | 5 | 1 |
| db_pool_size | 3 | 1 |

**분리 이유:**
- 운영: 안정성, 고가용성 중시
- 개발: 빠른 배포, 비용 절약 중시

---

## Bitbucket 변수 설정

파이프라인에서 사용하는 변수들.

| 변수명 | 레벨 | 설명 |
|--------|------|------|
| `NCLOUD_CR_URL` | Workspace | Container Registry URL |
| `NCLOUD_PRI_URL` | Workspace | Private Registry URL |
| `NCLOUD_ID` | Workspace | NCloud Access Key |
| `NCLOUD_KEY` | Workspace | NCloud Secret Key |
| `KUBE_CONFIG` | Workspace | K8s 클러스터 접속 정보 (base64) |
| `APPLICATION_NAME` | Repository | 앱 이름 |
| `BITBUCKET_COMMIT` | 자동 | 커밋 해시 |

**Workspace vs Repository:**
- Workspace: 회사 공통 (Registry, K8s 접속 정보)
- Repository: 프로젝트별 (앱 이름)

---

## 새 프로젝트 배포 체크리스트

회사에서 새 프로젝트 배포할 때 따르는 순서.

```
□ 1. Bitbucket Repository 생성
□ 2. APPLICATION_NAME 변수 설정 (Repository 레벨)
□ 3. 템플릿 파일 복사
     - Dockerfile
     - bitbucket-pipelines.yml
     - kube-deployment.yml
     - kube-dev-deployment.yml
     - config-map.yaml
□ 4. 프로젝트명/namespace/configMap 이름 수정
□ 5. ConfigMap 먼저 apply
     - kubectl apply -f config-map.yaml
□ 6. 코드 푸시 → 자동 배포
□ 7. 배포 확인
     - kubectl get pods -n my-namespace
```

---

## 배운 점 정리

1. **파이프라인 = 자동화**
   - git push 한 번으로 빌드 → 배포
   - 수동 작업 최소화

2. **이미지 태그 = 커밋 해시**
   - 매 배포마다 고유한 태그
   - 롤백이 쉬움

3. **환경 분리**
   - 운영/개발 설정 파일 분리
   - ConfigMap으로 환경변수 관리

4. **K8s Probe**
   - liveness: 죽은 Pod 재시작
   - readiness: 준비된 Pod만 트래픽 받음

5. **변수 레벨 분리**
   - 공통(Workspace) vs 프로젝트별(Repository)

---

## 체크리스트

K8s 배포 파이프라인 구축 시:

**Dockerfile**
- [ ] 경량 베이스 이미지 사용
- [ ] 불필요한 파일 제외 (.dockerignore)

**Pipeline**
- [ ] 브랜치별 배포 분기 (master/dev)
- [ ] 이미지 태그에 커밋 해시 사용
- [ ] Registry 인증 설정

**Kubernetes**
- [ ] 운영/개발 Deployment 파일 분리
- [ ] replicas 환경별 설정
- [ ] ConfigMap으로 환경변수 분리
- [ ] liveness/readiness Probe 설정
- [ ] imagePullSecrets 설정

**보안**
- [ ] 민감 정보는 Secret 사용 권장
- [ ] ConfigMap에 평문 비밀번호 지양

---

## 참고

- 민감 정보(DB 비밀번호, API 키)는 ConfigMap 대신 **K8s Secret**으로 분리하는 것이 보안상 좋음
- Secret은 base64 인코딩되어 저장되고, RBAC으로 접근 제어 가능
