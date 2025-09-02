# DNS 

## 1. DNS 질의 흐름
DNS는 도메인 이름(사람이 이해하는 주소 형태)을 실제 통신에 쓰이는 IP 주소로 변환하는 시스템
> 계층별로 NS 레코드 체인을 따라 내려가는 구조

1) **Root 네임서버**         : 전 세계에 Anycast로 분산된 13개의 논리 그룹. TLD 네임서버의 정보 제공(=NS 레코드 목록 반환)
2) **TLD  네임 서버**        : `.com`, `.net`, `.kr` 등 최상위 도메인 관리. 해당 도메인의 권한 네임서버의 정보 제공
3) **Authoritative 네임서버**: 특정 도메인의 최종 레코드(A, MX, CNAME 등) 보관 및 권한 있는 응 제공

📌 Root → TLD → Authoritative 과정을 거쳐 최종 응답을 반환함

  ## DNS 질의 단계별 응답

  | 단계                  | 질의 대상         | 반환되는 응답                                      | 실제 의미                                   |
  |-----------------------|------------------|------------------------------------------------------|--------------------------------------------|
  | Root 네임서버         | www.example.com  | `.com` 도메인을 관리하는 TLD 네임서버들의 NS 레코드   | ".com은 이 서버들이 관리한다 → 다음 단계로" |
  | TLD 네임서버 (.com)   | www.example.com  | `example.com`의 권한 네임서버(NS) + 글루 레코드(A)    | "example.com은 이 권한 서버들이 관리한다"   |
  | Authoritative 네임서버| www.example.com  | 최종 레코드 (A, MX, CNAME 등)                         | "www.example.com → 203.0.113.10"           |


---

📌 요약:
- Root/TLD도 결국 **NS 레코드**를 반환하는 것 → 우리가 표현할 때 “위치를 알려준다”라고 말하는 것의 본질  
- Authoritative에서야 비로소 **실제 리소스 레코드(A, MX, CNAME 등)** 를 반환 → 최종 답변


---

## 2. Recursive vs Authoritative DNS
| 구분 | Recursive DNS (재귀 DNS / Resolver) | Authoritative DNS (권한 DNS, AWS 번역: 신뢰할 수 있는 DNS) |
|------|-------------------------------------|-------------------------------------------------------|
| 역할 | 사용자를 대신해 Root → TLD → 권한 서버까지 재귀 질의 | 특정 도메인에 대한 **최종 레코드** 보관 및 응답 |
| 소유 데이터 | 자체 도메인 데이터 없음, 캐시만 보관 | 도메인 원천 데이터 보관 |
| 예시 | Google DNS (8.8.8.8), Cloudflare DNS (1.1.1.1), ISP DNS | AWS Route53, Azure DNS, Cloudflare Authoritative |
| 응답 방식 | 캐시에 있으면 즉시 응답, 없으면 다른 서버 질의 | 항상 최종 정답을 직접 제공 |

---

## 3. DNS 캐싱과 TTL
- **TTL(Time To Live)**: 레코드가 캐시에 유지되는 시간  
- **Positive Cache**: 정상 응답(IP) 저장  
- **Negative Cache**: NXDOMAIN도 TTL 동안 캐시됨  
- 장점: 응답 속도 향상, 부하 감소  
- 단점: 변경사항 전파가 늦어짐 (DNS 전파 지연 문제)

---

## 4. 주요 레코드 타입
- **A / AAAA**: IPv4 / IPv6 주소
- **CNAME**: 별칭 레코드, 다른 도메인 참조
- **MX**: 메일 서버 주소
- **TXT**: 도메인 검증, SPF/DKIM/DMARC 설정
- **PTR**: 역방향(IP → 도메인), 메일 서버 신뢰성 검증에 중요

---

## 5. 트러블슈팅 도구
- **dig**: 세부 질의 확인 가능 (`dig +trace`, `dig @8.8.8.8 example.com`)
- **nslookup**: 간단 확인 용도
- **host**: 도메인 ↔ IP 매핑 간단 조회

📌 예시: `dig www.example.com +trace`  
→ Root → TLD → Authoritative 응답 과정을 직접 확인 가능.

---

## 6. 클라우드 환경에서의 DNS
- **Public DNS**: 인터넷 전체에서 사용 가능 (Route53, Azure DNS, Cloudflare DNS)  
- **Private DNS**: VPC/VNet 내부에서만 사용 (Azure Private DNS, Route53 Private Hosted Zone)  
- **Hybrid 구조**: 온프렘 ↔ 클라우드 연결 시 Conditional Forwarding으로 특정 도메인만 내부 DNS로 전달  

---

## ✅ 면접 답변용 한 줄 요약
“DNS는 Root → TLD → Authoritative 단계로 도메인을 IP로 변환하며, Recursive DNS가 이를 대신 수행하고 Authoritative DNS는 최종 정답을 제공합니다. TTL과 캐싱 때문에 전파 지연이 발생할 수 있고, 클라우드에서는 Public/Private DNS로 구분해 운영합니다.”
