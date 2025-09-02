# DNS 정리

DNS(Domain Name System)는 사람이 이해하기 쉬운 도메인 이름을 실제 통신에 쓰이는 IP 주소로 바꿔주는 시스템.  
단순히 "이름 → IP 변환"이 아니라, 계층 구조(NS 체인)랑 캐싱(TTL) 때문에 생각보다 까다롭다.  
운영하다 보면 여기서 장애가 많이 생김.

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

→ AWS 문서에서는 Authoritative를 "신뢰할 수 있는 DNS"라고 번역해놨지만, 의미는 권한 있는 DNS가 맞음.  

---

## 3. 주요 레코드 타입 (중요한 것만)

- **A / AAAA**: IPv4 / IPv6 주소  
- **CNAME**: 별칭. apex 도메인에는 둘 수 없음 → 클라우드 사업자들은 ALIAS/ANAME 같은 기능 제공  
- **MX**: 메일 서버 지정. priority 값 잘못 잡으면 메일 라우팅 꼬임  
- **TXT**: SPF, DKIM, DMARC 같은 메일 인증에 필수. 예전에 고객사에서 DKIM 누락돼서 메일 스팸 처리된 적 있었음  
- **PTR**: 역방향(IP → 도메인). 메일 서버 신뢰도랑 직결  

→ 단순히 암기하는 것보다, **왜 중요한지**랑 실제로 어디서 문제 생기는지 같이 적는 게 도움 됨.  

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
