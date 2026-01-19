# 🏠 AI 호스트 기반 게스트하우스 플랫폼 (Refactoring)

> **"데이터가 없는 신규 숙소도 AI 컨설팅을 받을 수 있을까?"**<br/>
> 생성형 AI(Gemini)를 활용해 리뷰를 분석하고, 운영 솔루션을 제안하는 **B2B 호스트 관리 플랫폼**입니다. <br/>
> **단일 서버의 보안 취약점을 해결하기 위해 NCP VPC 환경에서 서버를 물리적으로 분리(User/Admin)했습니다.**

<br/>

## 🛠️ Tech Stack
| Category | Stack |
| :--- | :--- |
| **Backend** | ![Java](https://img.shields.io/badge/Java-17-blue) ![Spring Boot](https://img.shields.io/badge/SpringBoot-3.4-green) ![JPA](https://img.shields.io/badge/JPA-Hibernate-59666C) ![QueryDSL](https://img.shields.io/badge/QueryDSL-5.0-blue) |
| **Infra & DB** | ![NCP](https://img.shields.io/badge/Naver_Cloud-Platform-03C75A) ![Docker](https://img.shields.io/badge/Docker-2496ED) ![Nginx](https://img.shields.io/badge/Nginx-009639) ![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1) ![Redis](https://img.shields.io/badge/Redis-7.0-DC382D) |
| **AI & API** | ![Gemini](https://img.shields.io/badge/Google%20Gemini-Pro-8E75B2) ![RestTemplate](https://img.shields.io/badge/RestTemplate-Spring-6DB33F) |
| **Frontend** | ![Vue.js](https://img.shields.io/badge/Vue.js-3-4FC08D) ![Vite](https://img.shields.io/badge/Vite-646CFF) |

<br/>

## 🏗️ System Architecture & Infra
> **보안 강화를 위한 VPC 네트워크 분리 (Public / Private Zone)**

![System Architecture](./images/system.jpeg)

- **Reverse Proxy (Nginx):** Public Zone에 배치하여 외부 트래픽을 수신하고, 내부 Private Zone의 서버로 라우팅합니다.
- **Physical Separation:** `Main Server`(사용자용)와 `Admin Server`(관리자용)를 물리적으로 분리하여, 트래픽 폭주 시에도 관리자 기능의 안정성을 보장합니다.
- **Security:** DB와 Redis는 외부 접근이 완전히 차단된 Private Subnet에 배치하여 보안을 강화했습니다.

<br/>

## ⚡️ Key Troubleshooting (핵심 문제 해결)

### 1. AI 응답 파싱 오류 해결 및 Cold Start 전략
> **Issue:** Gemini API가 간헐적으로 비정형 데이터를 반환하여 파싱 에러 발생 & 신규 숙소 분석 불가.

<details>
<summary>👉 <b>해결 과정 자세히 보기 (Click)</b></summary>

**[원인]**
- LLM 특성상 프롬프트 제어에도 불구하고 응답 타입(String vs List)이 불규칙함.
- 초기 데이터가 없는(Cold Start) 숙소는 분석할 리뷰 데이터가 없어 서비스 가치가 떨어짐.

**[해결]**
1. **방어적 파싱 로직(`parseSafe`) 구현:** 응답 객체를 `Object`로 받아 `instanceof`로 타입을 검사한 후, 강제로 `List<String>` 형태로 정규화하여 **파싱 에러율 0%** 달성.
2. **이원화된 프롬프트 전략:**
    - 데이터 충분 시: `Review Analysis Mode` (리뷰 기반 장단점 분석)
    - 데이터 부족 시: `Trend Consulting Mode` (지역/시즌 트렌드 기반 조언 제공)

**[결과]**
- 리뷰가 0개인 숙소도 **100% 정보 제공**이 가능해져 초기 사용자 경험(UX) 개선.
</details>

### 2. 관리자 서버 물리적 분리 (Air-gap Deployment)
> **Issue:** 단일 서버 운영 시 보안 취약점 노출 및 트래픽 간섭 문제 발생.

<details>
<summary>👉 <b>해결 과정 자세히 보기 (Click)</b></summary>

**[해결]**
1. [cite_start]**서버 아키텍처 분리:** `User Server`(Public Subnet)와 `Admin Server`(Private Subnet)로 물리적 분리[cite: 5].
2. **폐쇄망 배포 파이프라인 구축:** 외부 인터넷이 차단된 Private Subnet에 배포하기 위해, Public Server를 경유하여 Docker 이미지를 전송하는 Air-gap 방식 적용.
3. [cite_start]**ACG(Firewall) 최적화:** `0.0.0.0/0` 허용을 제거하고, 내부 VPC 대역(`10.0.X.X`)과 Nginx IP에서만 접근 가능하도록 Inbound 규칙 엄격 제어[cite: 7].

**[결과]**
- 외부 공격으로부터 관리자 서버 원천 격리 및 트래픽 간섭 제거.
</details>

<br/>

## 💻 Service Demo
> **데이터 부족(Cold Start) 시에도 AI가 지역 트렌드를 분석하여 컨설팅을 제공합니다.**

![AI Demo](./images/demo.gif)
<br/>

## 📂 Project Structure
```bash
src
├── main
│   ├── java/com/geharbang
│   │   ├── domain
│   │   │   ├── host       # AI 리포트 생성 로직 (Prompt Engineering)
│   │   │   ├── settlement # 정산 및 매출 관리
│   │   │   └── ...
│   │   ├── global
│   │   │   ├── config     # NCP Object Storage & Gemini Config
│   │   │   └── security   # Spring Security 설정
...
