# **Managing Deployments Using Kubernetes Engine**

## ****Task 1. Learn about the deployment object****

배포 객체에 대해 알기 kubectl explain 

`kubectl explain <배포 객체 이름>`

기타 옵션들

[Kubernetes - kubectl explain](https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl_explain/)

## ****Task 2. Create a deployment****

`deployments/auth.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment #배포 객체 종류
metadata:
  name: auth
spec:
  replicas: 1 # 복제본 몇 개 만들 것인지 이 필드 값을 조절하여 Pods의 개수 조절 가능
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      labels:
        app: auth
        track: stable
    spec:
      containers:
        - name: auth
          image: "kelseyhightower/auth:1.0.0"
          ports:
            - name: http
              containerPort: 80 # HTTP 트래픽용 포드 80이 개방됨
            - name: health
              containerPort: 81 # 상태 확인용 포드 81이 개방됨
          resources:
            limits:
              cpu: 0.2
              memory: "10Mi" # 성능 제한
          livenessProbe:
            httpGet:
              path: /healthz
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /readiness
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            timeoutSeconds: 1
```

위 yml 적용한 deployments를 백포 객체로 만들고, 생성 여부 확인한다.

![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/de8b1ca7-06bd-4884-8d8f-4bab6766a5c4)

각각의 포트 연결 코드가 들어있는 frontend.yaml, hello.yaml도 배포를 만들고 노출한다.

확인

![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/8aa27321-3e85-4139-9d41-154c30476f2a)

```yaml
curl -ks https://kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`
```

⇒ Kubernetes 클러스터에서 "frontend"라는 Service를 조회하고, Service의 LoadBalancer의 IP 주소(아마 EXTERNAL-IP)를 JSON 경로(**`jsonpath`**)를 사용하여 추출, 그 걸로 curl명령어 실행

![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/4d286ebe-b87b-46ac-9340-acddd00739df)

replica 개수는 쉽게 scaling이 가능하다. 

![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/e34725e6-fe8e-425f-b28a-f6878174e872)

(확인 결과 yaml 파일의 코드가 바뀌는 것은 아님.)

## ****Task 3. Rolling update****

배포는 롤링 업데이트 매커니즘을 통해 이미지를 새 버전으로 업데이트 하도록 지원한다.

배포가 새 버전으로 업데이트 됨 → 세 버전의 replica set이 만들어짐 → 이전 replica set의 복제본 개수 감소하면서 새 버전의 replica set의 복제본 수 천천히 증가

![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/b8192bcf-9c37-4346-952a-e446ba94c389)

- 롤링 업데이트 트리거하기(**Trigger a rolling update)**
    - image의 버전을 변경하고 저장하면 `kubectl get replicaset`  명령어 수행시 실시간으로 업데이트 된 버전의 replica set 으로 업데이트 되는 것을 볼 수 있음
- 롤링 업데이트 일시중지/ 재개
    - 실행 중인 출시에 문제가 생긴다면?
        
        `kubectl rollout pause deployment/hello`
        
    - 재개하기
        
        `kubectl rollout resume deployment/hello`
        
- 완료된다면?
    
    `kubectl rollout status deployment/hello`시 출력결과
    
    `deployment "hello" successfully rolled out`
    
- 롤백 하기
    
    `kubectl rollout undo deployment/hello`
    
    ```yaml
    kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
    ```
    
    으로 실시간으로 replica 들이 하나씩 기존 버전으로 돌아가고 있는 것을 확인할 수 있다.
    

## ****Task 4. Canary deployments****

![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/4d6833a3-d61d-4e93-9e64-d8154b0d7fc4)

배포 시 두개의 클론을 생성한다. 

새로운 기능을 추가한 업데이트가 있을 시에는 하나만 반영한다.(카나리 버전 (↔ 안정 버전))

로드밸런서는 안정 버전에서 카나리 버전으로 트래픽의 일부를 점진적으로 전환한다. 일부 사용자 집합으로 새 버전을 테스트하는 효과가 있다. 문제가 생기면 필요한 조정을 수행할 수 있다.

문제가 없다고 느껴지면 모든 트래픽을 카나리 버전으로 전환하고, 롤아웃을 완료한다. 카나리 버전이 안정 버전으로 승격되는 것이다.

- 새 버전의 카나리 배포 만들기
    - `kubectl create -f deployments/hello-canary.yaml`

하지만, UI의 변경이 업데이트 내용인 경우에는 특정 사용자에게 혼동을 주지 않기 위해 사용자를 지정하여 한 배포에 고정해야되는 일이 있을 수 있다.  `sessionAffinity` 필드를 추가하여 Client IP를 설정할 수 있다.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: "hello"
spec:
  sessionAffinity: ClientIP
  selector:
    app: "hello"
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80
```

