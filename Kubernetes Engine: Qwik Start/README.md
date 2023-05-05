# Kubernetes Engine: Qwik Start
## Set a default compute zone
- compute zone이란, 클러스터와 리소스가 존재하는 대략적인 위치이다. ex) us-central1-a is a zone in the us-central1 region
- 기본 컴퓨팅 리전(영역) 설정 : `gcloud config set compute/region(zone) [region(zone)]`

## Create a GKE cluster
- 클러스터 -> 1개 이상의 클러스터 마스터 머신 + 여러 노드(작업자 머신, Kubernetes 프로세스를 실행하는 Compute Engine 가상 머신(VM) 인스턴스)
- `gcloud container clusters create --machine-type=e2-medium --zone=us-east1-b lab-cluster`는 'lab-cluster'라는 클러스터를 생성한다.

## Get authentication credentials for the cluster
- 클러스터와 상호작용 하려면 사용자 인증 정보가 필요함 -> `gcloud container clusters get-credentials lab-cluster `


## Deploy an application to the cluster
- GKE는 kubernetes 객체를 사용하여 클러스터의 리소스를 만들고 관리한다.
- 쿠버네티스의 배포 객체 : 웹 서버와 같은 스테이트리스(Stateless) 애플리케이션을 배포힐 때 사용
              서비스 객체: 애플리케이션에 액세스하기 위한 규칙과 부하 분산 방식을 정의, 실행중인 애플리케이션을 네트워크 서비스로 노출하는 추상화 방법
- 새 배포 생성 : `kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0` - 샘플 이미지(hello-app)을 들고와서 hello-server로 배포
- 서비스 생성 : `kubectl expose deployment hello-server --type=LoadBalancer --port 8080` 
   8080포트에 컨테이너를 노출시킨다. 
   type="LoadBalancer"는 컨테이너의 Compute Engine 부하 분산기를 생성한다. 
- `kubectl get service1` 현재 실행중인 pod(a set of running containers in your cluster)들을 출력한다.
- `http://[EXTERNAL-IP]:8080`(위 명령어에서 EXTERNAL-IP가 pending 상태면 대기) 를 웹 브라우저에 입력하면 애플리케이션을 볼 수 있다.
  ![image](https://user-images.githubusercontent.com/63632349/236534783-574b880f-6374-4cfa-a37c-f5847474c6b0.png)
  
 ## Deleting the cluster
 - `gcloud container clusters delete lab-cluster`는 이름이 `lab-cluster`인 클러스터를 제거한다.

   

 
