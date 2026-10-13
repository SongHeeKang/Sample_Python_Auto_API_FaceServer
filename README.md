# Sample_Python_Auto_API_FaceServer

> ⚠️ **본 프로젝트는 포트폴리오용으로 제작된 샘플입니다.**
>
> 실무에서 진행한 API 테스트 자동화 작업을 바탕으로 재구성했으며,
> 보안 및 개인정보 보호를 위해 서버 주소, API Key, API 엔드포인트 경로, 계정/식별 정보, 테스트 이미지 등
> 민감한 정보는 모두 제거하거나 임의의 값(`xxx`, `api-01` 등)으로 대체했습니다.
> 따라서 이 저장소의 코드는 실제 서버에 연결되어 동작하지 않습니다.

## 개요

얼굴 인식 서버(Face Server) API를 대상으로 **Postman + Newman + Jenkins** 기반의 API 테스트 자동화를 구성한 프로젝트입니다.

- Postman Collection으로 API별 정상/예외 케이스 테스트 시나리오 작성
- Newman CLI로 Collection을 실행하고 Allure 리포트 생성
- Jenkins Pipeline으로 테스트 실행부터 리포트 게시까지 자동화

## 기술 스택

| 구분 | 사용 도구 |
|---|---|
| API 테스트 | Postman (Test Script: JavaScript, Chai Assertion) |
| 테스트 실행 | Newman |
| CI | Jenkins Pipeline |
| 리포트 | Allure (`newman-reporter-allure`) |

## 프로젝트 구조

```
.
├── JenkinsFile        # Jenkins Pipeline (Newman 실행 + Allure 리포트 게시)
├── collection.json    # Postman Collection (19개 API 그룹, 115개 테스트 요청)
├── environment.json   # Postman Environment (URL, API_KEY 등 변수, 값은 마스킹됨)
└── images/            # 테스트 이미지 폴더 (개인정보 보호를 위해 파일 제외, 필요 파일 목록만 기재)
```

## 테스트 구성

- API 엔드포인트는 `/api-01` ~ `/api-19`로 익명화되어 있습니다.
- 각 요청은 응답 코드에 따라 시나리오를 나눴습니다.
  - `SUCC-xxxx`: 정상 응답 검증 (쿼리 파라미터 조합, 이미지 크기/포맷별 케이스 등)
  - `FAIL-xxxx`: 얼굴 검출/베스트 샷 추출 실패 케이스
  - `ERR-xxxx`: 필수 필드 누락, 잘못된 파일 형식 등 요청 오류 케이스
- 각 요청의 Test Script에서 다음 항목을 검증합니다.
  - HTTP Status Code 및 응답 시간 (1000ms 이내)
  - 응답 Body 구조, 필수 파라미터 존재 여부 및 타입
  - `return_code` / `return_msg` 값
  - API 간 결과 값 비교 (Environment 변수에 저장한 값을 다른 API 응답과 비교)

## 실행 방법

```bash
npm install -g newman newman-reporter-allure

# environment.json의 URL, API_KEY 등을 실제 값으로 설정하고
# images/ 폴더에 테스트 이미지를 준비한 후 실행
newman run collection.json -e environment.json --insecure \
  -r allure --reporter-allure-export allure-results

allure serve allure-results
```

Jenkins에서는 `JenkinsFile`의 Pipeline을 사용하며, `URL` 파라미터로 대상 서버 주소를 전달합니다.