## ****Task 5. Blue-green deployments****

롤링 업데이트 :최소한의 오버헤드, 성능 영향, 다운타임 으로 애플리케이션을 배포할 수 있어서 좋다.

![image](https://github.com/yejin9858/gcp-studyjam-gke/assets/63632349/ed090bbc-1fb8-4d63-b893-47dc5619eea0)

하지만.  배포를 모두 완료한 후에 로드밸런서를 수정하여 새 버전을 가리키도록 하는 것이 유리한 경우가 있다. ( ex) 서비스 중단 없는 배포 등)리소스가 2배로 든다는 것이 단점이다.

- Bule-green 배포가 유리한 경우
    
    Blue-green 배포는 다음과 같은 여러 가지 시나리오에서 유용합니다:
    
    1. **서비스 중단 없는 배포**: Blue-green 배포는 기존 버전(blue) 옆에 새 버전(green)을 배포하고 트래픽을 green 환경으로 전환함으로써 애플리케이션을 중단 없이 업데이트할 수 있습니다. 배포 과정에서도 끊김 없는 서비스를 보장할 수 있습니다. 문제가 발생하면 빠르게 blue 환경으로 전환할 수 있습니다.
    2. **롤백과 롤포워드**: Blue-green 배포는 간편한 롤백 메커니즘을 제공합니다. 새 버전(green)이 예상치 못한 버그나 문제를 도입한 경우, 트래픽을 이전 버전(blue)으로 간단히 전환하여 영향을 최소화할 수 있습니다. 반대로, green 버전이 잘 동작하는 경우 쉽게 롤포워드하여 새로운 안정 버전으로 전환할 수 있습니다.
    3. **테스트와 검증**: Blue-green 배포를 통해 전체 사용자에게 노출하기 전에 새 버전을 철저히 테스트하고 검증할 수 있습니다. 트래픽의 일부를 green 환경으로 유도하여 기능, 성능, 호환성 등의 테스트를 수행할 수 있습니다. 이렇게 하면 기존 프로덕션 환경에 영향을 주지 않고 테스트할 수 있습니다.
    4. **용량 확장**: Blue-green 배포는 애플리케이션의 용량 확장에 활용될 수 있습니다. 트래픽이 증가하면 green 환경의 여러 인스턴스를 배포하여 부하를 처리하고, 트래픽을 점진적으로 전환합니다. 이를 통해 기존 프로덕션 환경을 방해하지 않고 효율적인 확장이 가능합니다.
    5. **인프라 업그레이드**: Blue-green 배포는 기반이 되는 구성 요소, 라이브러리, 데이터베이스 등의 인프라 업그레이드를 용이하게 합니다. 업데이트된 인프라를 green 환경에 배포하고 트래픽을 점진적으로 마이그레이션하여 원활한 전환을 보장할 수 있습니다.
    6. **A/B 테스트**: Blue-green 배포는 A/B 테스트 시나리오에 활용될 수 있습니다. (3. 격리된 테스트 환경)

green과 blue 중 사용할 배포를 `kubectl apply -f` 하면 된다. 롤백도 같은 방식이다.
