# 🏠 지금이곳 - AI 기반 B2B 호스트 관리 플랫폼

<div align="center">

![Project Status](https://img.shields.io/badge/status-completed-success?style=flat-square)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.4-6DB33F?style=flat-square&logo=springboot)
![Java](https://img.shields.io/badge/Java-17-007396?style=flat-square&logo=openjdk)
![NCP](https://img.shields.io/badge/Naver_Cloud-VPC-03C75A?style=flat-square&logo=naver)
![Redis](https://img.shields.io/badge/Redis-7.x-DC382D?style=flat-square&logo=redis)

**핵심 가치:** #AI_Insight  #Zero_Trust_Infra  #Service_Stability  #Multi_Tier_Architecture

</div>

---

## 📑 목차
- [프로젝트 소개](#-프로젝트-소개)
- [기술 스택](#-기술-스택)
- [핵심 기여 및 기술적 성과](#-핵심-기여-및-기술적-성과)
- [시스템 아키텍처](#-시스템-아키텍처)
- [트러블 슈팅 (Deep Dive)](#-트러블-슈팅-deep-dive)
- [회고](#-회고)

---

## 🎯 프로젝트 소개

**"데이터 기반의 숙소 운영, AI 호스트 솔루션으로 완성하다"**

**지금이곳**은 중소형 게스트하우스 호스트를 위한 **AI 기반 컨설팅 및 관리 플랫폼**입니다. 생성형 AI(Google Gemini)를 활용해 흩어진 리뷰 데이터를 분석하여 실무적인 운영 리포트를 제공합니다. 특히, 사용자 트래픽과 관리 기능을 분리한 **Multi-tier 아키텍처**를 구축하여 시스템의 보안성과 확장성을 동시에 확보했습니다.

### 💡 핵심 해결 과제
- **Cold Start 문제:** 리뷰가 부족한 신규 숙소에 대한 데이터 기반 운영 가이드 제공
- **보안 위협:** 관리자 기능 및 DB 자원의 외부 노출 최소화
- **AI 불확실성:** LLM 응답의 비결정성(Non-deterministic)으로 인한 시스템 런타임 에러 방지

---

## 🛠️ 기술 스택

### Backend & AI
- **Core:** Java 17, Spring Boot 3.4, JPA (Hibernate), QueryDSL
- **Security:** Spring Security (JWT 기반 인증/인가)
- **AI:** Google Gemini Flash API (Prompt Engineering)

### Infrastructure & DevOps
- **Cloud:** Naver Cloud Platform (VPC, Subnet, ACG)
- **Server:** Nginx (Reverse Proxy), Docker, Ubuntu 22.04 LTS
- **Storage:** MySQL 8.0, Redis (Session/Cache), Object Storage

---

## 👨‍💻 핵심 기여 및 기술적 성과

### 1. AI 서비스의 가용성 및 신뢰성 설계 (High Availability)
- **Provider 패턴 도입:** `GEMINI`, `RULE`, `MOCK` 3가지 모드를 설정에 따라 전환하는 아키텍처를 설계하여 API 장애 시에도 서비스가 중단되지 않는 **Fallback 메커니즘**을 구축했습니다.
- **Cold Start 대응 로직:** 리뷰가 없는 숙소의 경우 지역/유형별 통계 데이터를 기반으로 한 `RULE` 기반 리포트를 생성하여 정보 제공률 100%를 달성했습니다.

### 2. Zero-Trust 기반의 보안 인프라 구축
- **Network Isolation:** NCP VPC 내에 Public/Private Subnet을 엄격히 분리했습니다.
- **폐쇄망 아키텍처:** Admin API 서버와 DB를 **Private Subnet**에 배치하고, 외부 접근을 원천 차단했습니다. 오직 Public Subnet의 Nginx 리버스 프록시와 Bastion Host를 통해서만 인가된 트래픽이 흐르도록 설계했습니다.

---

## 🏗️ 시스템 아키텍처
> **보안과 성능의 균형을 위해 외부 접점(Nginx)과 내부 자원(Admin/DB)을 물리적으로 격리했습니다.**

<div align="center">
<img src="images/system.jpeg" alt="System Architecture" width="90%">
</div>

---

## 🚒 트러블 슈팅 (Deep Dive)

<details>
<summary>👉 <b>1. LLM의 비결정적 응답 제어 및 방어적 파싱 (파싱 에러 0% 달성)</b></summary>

**[Situation]**
Gemini API가 요청한 JSON 규격과 다른 타입(String vs List)이나 불규칙한 Key 값을 반환하여 런타임에서 `ClassCastException`이 빈번하게 발생했습니다.

**[Action]**
- **Strict Prompting:** 응답 형식을 엄격히 규정하는 퓨샷(Few-shot) 프롬프트 엔지니어링을 적용.
- **Defensive Parsing Layer:** 응답을 `Object`로 수용 후 `instanceof` 검사를 수행하는 전용 헬퍼 메서드 구현.

```java
/* 예측 불가능한 AI 응답 타입을 안전하게 List<String>으로 정제 */
private List<String> convertToList(Object obj) {
    if (obj instanceof List<?>) {
        return ((List<?>) obj).stream()
                .map(Object::toString)
                .collect(Collectors.toList());
    }
    if (obj instanceof String) {
        return Arrays.asList(((String) obj).split(","));
    }
    return Collections.emptyList();
}

```

**[Result]**
비결정적인 AI 응답 환경에서도 **파싱 예외를 100% 핸들링**하여 시스템 안정성을 확보했습니다.

</details>

<details>
<summary>👉 <b>2. Multi-tier 환경의 보안 라우팅 및 클라이언트 IP 추적</b></summary>

**[Problem]**
관리자 서버를 Private Subnet으로 격리한 후, 리버스 프록시를 거치면서 실제 사용자의 IP가 내부 프록시 IP로 치환되어 보안 로그의 신뢰성이 하락했습니다.

**[Solution]**

* **Nginx Header Forwarding:** `X-Forwarded-For` 헤더를 통해 실제 IP를 전달하도록 설정.
* **VPC 내부 라우팅:** 보안 네트워크 내부 IP를 명시하여 트래픽 경로를 최적화.

```nginx
# Nginx 설정 
location /api/admin/ {
    proxy_pass [http://10.0.x.x:8081](http://10.0.x.x:8081); # Private Subnet 내 Admin 서버
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
}

```

**[Result]**
내부망 보안을 유지하면서도 **정확한 클라이언트 트래킹 및 보안 감사**가 가능한 환경을 구축했습니다.

</details>

<details>
<summary>👉 <b>3. Spring Profile을 활용한 논리적 환경 분리 및 인프라 최적화</b></summary>

**[Problem]**
사용자 API와 관리자 전용 로직이 단일 프로필에서 구동되어 DB 커넥션 풀 및 리소스 접근 권한 관리가 비효율적이었습니다.

**[Action]**

* **Profile Isolation:** `default`와 `admin` 프로필을 분리하여 환경별 리소스를 독립적으로 로드.
* **Conditional Configuration:** 관리자 프로필 가동 시에만 내부망 전용 Redis 세션 스토리지에 접근하도록 설정.

**[Result]**
운영 복잡도를 낮추고 각 환경에 최적화된 리소스 사용을 통해 시스템 효율성을 극대화했습니다.

</details>

---

## 🤔 회고

### 잘한 점 (Keep)

* **안정성 최우선 설계:** AI 장애 및 인프라 보안 사고에 대비한 다중 방어 체계를 구축한 경험이 큰 자산이 되었습니다.
* **인프라 역량 내재화:** 단순 코딩을 넘어 VPC와 Subnet 설계를 통해 트래픽의 흐름을 통제하는 엔지니어링 사고를 길렀습니다.

### 개선 방안 (Try)

* **CI/CD 자동화:** 현재의 수동 배포를 GitHub Actions 기반의 Blue-Green 무중단 배포 환경으로 고도화하고 싶습니다.
* **모니터링 강화:** Prometheus와 Grafana를 도입하여 인프라 자원 사용량을 실시간으로 시각화할 예정입니다.

---

## 📬 Contact

* **Email:** koo4934@gmail.com

* **Portfolio:** [https://geeunii.github.io](https://geeunii.github.io)

---
