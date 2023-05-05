# Orchestrating the Cloud with Kubernetes
## Pods
- 한 개 이상의 컨테이너 + 볼륨(데이터 디스크) set 
- 상호 의존성이 높은 컨테이너가 여러 개 있으면 하나의 포드로 패키징
- 한 포드 안의 컨테이너들은 서로 통신할 수 있으며 볼륨을 공유한다. 네트워크 네임 스페이스도 공유하므로 한 포드 당 IP 주소를 하나씩 가지고 있다.
- 볼륨은 포드가 존재하는 한 계속해서 존재한다.
- 포드 만들기
  - 모놀리식 포드 구성 파일 살펴보기
```
#pods/monolith.yaml
apiVersion: v1
kind: Pod
metadata:
  name: monolith
  labels:
    app: monolith
spec:
  containers:     # 포드는 하나의 컨테이너(모놀리식)으로 구성이 되어있다.
    - name: monolith
      image: kelseyhightower/monolith:1.0.0
      args:
        - "-http=0.0.0.0:80"
        - "-health=0.0.0.0:81"
        - "-secret=secret"      # 시작할 때 컨테이너로 몇 사기 인수가 전달된다.
      ports:
        - name: http
          containerPort: 80   # HTTP 트래픽용 포드 80이 개방됨
        - name: health
          containerPort: 81
      resources:
        limits:
          cpu: 0.2
          memory: "10Mi"
```
  - `kubectl create -f pods/monolith.yaml`로 monolith pod를 생성한다.
  - `kubectl describe pods monolith`로 상세 보기 가능

## Interacting with pods
- `kubectl port-forward monolith 10080:80` 로 컨테이너의 80포트를 10080으로 노출시킨다.
- `curl http://127.0.0.1:10080`는 "hello"를 응답한다.
- `curl http://127.0.0.1:10080/secure`는 authorization failed가 발생한다. 따라서 `curl -u user http://127.0.0.1:10080/login`로 로그인을 진행한다. `TOKEN=$(curl http://127.0.0.1:10080/login -u user|jq -r '.token')`로 토큰 값을 저장하여 `curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:10080/secure`를 수행하면 'hello'를 다시 볼 수 있다.
-  `kubectl exec monolith --stdin --tty -c monolith -- /bin/sh`로 monolith pod의 쉘에 접속한다. `ping -c 3 google.com`등을 테스트 해 볼 수 있다.

## Services
- 포드는 다양한 이유로 중지되거나 시작할 수 있으며 이에 따라 IP 주소가 변경될 수도 있어서 안정적이지 않다.
- 서비스는 포드를 위해 안정적인 엔드포인트를 제공한다.
- 서비스는 라벨을 사용하여 어떤 포드에서 작동할지 결정한다. 
- 서비스 만들기
  - https 트래픽 처리 가능한 보안 포드 만들기
    `kubectl create secret generic tls-certs --from-file tls/`
    `kubectl create configmap nginx-proxy-conf --from-file nginx/proxy.conf`
    `kubectl create -f pods/secure-monolith.yaml`
   - 모놀리식 서비스 만들기 `kubectl create -f services/monolith.yaml`
   - GCP 방홤벽 규칙 생성 `gcloud compute firewall-rules create allow-monolith-nodeport \ --allow=tcp:31000` 31000 포트의 NodePort 타입의 서비스가 접근할 수 있도록 하는 방화벽 규칙 'allow-monolith-nodeport' 생성

## Adding labels to pods
- 라벨 추가 `kubectl label pods secure-monolith 'secure=enabled'
- 엔드포인트 목록 확인 `kubectl describe services monolith | grep Endpoints`

## Deploying applications with Kubernetes
- 디플로이먼트 : 실행중인 포드의 개수가 사용자가 명시한 포드 개수와 동일하게 만드는 선언적 방식
- 포드가 중단되더라도 새로운 포드를 만들어 다시 실행한다. 재시작을 담당하여 처리한다.
- auth 디플로이먼트 구성 파일 검토
```
# cat deployments/auth.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth
spec:
  selector:
    matchlabels:
      app: auth
  replicas: 1 # 디플로이먼트는 복제본 한 개를 만든다.
  template:
    metadata:
      labels:
        app: auth
        track: stable
    spec:
      containers:
        - name: auth
          image: "kelseyhightower/auth:2.0.0"
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
 ```
 - kubectl create 명령어를 사용하여 auth, hello, frontend의 디플로이먼트와 .
 - 
