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

### Postman Collection

<img src="https://github.com/user-attachments/assets/fa1a4ee9-2b6b-4de9-a13b-b3920dde77e5" alt="Postman Collection 구성" width="300">

- API 엔드포인트는 `/api-01` ~ `/api-19`로 익명화되어 있습니다.
- API별로 폴더를 나누고, 각 폴더 안에 응답 코드 기준의 테스트 요청을 구성했습니다.
- 각 요청은 응답 코드에 따라 시나리오를 나눴습니다.
  - `SUCC-xxxx`: 정상 응답 검증 (쿼리 파라미터 조합, 이미지 크기/포맷별 케이스 등)
  - `FAIL-xxxx`: 얼굴 검출/베스트 샷 추출 실패 케이스
  - `ERR-xxxx`: 필수 필드 누락, 잘못된 파일 형식 등 요청 오류 케이스
- 각 요청의 `Tests` 탭에 JavaScript(Chai Assertion)로 검증 스크립트를 작성했습니다. 상세 내용은 아래 [Test Case Assertion](#test-case-assertion-javascript)을 참고하세요.

### Test Case Assertion (JavaScript)

<img src="https://github.com/user-attachments/assets/9ca7c36b-f392-4370-b29f-e92d3e33d34e" alt="Postman Test Script 예시" width="600">

요청마다 시나리오에 맞는 Assertion을 여러 개 조합해 작성했습니다. 사용한 검증 유형은 다음과 같습니다.

| # | 검증 유형 | 검증 내용 | 주요 Assertion |
|---|---|---|---|
| 1 | Status Code | 정상 케이스는 `200`, 오류 케이스는 `400`인지 확인 | `pm.response.to.have.status()` |
| 2 | 응답 시간 | 일반 API는 1000ms, 처리량이 많은 다중 이미지 API는 10000ms 이내인지 확인 | `to.be.below()` |
| 3 | 응답 형식 | 응답이 정상(`ok`)이고 Body가 있으며 JSON 형식인지 확인 | `to.be.ok` / `to.be.withBody` / `to.be.json` |
| 4 | 응답 구조 | 필수 필드가 있고 `error` 필드는 없는지 확인 | `to.have.jsonBody()` / `to.not.have.jsonBody()` |
| 5 | 결과 코드/메시지 | `return_code`, `return_msg`가 시나리오별 기대 값(`SUCC-0000`, `ERR-4003` 등)과 같은지 확인 | `to.be.eql()` / `to.have.property()` |
| 6 | null / undefined | 응답 값이 비어 있지 않은지 확인 | `to.not.be.null` / `to.not.be.undefined` / `to.not.be.oneOf([null, undefined])` |
| 7 | 데이터 타입 | 각 필드의 타입(`number`, `string`, `array`)이 맞는지 확인 | `to.be.a()` |
| 8 | 값 범위 | 신뢰도, 얼굴 좌표, 포즈 값이 유효 범위 안에 있는지 확인 (예: `confidence` 0.8 ~ 0.999) | `to.be.above().lessThan()` |
| 9 | 검출 결과 값 | 검출된 얼굴 수(`count`), 정렬 이미지 크기(112×112)와 채널 수(3) 등 기대 값 확인 | `to.be.eql()` |
| 10 | 실패 시 결과 없음 | 얼굴 검출 실패 시 결과 필드가 `null`인지 확인 | `to.be.null` |
| 11 | 오류 응답 | 오류 케이스에서 응답이 오류 상태인지 확인 | `to.not.be.ok` / `to.be.error` |
| 12 | API 간 결과 비교 | 앞선 API의 결과(얼굴 위치, 포즈, 특징값, 신뢰도 등)를 Environment 변수에 저장하고, 같은 이미지로 호출한 다른 API의 결과와 같은지 비교 | `pm.environment.set()` / `pm.environment.get()` + `to.be.eql()` |

```javascript
var responseData = pm.response.json();

// 1. Status Code
pm.test("Test1: Status code is 200", function () {
    pm.response.to.have.status(200);
});

// 2. 응답 시간
pm.test("Test2: Response time is less than 1000ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(1000);
});

// 5~7. 결과 코드, null/undefined, 타입
pm.test("Test3: To check Response Result return_code & Param Type", function () {
    pm.expect(responseData.return_code).to.not.be.oneOf([null, undefined]);
    pm.expect(responseData.return_code).to.be.a("string");
    pm.expect(responseData.return_code).to.be.eql("SUCC-0000");
});

// 3~4. 응답 형식과 구조
pm.test("Test4: response must be valid and have a body", function () {
    pm.response.to.be.ok;
    pm.response.to.be.withBody;
    pm.response.to.be.json;
    pm.response.to.have.jsonBody("return_code");
    pm.response.to.not.have.jsonBody("error");
});

// 8. 값 범위
pm.test("Test5: To check Response Result value & Param Type", function () {
    pm.expect(responseData.confidence).to.be.a("number");
    pm.expect(responseData.confidence).to.be.above(0.8).lessThan(0.999);
});

// 12. API 간 결과 비교 (앞선 API에서 저장한 값과 비교)
// 앞선 API 요청: pm.environment.set("FacesBoxX", pm.response.json().faces[0].box.x)
var environmentVariableValue = pm.environment.get("FacesBoxX");
var responseValue = responseData.result[0].box.x;
pm.test("Test6: To check /api-04 Response box.x Param and /api-08 Response box.x Param", function () {
    pm.expect(responseValue).to.be.eql(environmentVariableValue);
    pm.expect(responseValue).to.be.a("number");
});
```

## Jenkins Pipeline

<img src="https://github.com/user-attachments/assets/4072d255-cb76-4fc0-b0bd-1fe039770359" alt="Jenkins Pipeline" width="400">

`JenkinsFile`의 Pipeline 흐름은 다음과 같습니다.

1. **파라미터 입력**: Jenkins Job 실행 시 `URL` 파라미터로 테스트 대상 서버 주소를 받습니다.
2. **Environment 치환**: `sed`로 `environment.json`의 `URL` 값을 입력한 주소로 바꿉니다.
3. **Newman 실행**: Node.js와 Newman이 준비되어 있는지 확인한 뒤 Collection을 실행하고, `newman-reporter-allure`로 결과를 `allure-results`에 저장합니다.
4. **예외 처리**: 테스트 실행 중 오류가 나면 `currentBuild.result = 'FAILURE'`로 빌드를 실패 처리하고 오류 메시지를 남깁니다.
5. **리포트 게시**: `post { always { ... } }`에서 성공/실패와 관계없이 항상 Allure 리포트를 게시합니다.

```groovy
stage('run newman') {
    steps {
        script {
            try {
                def URL = params.URL
                sh '''
                sed -i "s;URL;${URL};g" environment.json
                '''
                runNewmanTests()
            } catch (Exception e) {
                currentBuild.result = 'FAILURE'
                echo "Error during 'run newman' stage: ${e.message}"
            }
        }
    }
}

post {
    always {
        publishAllureResults("allure-results")
    }
}
```

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
