# istio-system
## istio concept
- 다른 pod 내의 컨테이너 끼리의 통신을 용이하게 하기 위한 시스템
- proxy를 중앙 관리하는 proxy server에서 로그를 수집하여 트래픽을 관리하고 이것을 통해 시각화가 가능하다
## traffic control
### traffic suspend
- aws로 향하는 외부 api-service의 traffic을 중지한다
- yaml 파일로 생성
### Delete All Traffic Routing
- 가중치 기반 라우팅, traffic suspend 삭제

## VirtualService
- 서비스 메시에서 사용자 지정 라우팅 규칙 설정 가능하게 해준다
- 사실상 라우팅 규칙 yaml이다

### istio-pilot
- VirtualService를 관리하는 pod
- 사용자 지적 라우팅 규칙의 결과로서 변경되어야 할 모든 프록시에 배포

## Sesstion affinity(Sticky Session)
- 유저의 세션을 저장하여 계속해서 같은 버전으로 연결되게 (동일한 유저가 새로고침해도 계속해서 동일한 버전에 접속되도록)
- header or ip 기반
- Consistent Hashing

### 세션 어피니티+가중치 서브셋
- envoy가 지원하지 않는다
- [github issue](https://github.com/envoyproxy/envoy/issues/8167)

## webapp weight routing
- 프록시 실행은 컨테이너가 요청을 보낸 이후.
- 따라서 webapp pods 앞에 proxy가 있어야 canary가 가능해진다
- istio gateway pod가 그 역할을 한다
### istio gateway (ingress gateway)
- 외부에서 들어오는 트래픽에 대한 가중치 기반 라우팅 구현 (sticky session+weight based routing은 여전히 불가능)
- 엣지 프록시 설정하게 해준다
- Gateway pod에서 selector로 불러올 수 있다 (kubectl get po -n istio-system --show-labels 로 확인 가능)
- VirtualService에 gateways에 name 추가하여 설정


## prefix
- [HTTPMatchRequest](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPMatchRequest)

## Header Based Routing 
- Kiali: Create Matching Routing 에서 VirtualService yaml 생성 가능
- [ModHeader](https://chromewebstore.google.com/detail/modheader-modify-http-hea/idgpnmonknjnojddfkpgkljpfnnfcklj)
- header를 바로 수정한다고 변경사항이 적용되진 않는다(내부 microservice까지 변경사하잉 전파되지 않으므로)
- 개발 단계에서 header 전파를 수동으로 구현해야한다
- "my-header"->"x-my-header" 로 변경하면 새로고침 없이 사진이 보인다

### 다크 배포 (Dark Release)
- 스테이징 클러스터를 마련할 필요 없이 실사용 클러스터에서 배포 (비용 절감)
- 헤더 기반 라우팅으로 구현
- 개발 레벨에서 header 수동 전파를 해야하는 단점이 있다 (tradeoff)

## 연계 고장 (Cascading Failure)
- 하나의 서비스가 늦게 응답하거나 종료되어 다른 연계된 서비스도 고장
### Circuit Breaking
- 회로 차단기
- fail fast 매커니즘
### Netflix/Hystrix
- 2018 개발 중단
#### 전통적 서킷 브레이커 (classic circuit breaker)
- 모든 마이크로서비스가 서킷 브레이커를 가지도록 작업해야하는 장황한 작업
- 특정 언어에 종속

### envoy circuit breaker
- proxy circuit breaker 사례 중 하나
- 서킷 브레이커는 pod level에서 작동

### 극단치 탐지 (Outlier Detection)
- [OutlierDetection](https://istio.io/latest/docs/reference/config/networking/destination-rule/#OutlierDetection)
#### Destination Rule
- 부하 분산
- 서킷 브레이킹

### 서킷 브레이커 테스트
#### 로드 테스팅
- [fortio](https://github.com/fortio/fortio)
#### 극단치 탐지 (outlierDetection, =서킷 브레이커)
- consecutive5xxErrors : 500번 에러 발생 횟수
- interval : 탐지 주기
- baseEjectionTime: 서킷 브레이킹 활성화 시간(=요청 안보내는 시간). 지난 뒤에 다시 요청 보냄
- maxEjectionPercent : 서킷 브레이킹 비율. 트래픽을 얼마나 통과시키고 차단시킬지 % (~100)