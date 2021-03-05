
# skstore(sk 장터)
   　  

# 서비스 시나리오

`기능적 요구사항`
1. 사용자가 주문서비스에서 제품을 주문한다.
1. 사용자가 주문금액을 결제한다.
1. 주문금액 결제가 완료되면 주문내역이 상품정보에 전달된다.
1. 상품정보에 주문정보가 전달되면 주문서비스에 배송상태를 배송시작 상태로 변경한다.
1. 배송이 시작되면 주문서비스에서 현재 배송상태를 조회할 수 있다.
1. 사용자가 주문을 취소할 수 있다.
1. 사용자가 주문금액에 대한 결제상태를 Deposit 서비스에서 조회 할 수 있다.
1. 사용자가 모든 진행내역을 볼 수 있어야 한다.
  
  
`비기능적 요구사항`
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다.(Sync 적용)
    1. 주문을 취소하면 Deposit을 환불하고 product에 주문취소 내역을 전달한다.(Async 적용)
1. 장애격리
    1. Deposit 시스템이 과중되면 예약을 받지 않고 잠시후에 하도록 유도한다(Circuit breaker, fallback)
    1. Item 서비스가 중단되더라도 예약은 받을 수 있다.(Async, Event Driven)
1. 성능
    1. 사용자가 주문상황을 조회할 수 있도록 별도의 View 로 구성한다.(CQRS)  


　    
  
  
# 체크포인트

1. Saga
2. CQRS
3. Correlation
4. Req/Resp
5. Gateway
6. Deploy/ Pipeline
7. Circuit Breaker
8. Autoscale (HPA)
9. Zero-downtime deploy (Readiness Probe)
10. Config Map/ Persistence Volume
11. Polyglot
12. Self-healing (Liveness Probe)  

　  
  
# 분석/설계

