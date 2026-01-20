# 🏠 지금이곳 - AI 기반 게스트하우스 플랫폼

<div align="center">

![Project Status](https://img.shields.io/badge/status-completed-success?style=flat-square)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=flat-square&logo=springboot)
![NCP](https://img.shields.io/badge/Naver_Cloud-Platform-03C75A?style=flat-square&logo=naver)
![Vue.js](https://img.shields.io/badge/Vue.js-3.x-4FC08D?style=flat-square&logo=vuedotjs)

**프로젝트명:** 지금이곳 | **프로젝트 유형:** K-Digital Training 최종 프로젝트

**프로젝트 기간:** 2025.12.10 ~ 2026.01.15

**핵심 키워드:** #AI호스트솔루션 #서버분리 #보안강화 #프롬프트엔지니어링

</div>

---

## 📑 목차

- [프로젝트 소개](#-프로젝트-소개)
- [기술 스택](#-기술-스택)
- [핵심 기여 및 역할](#-핵심-기여-및-역할)
- [시스템 아키텍처](#-시스템-아키텍처)
- [데이터베이스 설계](#-데이터베이스-설계)
- [트러블 슈팅](#-트러블-슈팅)
- [서비스 시연](#-서비스-시연)

---

## 🎯 프로젝트 소개

**"신규 숙소도 데이터 기반의 AI 컨설팅을 받을 수 있을까?"** 라는 질문에서 출발했습니다.  
생성형 AI(Google Gemini)를 활용해 숙소 리뷰, 트렌드를 분석하고 운영 솔루션을 제안하는 **B2B 호스트 관리 플랫폼**입니다. 기존의 모놀리식 구조를 개선하여, **NCP VPC 환경에서 User/Admin 서버를 논리적으로 분리**하고 보안을 강화했습니다.

### 💡 기획 배경

- 신규 호스트는 리뷰 데이터가 부족하여 객관적인 숙소 상태 파악이 어려움 (Cold Start 문제)
- 트래픽이 몰릴 경우, 관리자 기능(정산, 모니터링)까지 영향을 받는 구조적 문제 해결 필요
- 단순 통계를 넘어선, AI 기반의 개인화된 인사이트 제공 니즈 증가

---

## 🛠️ 기술 스택

- **Backend:** Spring Boot 3.4, Java 17, JPA, QueryDSL, Spring Security
- **Database:** MySQL 8.0, Redis (for Session/Cache)
- **Infra & DevOps:** Naver Cloud Platform (VPC, Server, Object Storage), Nginx, Docker
- **AI & External API:** Google Gemini Flash, Toss Payments, Kakao Map, CoolSMS
- **Frontend:** Vue.js 3, Vite, Pinia

---

## 👨‍💻 핵심 기여 및 역할

프로젝트의 **관리자 및 호스트 기능 개발을**했으며, **AI 기능 설계부터 인프라 구조 개선**에 이르기까지 핵심적인 역할을 수행했습니다. (총 6인 중 기여도 약 25%)

### 1. AI 기반 호스트 리포트 시스템 설계 및 개발

**[문제 정의]** 신규 호스트는 리뷰가 없어 숙소의 장단점을 파악하기 어렵고, AI 응답의 불확실성은 서비스 안정성을 저해하는 핵심 문제였습니다.

**[해결 과정]**

- **AI Provider 모델 설계 (`HostAiInsightService.java`):**
  안정성과 확장성을 고려하여 설정(`ai.summary.provider`)에 따라 AI 모델을 선택하는 Provider 패턴을 도입했습니다.
  - **`GEMINI`:** 리뷰 데이터가 충분할 경우, Gemini API를 호출하여 심층 분석 리포트를 생성합니다. API 호출 실패 시에는 **안정성 확보를 위해 `RULE` 기반 로직으로 자동 폴백(Fallback)** 됩니다.
  - **`RULE`:** 리뷰가 부족한 **Cold Start** 상황에서, 지역 및 숙소 유형에 기반한 "초기 운영 가이드"를 제공하여 **모든 호스트에게 100% 정보 제공률을 달성**했습니다.
  - **`MOCK`:** 개발 및 테스트 단계에서 사용되는 Mock 데이터 제공 로직입니다.

- **프롬프트 엔지니어링 (`buildReviewPrompt`):**
  Gemini가 일관된 고품질의 JSON을 반환하도록, `String.format`을 활용해 **동적으로 프롬프트를 생성**했습니다. 총평, 좋았던 점, 개선 포인트 등 **섹션 순서와 출력 형식을 명확히 지정**하여 응답 데이터의 정합성을 확보했습니다.

- **방어적 파싱 로직 (`parseSafe`):**
  AI가 예상치 못한 포맷(e.g., `List` 대신 `String`)을 반환해도 **`ClassCastException` 없이** 안전하게 파싱하는 `parseSafe` 로직을 구현했습니다. AI 응답을 `Object`로 받은 뒤, 타입 검사를 통해 유연하게 `List<String>`으로 변환하여 **AI 응답 파싱 에러율 0%** 를 달성했습니다.

### 2. VPC 기반의 안전한 서버 인프라 구축

**[문제 정의]** 단일 서버 구조는 보안에 취약하며, 사용자 트래픽이 관리자 기능에 직접적인 영향을 미치는 위험이 있었습니다.

**[해결 과정]**

- **네트워크 및 서버 환경 분리:**
  - **네트워크 분리:** NCP의 VPC를 활용하여 외부 접근이 가능한 `Public Subnet`과 내부에서만 통신하는 `Private Subnet`을 설계했습니다.
  - **서버 환경 분리:** User API 서버와 Bastion Host는 Public Subnet에, **Admin API 서버와 DB(MySQL, Redis)는 Private Subnet에 배치**하여 외부 직접 접근을 원천 차단했습니다.
- **Spring Profile을 활용한 논리적 분리 (`application-admin.yml`):**
  하나의 Spring Boot 프로젝트를 `default`(사용자용)와 `admin` 프로필로 나누어 배포했습니다. **`admin` 프로필 활성화 시, DB와 Redis 등 모든 인프라 연결 정보를 Private IP로 바라보도록 설정**하여 환경을 완벽히 분리했습니다.
- **Nginx 리버스 프록시를 통한 보안 라우팅:**
  Public Subnet의 Nginx가 모든 요청을 받아, `/api/admin/**` 경로는 Private Subnet의 Admin 서버(`10.0.x.x:8081`)로, 그 외 API는 User 서버로 안전하게 라우팅합니다. 또한, `proxy_set_header X-Real-IP $remote_addr` 설정을 통해 **백엔드 서버가 실제 클라이언트 IP를 추적**할 수 있도록 하여 로깅 및 보안 분석의 정확도를 높였습니다.

---

## 📐 시스템 아키텍처

**보안을 최우선으로 고려하여 NCP VPC 환경 내에서 Public/Private Subnet을 분리 설계했습니다.** 외부 요청은 오직 Nginx를 통해서만 인가된 서버로 라우팅되며, DB와 Admin 서버는 폐쇄망에 배치하여 외부 위협을 원천 차단했습니다.

<div align="center">
  <img src="images/system.jpg" alt="System Architecture" width="90%">
</div>

---

## 💾 데이터베이스 설계

**설계 목표:** 호스트, 숙소, 예약, 리뷰, 정산 등 각 도메인의 역할을 명확히 분리하고, 기능 확장에 유연하게 대처할 수 있는 구조를 만드는 데 집중했습니다.

- **정규화:** 데이터 무결성을 위해 정규화를 진행하고, 주요 비즈니스 로직(숙소-예약, 사용자-리뷰)에 외래 키 제약 조건을 설정했습니다.
- **인덱싱:** 예약 조회, 리뷰 검색 등 주요 조회 성능에 영향을 미치는 컬럼(e.g., `accommodation_id`, `created_at`)에 인덱스를 적용하여 검색 속도를 최적화했습니다.
- **초기 데이터 (`seed`):** `backend/src/main/resources/db/seed/` 경로에 초기 데이터를 관리하여, 어떤 개발 환경에서든 동일한 테스트 데이터를 기반으로 개발할 수 있도록 환경을 통일했습니다.

### ERD (Entity-Relationship Diagram)

<div align="center">
  <img src="images/erd.jpg" alt="ERD Diagram" width="95%">
</div>

---

## 🚒 트러블 슈팅

> **프로젝트 진행 중 발생한 주요 기술적 이슈와 해결 과정입니다.**

<details>
<summary><strong>👉 1. AI 응답 불확실성 및 데이터 파싱 오류 (ClassCastException)</strong></summary>

- **[문제]** Gemini API에게 `List<String>` 형태를 요청했으나, 간헐적으로 단순 `String`("아이템1, 아이템2")이나 포맷이 깨진 데이터를 반환하여 `ClassCastException`과 함께 500 에러가 발생했습니다.
- **[해결]** `parseSafe`와 `convertToList`라는 방어적 파싱 로직을 2단계로 구현했습니다. AI 응답을 `Object`로 받은 뒤, `instanceof`로 타입을 확인하고 다양한 케이스에 대응하여 `List<String>`으로 안전하게 변환함으로써 **파싱 에러율 0%** 를 달성했습니다.

  ```java
  // HostAiInsightService.java

  /**
   * AI가 반환한 JSON 문자열을 안전하게 Map으로 파싱하고,
   * 예측 불가능한 Key와 Value 형태를 표준 포맷으로 정제합니다.
   */
  private Map<String, Object> parseSafe(String jsonString) {
      try {
          // 1. 응답을 Map<String, Object>로 유연하게 파싱
          Map<String, Object> map = objectMapper.readValue(jsonString, new TypeReference<>() {});

          // 2. AI가 혼용한 키 값들을 표준 키(e.g., summary, pros)로 보정
          if (map.containsKey("overview")) map.put("summary", map.get("overview"));
          if (map.containsKey("strength")) map.put("pros", map.get("strength"));
          if (map.containsKey("weakness")) map.put("cons", map.get("weakness"));
          if (map.containsKey("improvements")) map.put("cons", map.get("improvements"));

          // 3. 모든 필드를 convertToList를 통해 List<String>으로 강제 변환
          map.put("pros", convertToList(map.get("pros")));
          map.put("cons", convertToList(map.get("cons")));
          map.put("actions", convertToList(map.get("actions")));
          map.put("monitoring", convertToList(map.get("monitoring")));

          map.putIfAbsent("summary", "데이터 분석을 완료했습니다.");
          return map;
      } catch (Exception e) {
          log.error("AI 응답 파싱 실패: {}", jsonString, e);
          // 파싱 실패 시에도 안정적인 UI 렌더링을 위해 Fallback 객체 반환
          return Map.of(
              "summary", "AI 응답을 분석하는 중 오류가 발생했습니다.",
              "pros", List.of("분석 실패"),
              "actions", List.of("잠시 후 다시 시도해주세요.")
          );
      }
  }

  /**
   * Object 타입의 값을 어떤 경우에도 List<String>으로 변환합니다.
   * - Case 1: 정상적인 List -> 그대로 반환
   * - Case 2: "[item1, item2]" 형태의 문자열 -> JSON 배열로 파싱
   * - Case 3: "item1, item2" 형태의 문자열 -> 쉼표로 분리
   */
  private List<String> convertToList(Object obj) {
      if (obj == null) return new ArrayList<>();

      if (obj instanceof List<?>) {
          return ((List<?>) obj).stream()
                  .map(item -> Objects.toString(item, "").replace("**", ""))
                  .collect(Collectors.toList());
      }

      if (obj instanceof String) {
          String str = (String) obj;
          // Case 2: 문자열이 JSON 배열 형태일 경우
          if (str.startsWith("[") && str.endsWith("]")) {
              try {
                  return objectMapper.readValue(str, new TypeReference<List<String>>() {});
              } catch (Exception e) {
                  // JSON 파싱 실패 시 쉼표 기준으로 분리
                  return Arrays.stream(str.substring(1, str.length() - 1).split(","))
                          .map(String::trim).filter(s -> !s.isEmpty()).collect(Collectors.toList());
              }
          }
          // Case 3: 일반 문자열일 경우
          return Arrays.stream(str.split(","))
                  .map(String::trim).filter(s -> !s.isEmpty()).collect(Collectors.toList());
      }

      return List.of(obj.toString());
  }

```
```
</details>

<details>
<summary><strong>👉 2. Nginx 502 Bad Gateway (리버스 프록시 라우팅 실패)</strong></summary>

* **[문제]** 서버 분리 후, 관리자 페이지 접속 시 502 에러가 발생했습니다. Public Subnet의 Nginx가 Private Subnet에 있는 Admin Server(`10.0.x.x:8081`)를 찾지 못하는 문제였습니다.
* **[해결]** Nginx의 `location /api/admin/` 블록에 `proxy_pass` 대상으로 `localhost`가 아닌 **Private IP를 명시**하여 라우팅 경로를 확정했습니다. 이를 통해 외부 요청이 Nginx를 통해서만 내부망 서버에 도달하도록 강제했습니다.
```nginx
# /etc/nginx/sites-available/default
server {
    listen 80;
    server_name your_domain.com;

    # ... (기타 설정) ...

    # User API 서버 (Public Subnet 내 위치)
    location /api/ {
        proxy_pass http://localhost:8080; # or 127.0.0.1:8080
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
    }

    # Admin API 서버 (Private Subnet 내 위치)
    location /api/admin/ {
        # Admin 서버의 Private IP를 직접 지정하여 Private Subnet으로 라우팅
        proxy_pass [http://10.0.](http://10.0.)x.x:8081; # 보안상 마스킹 처리 (실제 Private IP)
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
    }
}

```


</details>

<details>
<summary><strong>👉 3. Docker "Bind for 0.0.0.0:8080 failed" (좀비 포트 충돌)</strong></summary>

* **[문제]** `docker run` 시 8080 포트가 이미 사용 중이라는 에러가 발생했습니다. `netstat -tnlp`으로 확인해도 점유 프로세스가 보이지 않는 'Ghost Port' 현상이었습니다.
* **[해결]** `systemctl stop docker.socket`으로 소켓을 정리하고, User/Admin 서버의 Host 포트를 각각 `8080`, `8081`로 명확히 분리하여 충돌을 원천적으로 방지했습니다.
```bash
# 1. (필요 시) 포트 점유의 원인이 될 수 있는 Docker 소켓 서비스를 중지
$ sudo systemctl stop docker.socket

# 2. User/Admin 서버 실행 시 Host 포트를 명시적으로 분리하여 실행
# User Server
$ docker run -d -p 8080:8080 --name user-server \
  -e "SPRING_PROFILES_ACTIVE=default" your-app-image

# Admin Server
$ docker run -d -p 8081:8081 --name admin-server \
  -e "SPRING_PROFILES_ACTIVE=admin" your-app-image

```



</details>

---

## 💻 서비스 시연

> **데이터 부족(Cold Start) 시에도 AI가 지역 트렌드를 분석하여 컨설팅을 제공합니다.**

*(서비스 동작을 보여주는 GIF나 동영상 링크를 추가하면 좋습니다)*

---

## 📫 Contact

**Email:** koo4934@gmail.com

**Portfolio:** https://geeunii.github.io

```

```
