# DNS 정리

DNS(Domain Name System)는 사람이 이해하기 쉬운 도메인 이름을 실제 통신에 쓰이는 IP 주소로 바꿔주는 시스템.  
-> 단순히 "이름 → IP 변환"이 아니라, 계층 구조(NS 체인)랑 캐싱(TTL) 때문에 생각보다 까다로움.
  TTL 전파 지연으로 인 일부 지역은 이전 IP로 접속하거나, NXDOMAIN 캐시 때문에 신규 도메인이 바로 안 뜨는 경우가 실제로 발생

---

## 1. DNS 질의 흐름

도메인 질의는 Root → TLD → Authoritative 단계를 거쳐 최종 IP에 도달한다.

| 단계                  | 반환되는 응답                            | 예시                                      |
|-----------------------|------------------------------------------|-------------------------------------------|
| Root 네임서버         | TLD 네임서버 NS 레코드                   | `.com. IN NS a.gtld-servers.net.`         |
| TLD 네임서버 (.com)   | Authoritative 네임서버(NS + 글루 A)       | `example.com. IN NS ns1.exampledns.com.`  |
| Authoritative 네임서버| 최종 레코드(A, MX, CNAME 등)              | `www.example.com. IN A 93.184.216.34`     |

→ “위치를 알려준다”는 말은 결국 **NS 레코드를 반환한다는 얘기**.  
Root는 TLD NS를, TLD는 Authoritative NS를 알려주고, 진짜 IP는 Authoritative에서만 받는다.  

---

## 2. Recursive vs Authoritative DNS

| 구분 | Recursive DNS (재귀 DNS, Resolver) | Authoritative DNS (권한 DNS, AWS 번역: 신뢰할 수 있는 DNS) |
|------|------------------------------------|----------------------------------------------------------|
| 역할 | 클라이언트 대신 Root → TLD → Authoritative까지 질의 | 특정 도메인의 최종 레코드를 권한 있게 보관하고 응답 |
| 예시 | Google DNS(8.8.8.8), Cloudflare(1.1.1.1) | AWS Route53, Azure DNS, Cloudflare Authoritative |
| 핵심 | **중개자 (캐시)** | **정답 보관자 (원천 데이터)** |

---

## 3. 주요 레코드 타입 

- **A / AAAA**: IPv4 / IPv6 주소  
- **CNAME**: 별칭. apex 도메인에는 둘 수 없음 → 클라우드 사업자들은 ALIAS/ANAME 같은 기능 제공
    > Note: 서버 IP 바뀌면 TTL 때문에 전파 지연이 생김 (빈번한 이슈)
- **MX**: 메일 서버 지정. priority 값 잘못 잡으면 메일 라우팅 꼬임
    > Note: 클라우드 로드밸런서 붙일 때 실수로 apex에 CNAME 걸면 안 됨. (ALIAS/ANAME 필요)  
- **TXT**: SPF, DKIM, DMARC 같은 메일 인증에 필수
    > Note: DKIM 누락되면 메일 스팸 처리됨
- **PTR**: 역방향(IP → 도메인). 메일 서버 신뢰도와 직결
    > Note: 메일 서버 PTR 누락되면 외부 서버가 신뢰 안 해서 메일 튕김

→ 실제로 어디서 문제 생기는지 같이 알아둘 것


---

## 4. TTL과 캐싱

- **TTL**: 레코드가 캐시에 머무는 시간  
- Positive Cache: 정상 응답(IP) 저장  
- Negative Cache: NXDOMAIN도 TTL 동안 캐시됨  

장점: 속도 빠르고 서버 부하 줄어듦  
단점: 변경 전파가 늦음 (DNS 전파 지연 문제)  

→ Azure DNS에서 TTL 3600으로 설정한 레코드 바꿔봤는데, 실제로는 30분 넘게 기존 IP가 캐시돼 있었음. TTL 전파 지연이 진짜 이렇게 체감됨.  

---

## 5. 테스트 & 실습 기록

### 5.1 기본 조회
```bash
dig www.google.com A +noall +answer
```

---

## 6. DNS에서 자주 발생하는 장애 유형

### 6.1 NXDOMAIN 캐싱 (Negative Cache)
- 신규 도메인을 등록하기 전 질의하면 **NXDOMAIN**(존재하지 않는 도메인) 응답을 받는다.  
- 문제는 이 응답도 TTL 동안 캐시된다는 점.  
- 그래서 등록 직후에도 일부 리졸버는 계속 NXDOMAIN을 반환 → "도메인 안 뜬다" 현상 발생.

🔑 실제로 신규 서비스 런칭할 때, 사전에 질의가 있었다면 몇 분~몇 시간 동안 접속이 안 될 수 있다.

---

### 6.2 TTL 전파 지연 (Stale Record)
- 기존 A 레코드를 새로운 IP로 교체해도, TTL이 남아 있는 리졸버는 예전 값을 계속 반환한다.  
- 예: `www.example.com` → `203.0.113.10` 에서 `203.0.113.20`으로 교체했는데, 일부 지역 사용자는 여전히 203.0.113.10으로 접속.  
- TTL 만료 후에야 전파가 완료됨.

🔑 배포 시점에는 TTL을 짧게(예: 60초) 줄여두고, 안정화 후 다시 늘리는 전략을 쓴다.

---

### 6.3 권한 네임서버(NS) 불일치
- TLD에 등록된 NS 정보와 실제 네임서버 설정이 어긋나는 경우.  
- 일부 리졸버에서는 정상, 일부에서는 NXDOMAIN → "지역별로 접속이 되기도 하고 안 되기도 함" 현상 발생.

---

### 6.4 Reverse DNS(PTR) 누락
- 메일 서버 IP ↔ 도메인 매핑이 없으면 신뢰도 문제 발생.  
- 외부 수신 서버에서 스팸 판정 → "메일은 보냈는데 수신자가 못 받는다" 문제.

---

📝 요약:  
DNS 장애는 단순히 “DNS 서버가 죽었다” 문제가 아니라,  
**캐싱(TTL, Negative Cache), 전파 지연, NS 설정 오류** 때문에 더 자주 발생한다.