### Event Storming 결과
![model](https://user-images.githubusercontent.com/78134032/110052437-f3950f80-7d9a-11eb-9050-f4f227360621.jpg)
　  
　     
### 기능적 요구사항 검증(1)

![1](https://user-images.githubusercontent.com/47556407/108023276-8b89be00-7065-11eb-95bb-b13d35dfca3f.png)

    - 사용자가 주문서비스에서 제품을 주문한다.(OK)
    - 사용자가 주문금액을 결제한다.(OK)
    - 주문금액 결제가 완료되면 주문내역이 상품정보에 전달된다.(OK)
    - 상품정보에 주문정보가 전달되면 주문서비스에 주문상태를 배송시작 상태로 변경한다.(OK)
    - 주문이 완료되면 주문서비스에서 현재 주문상태를 조회할 수 있다.(OK)
    
　  
　  
### 기능적 요구사항 검증(2)   
![2](https://user-images.githubusercontent.com/47556407/108023413-cee42c80-7065-11eb-9ac2-f0f5e5c6e910.png)

    - 사용자가 주문을 취소할 수 있다.(OK)
    - 주문을 취소하면 주문금액을 환불한다.(OK)
    - 사용자가 주문금액에 대한 결제상태를 Deposit 서비스에서 조회 할 수 있다.(OK)  
    
    
　  
　  
### 기능적 요구사항 검증(3)   
![3](https://user-images.githubusercontent.com/47556407/108023936-f5569780-7066-11eb-9f9f-83063e69b7ed.png)

    - 사용자가 모든 진행내역을 볼 수 있어야 한다.(OK)
    
　  
　  
   
### 비기능 요구사항 검증

    - Deposit이 결재되지 않으면 주문이 안되도록 해아 한다.(Req/Res)
    - Item 서비스가 중단되더라도 예약은 받을 수 있어야 한다.(Pub/Sub)
    - Deposit 시스템이 과중되면 주문을 받지 않고 잠시후에 하도록 유도한다.(Circuit breaker)
    - 주문을 취소하면 Deposit을 환불하고 Item 에 주문취소 내역을 업데이트해야 한다.(SAGA)
    - 사용자가 주문상황을 조회할 수 있도록 별도의 view로 구성한다.(CQRS)  
    
　  
　  
       

# 구현

서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8084 이다)

```
cd order
mvn spring-boot:run

cd deposit
mvn spring-boot:run  

cd myinfo
mvn spring-boot:run 

cd item
mvn spring-boot:run 
```
    
　  
　  
   
### DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 order 마이크로 서비스)

![20210215_120254](https://user-images.githubusercontent.com/47556407/108024373-c8ef4b00-7067-11eb-90cd-073e201b2adb.png)
    
　  
　  
   
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

![20210215_120624](https://user-images.githubusercontent.com/47556407/108024593-38fdd100-7068-11eb-8452-bfa6a2471708.png)
    
　  
　  
   
- 적용 후 REST API 의 테스트

```
# order 서비스의 예약처리
http http://52.231.96.78:8080/orders productId=10 qty=5
```
![image](https://user-images.githubusercontent.com/47556407/108030441-972fb180-7072-11eb-85a8-b76ded12e852.png)


```
# order 서비스의 주문상태 확인
http localhost:8081/orders/1
```
![image](https://user-images.githubusercontent.com/47556407/108030523-bcbcbb00-7072-11eb-8978-6fa1fd65bbd1.png)

```
# product 서비스의 주문현황 확인
http localhost:8084/products/1

```
![image](https://user-images.githubusercontent.com/47556407/108032100-48374b80-7075-11eb-933a-49ab5edf49a5.png)
　  
　  
   

# Polyglot

Order, Deposit, MyInfo는 H2로 구현하고 Item 서비스의 경우 Hsql로 구현하여 MSA간의 서로 다른 종류의 Database에도 문제없이 작동하여 다형성을 만족하는지 확인하였다.

- order, deposit, myinfo 의 pom.xml 파일 설정

![image](https://user-images.githubusercontent.com/47556407/108033377-44a4c400-7077-11eb-80c6-7cae6864f161.png)
    
　  
 
- item 의 pom.xml 파일 설정

![image](https://user-images.githubusercontent.com/47556407/108033035-b8929c80-7076-11eb-892f-fcbfcc448f98.png)
    
　  
    
　  
　  
   

# Req/Resp
```
1. 분석단계에서의 조건 중 하나로 주문(order)->주문금액 결제(deposit) 간의 호출은 동기식 일관성을 유지하는
트랜잭션으로 처리하기로 하였다. 

2. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 
호출하도록 한다. 
```
    
　  
    
    
- 주문금액 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현  (Depositservice.java)

![image](https://user-images.githubusercontent.com/47556407/108034489-f1cc0c00-7078-11eb-8961-cb650db2b9fa.png)

    
　  
    

- 주문을 받은 직후(@PostPersist) 주문금액 결제를 요청하도록 처리

![image](https://user-images.githubusercontent.com/47556407/108034722-453e5a00-7079-11eb-93f0-ed180ae81dba.png)
    
　  
　  
### 동기식 결제 장애시

```
# 결제 (deposit) 서비스를 잠시 내려놓음
# 예약 처리
kubectl delete deploy deposit -n skuser03
```
![image](https://user-images.githubusercontent.com/47556407/108035947-f09bde80-707a-11eb-82d3-7972e611fafc.png)

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 예치금 결제 시스템이 장애가 나면 예약도 못받는다는 것을 확인

![image](https://user-images.githubusercontent.com/47556407/108036023-090bf900-707b-11eb-9c11-16fa02888f4b.png)
    
　  
　  
```
# 결재(deposit)서비스 재기동
kubectl create deploy deposit --image=skuser03.azurecr.io/deposit:latest -n skuser03
```
![image](https://user-images.githubusercontent.com/47556407/108036520-abc47780-707b-11eb-8d30-1b7667a05a4d.png)

![image](https://user-images.githubusercontent.com/47556407/108036597-c860af80-707b-11eb-8b36-d34ed75c379f.png)

    
　  
　  
    
　  
　  
   
# Gateway
- gateway > application.yml

![image](https://user-images.githubusercontent.com/47556407/108038063-c0097400-707d-11eb-9944-8b734b2f7dcf.png)
    
　  
　  

- Gateway의 External-IP 확인

![image](https://user-images.githubusercontent.com/47556407/108039061-f4c9fb00-707e-11eb-8409-c7bd37228b03.png)
    
　  
　  
- External-IP 로 order서비스에 접근

![image](https://user-images.githubusercontent.com/47556407/108038613-6786a680-707e-11eb-8dd1-92c3e04807bc.png)

    
　  
　      
　  
　  

# Deploy

- Deploy API 호출

```
# 소스를 가져와 각각의 MSA 별로 빌드 진행

# 도커라이징 : Azure Registry에 Image Push 
az acr build --registry skuser07 --image skuser07.azurecr.io/order:v7 .  
az acr build --registry skuser07 --image skuser07.azurecr.io/deposit:v7 . 
az acr build --registry skuser07 --image skuser07.azurecr.io/item:v7 .   
az acr build --registry skuser07 --image skuser07.azurecr.io/myinfo:v7 .   
az acr build --registry skuser07 --image skuser07.azurecr.io/gateway:v7 . 

# 컨테이터라이징 : Deploy, Service 생성
kubectl create deploy order --image=skuser07.azurecr.io/order:v7
kubectl expose deploy order --type="ClusterIP" --port=8080
kubectl create deploy deposit --image=skuser07.azurecr.io/deposit:v7
kubectl expose deploy deposit --type="ClusterIP" --port=8080
kubectl create deploy item --image=skuser07.azurecr.io/item:v7
kubectl expose deploy item --type="ClusterIP" --port=8080
kubectl create deploy myinfo --image=skuser07.azurecr.io/myinfo:v7 
kubectl expose deploy myinfo --type="ClusterIP" --port=8080
kubectl create deploy gateway --image=skuser07.azurecr.io/gateway:v7 
kubectl expose deploy gateway --type=LoadBalancer --port=8080

#kubectl get all
```
    
　  
　  
- Deploy 확인

![image](https://user-images.githubusercontent.com/47556407/108038482-3908cb80-707e-11eb-83fe-20dcc7094fc6.png)
    
　  
　  
      
      
    
　  
　  
    

# Circuit Breaker
```
1. 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함.  
2. 시나리오는 주문(order) --> 주문금액 결제(deposit) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 주문금액 결제 요청이 과도할 경우 CB 를 통하여 장애격리.  
3. Hystrix 를 설정: 요청처리 쓰레드에서 처리시간이 300 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```

    
　  
　  


- application.yml 설정

![image](https://user-images.githubusercontent.com/47556407/108139128-c130b480-7102-11eb-919b-a64042c30132.png)

    
　  

- 피호출 서비스(주문금액 결제:deposit) 의 임의 부하 처리  order.java(entity)

![image](https://user-images.githubusercontent.com/47556407/108139181-da396580-7102-11eb-8588-fae0598381f7.png)

    
　  
　  

`$ siege -c255 -t60S -r10 -v --content-type "application/json" 'http://52.231.70.170:8080/orders POST {"productId": "10", "qty":"5"}'`

- 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인 (동시사용자 255명, 60초 진행)

![image](https://user-images.githubusercontent.com/47556407/108157604-7cb71000-7126-11eb-8b16-da6f29c89cb8.png)


```
* 요청이 과도하여 CB를 동작함 요청을 차단
* 요청을 어느정도 돌려보내고나니, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청을 다시 받기 시작
* 다시 요청이 쌓이기 시작하여 건당 처리시간이 610 밀리를 살짝 넘기기 시작 => 회로 열기 => 요청 실패처리
```
    
　  
　  
![image](https://user-images.githubusercontent.com/47556407/108157652-9c4e3880-7126-11eb-97a2-5f5ab36c3327.png)


`운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌`
    
　  
    
　  
　  
   
# Auto Scale(HPA)
```
1. 앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 
이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.  
2. 주문금액 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 
넘어서면 replica 를 10개까지 늘려준다.
```

- 테스트를 위한 리소스 할당(order > deployment.yml)

![image](https://user-images.githubusercontent.com/47556407/108158884-3ca55c80-7129-11eb-9b85-054afaac7e3d.png)
    
　  
　  

### autoscale out 설정 

- kubectl autoscale deploy order --min=1 --max=10 --cpu-percent=15 -n skuser03

![image](https://user-images.githubusercontent.com/47556407/108158989-6eb6be80-7129-11eb-88c8-b79b2bb9891f.png)
    
　  
　  
- CB 에서 했던 방식대로 워크로드를 1분 동안 걸어준다.

`$ siege -c255 -t120S -v --content-type "application/json" 'http://52.231.70.170:8080/orders POST {"productId": "10", "Qty":"5"}' `

    
　  
　  
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:

`watch kubectl get all`

    
　  
　  
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

![image](https://user-images.githubusercontent.com/47556407/108164875-f81fbe00-7134-11eb-87b5-7ca950d77ab4.png)
    
　  
　  
    
　  
　  
   
# Zero-Downtown Deploy

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscale 나 CB 설정을 제거함

- siege 로 배포작업 직전에 워크로드를 모니터링 함.

`siege -c100 -t80S -v --content-type "application/json" 'http://20.194.19.238:8080/orders POST {"productId": "10", "Qty":"5"}'`
    
　  
　  

- 새버전으로의 배포 시작
```
kubectl set image deploy order order=skuser07.azurecr.io/order:v7
```
    
　  
　  
### readiness 옵션이 없는 경우 배포 중 서비스 요청처리 실패

![image](https://user-images.githubusercontent.com/47556407/108172175-d7109a80-713f-11eb-9466-cceaec00d656.png)
    
　  
　  
   
### readiness 옵션 추가

- deployment.yaml 의 readiness probe 의 설정

![image](https://user-images.githubusercontent.com/47556407/108172274-f90a1d00-713f-11eb-903d-c29c1e109f69.png)
    
　  
　  
```
# readiness 적용 이미지 배포
kubectl apply -f kubernetes/deployment.yaml
# 이미지 변경 배포 한 후 Availability 확인:
```
![image](https://user-images.githubusercontent.com/47556407/108172582-5900c380-7140-11eb-90ff-1061ad2f1e36.png)
    
　  
　  
- 배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

![image](https://user-images.githubusercontent.com/47556407/108172642-68800c80-7140-11eb-8830-0fbc2210c473.png)
    
　  
　      
    
　  
　  
   　  
　  
# Self-healing (Liveness Probe)

- deployment.yml 에 Liveness Probe 옵션 추가

![image](https://user-images.githubusercontent.com/47556407/108172764-8f3e4300-7140-11eb-8f83-e87f4a2d6b3f.png)
    
　  
　  
- order pod에 liveness가 적용된 부분 확인

![image](https://user-images.githubusercontent.com/47556407/108173813-f5779580-7141-11eb-90d7-906091ed9027.png)

    
　  
　  
- order 서비스의 liveness가 발동되어 7번 retry 시도 한 부분 확인

![image](https://user-images.githubusercontent.com/47556407/108174264-8f3f4280-7142-11eb-9941-170e07025c05.png)



