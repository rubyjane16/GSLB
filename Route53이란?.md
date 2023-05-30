### Route53
Route53이란 AWS에서 제공하는 DNS(domain name server)이다. 

​

우선 DNS란, IP주소가 아닌 도메인 이름으로 IP를 찾아가는 방식이다.

DNS에서 보관하고 관리하는 도메인에는 다양한 내용을 매핑할 수 있는 레코드가 존재하는데 

그 중 기본 레코드인 A(IPv4)레코드는 도메인 주소를 IP주소로 변환하는 레코드 이다.  

하나의 도메인 주소는 하나의 IP주소와 매핑되는데 이때 여러 개의 도메인 주소가 하나의 동일한 IP주소와 연결될 수도 있고 (A), 

여러 개의 IP 주소가 하나의 동일한 도메인에 연결될 수도 있다. (B)

<p align="center">
  <img src="https://github.com/rubyjane16/GSLB/assets/89911621/8a1ba6f0-33a0-4875-868e-efade3595f7b">
</p>

B의 경우, 하나의 도메인으로 여러개의 IP주소를 매핑하고 있다.

 이는 일종의 로브밸런싱(서버가 처리해야 할 업무를 여러 대의 서버로 분산시키는 것이다) 기술이다. 

그러나 일반 DNS는 서버의 상태를 확인하지 못하기 때문에 서버에 문제가 발생했을 때도

계속해서 요청이 들어가게 된다. 이를 해결하기 위해 GSLB가 구현되었다. 

<br>
### GSLB란?
<br>

DNS 서버와 로드밸런서의 역할을 동시에 수행한다고 볼 수 있다. 로드밸런서는 DNS 서버에서 제공하지 않는

헬스체크(Health check) 기능을 제공한다. 헬스체크란 연결된 다수의 서버의 상태를 주기적으로 확인하여 

서버의 상태가 통신 불가능인 경우 서비스에서 제외해 전세계(Global) 어느 곳에서도 서비스를 원활하게 제공하는 기능이다.

(GSLB 의 본질은 여러 개의 데이터센터 중 어느 곳에 트래픽을 전달할지 결정하는 것이며, 일반 로드밸런서의 본질은 하나의 데이터센터 내에 위치한 여러 대의 서버에 트래픽을 분산시키는 것이다. )

​

GSLB는 2가지의 구성방식이 있는데 

도메인의 모든 레코드를 GSLB에서 관리

도메인 내의 특정 레코드만 GSLB에서 사용

​

1의 경우 모든 레코드에 대한 질의가 GSLB에 들어오기 때문에 오히려 GSLB의 부하가 늘어날 수 있는 문제가 발생한다.

​

2의 경우는 DNS와 GSLB를 따로 운영하는 경우와 같이 운영하는 경우 2가지가 있다.

​

2-1.CNAME 레코드 사용: 별칭 (Alias)

Route53의 특이 레코더 별칭(Alias)으로 도메인과 도메인 주소를 매핑한다.

도메인 자체에 별칭을 줄 수 있어서 naver.com 도메인 자체에 ALIAS Target을 www로 줄 수 있다. 

naver.com을 치면 www.naver.com과 같은 IP를 응답하게 한다.

따라서 www.IP를 바꾸면 도메인 자체(naver.com) IP도 바뀌게 된다. (alias의 역할)

​

ex) www.naver.com=naver.com

​

2-2. NS 레코드 사용: 위임 (Delegation)

NS 레코드: 도메인에 대한 권한이 있는 네임 서버 정보를 설정하는 레코드 

Route53을 이용하여 도메인을 설정하면 NS레코드가 자동 생성된다. 

지정된 NS정보를 해당 네임서버 정보로 등록한다.

​

(NS 레코드를 사용하는 위임의 장점은 계층화된 도메인에 대해 일괄적인 처리가 가능하다는 것이다. 

예를 들어 web.google.com 을 특정 GSLB 에 위임했다면 a.web.google.com 도, b.web.google.com 도 

전부 같은 GSLB 에서 처리된다. )

<p align="center">
  <img src="https://github.com/rubyjane16/GSLB/assets/89911621/e7cd8c16-1c10-407a-9b82-d05d3966a735">
</p>

2-1의 경우

1)클라이언트가 LDNS(Local DNS)에게 웹 IP주소를 질의하면 

2-1)naver.com의 DNS 서버 주소를 받고, DNS 서버에 질의한다.

3-1)DNS에서 naver.com의 매핑 정보를 확인하여 CNAME 레코드인 www.naver.com 응답한다.

2-2)LDNS은  www.naver.com을 다시 질의

3-2)DNS 서버에서 www.naver.com은 GSLB에서 관리하고 있다고 응답

4)해당 GSLB를 찾아간다.

5)GSLB는 모든 서버에 헬스체크를 하고 

6)알고리즘 결과에 따라 IP주소를 반환한다.

7)LDNS는 받은 IP주소를 캐싱한 뒤, 클라이언트에게 전달한다.

클라이언트는 최종 응답 받은 IP주소로 패킷을 보낸다.

​

2-2의 경우 

1)클라이언트가 LDNS(Local DNS)에게 웹 IP주소를 질의하면 

2)LDNS가 클라이언트에게 받은 DNS쿼리를 DNS서버에 질의한다.

3)DNS에 해당 도메인의 정보가 존재하는 경우, GSLB에게 권한을 위임했으므로 GSLB의 주소를 넘긴다.

4)LDNS가 GSLB에게 같은 쿼리를 날리면 

5)GSLB는 모든 서버에 헬스체크를 하고 

6)알고리즘 결과에 따라 IP주소를 반환한다.

7)LDNS는 받은 IP주소를 캐싱한 뒤, 클라이언트에게 전달한다.

클라이언트는 최종 응답 받은 IP주소로 패킷을 보낸다.

​


참고: 

[Network] GSLB (Global Server Load Balancing) (tistory.com)

11.Route 53이란? (brunch.co.kr)


