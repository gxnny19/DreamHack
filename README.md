# DreamHack

Dreamhack 워게임의 코드 분석과 풀이 과정을 정리한 저장소입니다. **WEB**과 **PWN**으로 분야를 나누고, 각 분야 안에서 등급별로 문서를 관리합니다.

[WEB 풀이](#web) · [PWN 풀이](#pwn) · [폴더 구조](#폴더-구조) · [자료 출처](#자료-출처)

## 정리 현황

2026-11-17 기준, 저장소에 등록된 Markdown 문서 수입니다. 문제명과 등급은 현재 파일 경로를 기준으로 정리했습니다.

| 분야 | B4 | B3 | B2 | B1 | S4 | S3 | 합계 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| [WEB](WEB/) | 4 | 2 | 7 | 3 | 2 | 2 | **20** |
| [PWN](PWN/) | — | 1 | 1 | 1 | — | — | **3** |
| **전체** | **4** | **3** | **8** | **4** | **2** | **2** | **23** |

`B`는 Bronze, `S`는 Silver를 뜻합니다. 아래 목록에는 상세 풀이와 짧은 결과 메모가 함께 포함되어 있습니다. NoSQL 문제는 현재 목록에 포함되어 있지 않습니다.

## WEB

### Bronze 4 · B4

| 문제 | 핵심 주제 |
| --- | --- |
| [cookie](WEB/B4/B4%20cookie.md) | `username` 쿠키 변조와 관리자 권한 판정 |
| [pathtraversal](WEB/B4/B4%20pathtraversal.md) | `userid`를 이용한 내부 API 경로 조작 |
| [random-test](WEB/B4/B4%20random-test.md) | 응답 차이와 접두사 비교를 이용한 비밀번호 추론 |
| [session-basic](WEB/B4/B4%20session-basic.md) | 노출된 관리자 세션 ID를 이용한 인증 우회 |

### Bronze 3 · B3

| 문제 | 핵심 주제 |
| --- | --- |
| [ByPAssIF](WEB/B3/B3%20ByPAssIF.md) | 입력 조건 우회, 명령 실행과 시간 초과 예외의 KEY 노출 |
| [DreamDocs](WEB/B3/B3%20DreamDocs.md) | `X-User`·`Referer` 헤더를 신뢰하는 접근 제어 |

### Bronze 2 · B2

| 문제 | 핵심 주제 |
| --- | --- |
| [Another Ping](WEB/B2/B2%20Another%20Ping.md) | 쉘 명령어 치환과 `stderr`를 통한 결과 확인 |
| [ImageBox](WEB/B2/B2%20ImageBox.md) | 이미지 검사 조건 오류와 업로드된 PHP 실행 |
| [Simple Note Manager](WEB/B2/B2%20Simple%20Note%20Manager.md) | 백업 쿠키를 통한 OS Command Injection |
| [Tomcat Manager](WEB/B2/B2%20Tomcat%20Manager.md) | Path Traversal, 관리자 설정 확인과 WAR 배포 |
| [Tomcat Manager (ProcessBuilder 실습)](WEB/B2/B2%20Tomcat%20Manager%20-%20ProcessBuilder.md) | `flagapp.war`의 `flag.jsp`가 `ProcessBuilder`로 `/flag`를 실행해 FLAG를 획득하는 과정과 404 오류 해결 |
| [XSS-1](WEB/B2/B2%20XSS-1.md) | Reflected XSS와 봇의 FLAG 쿠키를 `/memo`로 전달하는 과정 |
| [access-log](WEB/B2/B2%20access-log.md) | 십진수 변환 결과를 기록한 짧은 메모 |

### Bronze 1 · B1

| 문제 | 핵심 주제 |
| --- | --- |
| [Are you admin?](WEB/B1/B1%20Are%20you%20admin.md) | 관리자 인증 헤더가 설정된 봇에서 XSS 실행 |
| [PTML](WEB/B1/B1%20PTML.md) | SVG 이벤트 속성과 `innerHTML`을 통한 Stored XSS |
| [Really Not SQL](WEB/B1/B1%20Really%20Not%20SQL.md) | WebDAV의 HTTP PUT으로 관리자 JSON 파일 수정 |

PTML 파일의 본문 제목은 현재 `Antigravity`로 표기되어 있습니다.

### Silver 4 · S4

| 문제 | 핵심 주제 |
| --- | --- |
| [baby-union](WEB/S4/S4%20baby-union.md) | UNION SQL Injection과 테이블·컬럼 구조 조회 |
| [Switching Command](WEB/S4/S4%20Switching%20Command.md) | PHP Type Juggling을 통한 인증 우회와 명령 실행 |

### Silver 3 · S3

| 문제 | 핵심 주제 |
| --- | --- |
| [CProxy: Inject](WEB/S3/S3%20CProxy%20Inject.md) | 비밀번호 객체가 SQL 비교식으로 변환되는 인증 우회 |
| [Padding Oracle](WEB/S3/S3%20Padding%20Oracle.md) | AES-CBC 패딩 오류 응답을 이용한 암호문 복호화 |

## PWN

| 등급 | 문제 | 핵심 주제 |
| --- | --- | --- |
| B3 | [flag-shop](PWN/B3/B3%20flag-shop.md) | 매번 섞이는 상품 목록을 파싱해 제한 시간 안에 50회 응답 |
| B2 | [addition-quiz](PWN/B2/B2%20addition-quiz.md) | 덧셈 문제를 파싱하고 시간 제한에 맞춰 답변 자동화 |
| B1 | [sint](PWN/B1/B1%20sint.md) | 음수 길이의 unsigned 변환, 스택 버퍼 오버플로와 신호 처리 |

## 폴더 구조

```text
DreamHack/
├── WEB/
│   ├── B4/
│   ├── B3/
│   ├── B2/
│   ├── B1/
│   ├── S4/
│   └── S3/
├── PWN/
│   ├── B3/
│   ├── B2/
│   └── B1/
├── assets/                 # 공통 이미지
├── AI_WRITEUP_HANDOFF.md    # 기존 문서 작성 지침
└── README.md
```

- 풀이 문서는 `분야/등급/등급 문제명.md` 형식으로 저장합니다.
- 문제별 이미지는 각 등급 폴더의 `assets/` 또는 문서에서 참조하는 하위 폴더에 있습니다. 일부 문서는 GitHub 첨부 이미지를 사용합니다.
- 상세 풀이에서는 코드·요청 분석, 페이로드 또는 자동화 스크립트, 실행 결과를 순서대로 확인할 수 있습니다. 문서마다 기록 범위는 다릅니다.

## 자료 출처

기존 풀이 자료는 [myhtml1231-star/write-up](https://github.com/myhtml1231-star/write-up)을 바탕으로 가져왔으며, 현재는 [gxnny19/DreamHack](https://github.com/gxnny19/DreamHack)에서 독립적으로 관리합니다. 이 README의 목차와 주제 요약은 현재 저장소에 있는 문서를 기준으로 작성했습니다.
