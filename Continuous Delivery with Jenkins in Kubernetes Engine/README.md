> 쿠버네티스 엔진의 젠킨스와 지속적인 딜리버리 파이프라인을 구현하는 방법을 배웁니다.
> 
> 
> ![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/988009f9-e119-46ce-b26e-8ed07cc25e32)
> 

> **쿠버네티스 엔진이란?**
Kubernetes Engine은 컨테이너를 위한 강력한 클러스터 관리자 및 조정 시스템인 `Kubernetes`의 Google Cloud 호스팅 버전
쿠버네티스 앱은 실행을 위해 필요한 모든 종속 항목 및 라이브러리와 함께 번들로 제공되는 컨테이너로 빌드되기 때문에 안전하고 가용성이 높으며 경량이라 빠른 배포가 가능하다.
> 

> **젠킨스란?**
빌드, 테스트, 배포 파이프라인을 유연하게 조정할 수 있는 오픈소스 자동화 서버
> 

> **Continuous Delivery / Continuous Deployment란?**
배포 자동화
> 

> 젠킨스와 쿠버네티스 엔진으로 배포하여 CD 파이프라인을 설정하면 상당한 이점을 얻을 수 있다.
- 하나의 가상 호스트로 여러 운영체제에서 작업 (일시적 빌드 실행자)
- 빠른 시작
- 전역 부하 분산기 설치되어있음
> 

## ****Provisioning Jenkins****

```yaml
gcloud container clusters create jenkins-cd \
--num-nodes 2 \ 
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform" 
```

—scope에 추가한 매개 변수는 Jenkins가 Cloud Source Repositories 및 Google Container Registry에 액세스할 수 있도록 함

## ****Configure and Install Jenkins****

Helm : Kubernetes 애플리케이션을 쉽게 구성하고 배포할 수 있게 해주는 패키지 관리자 `helm repo add jenkins [https://charts.jenkins.io](https://charts.jenkins.io/)`

Helm CLI를 사용하여 해당 구성 설정으로 차트 배포

`helm install cd jenkins/jenkins -f jenkins/values.yaml --wait`

values 파일에서는 Kubernetes Cloud를 자동으로 구성하고 필수 플러그인을 추가해준다.

Jenkins 서비스 계정 구성

```yaml
kubectl create clusterrolebinding jenkins-deploy --clusterrole=cluster-admin --serviceaccount=default:cd-jenkins
```

Cloud Shell에서 Jenkins UI로의 포트 전달 설정

```yaml
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
```

![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/a98254f9-7500-4af9-8002-bc6075e07785)

## ****Connect to Jenkins****

(실패)

## ****Understanding the Application****

- **백엔드 모드**에서 gceme는 포트 8080을 수신 대기하고 Compute Engine 인스턴스 메타데이터를 JSON 형식으로 반환합니다.
- **프런트엔드 모드**에서 gceme는 백엔드 gceme 서비스를 쿼리하고 결과 JSON을 사용자 인터페이스에서 렌더링합니다.

![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/2173adba-9caf-44a7-8965-9d6f5deb69e2)

## ****Deploying the Application****

Production, Canary 두 개의 환경에 배포할 것이다.(Canary Developments 참고)

Kubernetes 네임스페이스를 만들어 배포를 격리하고 `kubectl apply -f k8s/production -n production` , 프로덕션, 카나리아, 서비스 배포를 생성한다.

Production 환경의 프론트엔드 확장

`kubectl scale deployment gceme-frontend-production -n production --replicas 4`

![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/e619e616-db59-4693-879d-eff95cd292aa)

frontend service loadbalancer IP 환경 변수에 저장

## ****Creating the Jenkins Pipeline****

gceme 샘플 앱의 복사본을 만들어서 Gloude Source Repository에 푸시함

`gcloud source repos create default`

다음 깃 초기세팅 → 작업사항 푸시

Jenkins에 사용자 인증 정보를 추가해야한다.(실패)

## ****Creating the development environment****

파이프라인이 정상적으로 작동하려면 Jenkinsfile을 수정하여 프로젝트 ID를 설정해야함

```
PROJECT = "REPLACE_WITH_YOUR_PROJECT_ID"
APP_NAME = "gceme"
FE_SVC_NAME = "${APP_NAME}-frontend"
CLUSTER = "jenkins-cd"
CLUSTER_ZONE = "us-east1-c"
IMAGE_TAG = "gcr.io/${PROJECT}/${APP_NAME}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
JENKINS_CRED = "${PROJECT}"
```

## ****Kick off Deployment****

깃에 푸시하기

## ****Deploying a canary release****

카나리아 브랜치 만들고 배포

## ****Deploying to production****

프로덕션 브랜치 만들고 배포
