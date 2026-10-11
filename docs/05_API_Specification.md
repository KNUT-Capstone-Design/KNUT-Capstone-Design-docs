# 스미싱 예방 교육 및 탐지 시스템 API 명세서

- **버전:** Draft v0.2 / API v1
- **상태:** 팀 검토용 구현 계약 초안
- **관련 문서:** [기능 명세](01_Feature_Specification.md), [사용자 흐름](02_User_Flow.md), [아키텍처](03_System_Architecture.md)
- **범위:** 비회원 URL·문자·QR 분석 및 서버 상태 확인

## 1. 범위와 계약 원칙

교육·퀴즈는 OCI Docker PostgreSQL에 저장하고 BE API로 조회·채점한다. 데이터 모델은 `04_Database_Design.md`를 따른다. 회원, 검사 이력, 관리자, OCR API는 MVP 범위 밖이다.

본 문서의 길이 계산, URL 개수, 시간 제한 등 기존 문서에 없던 수치는 구현을 위한 제안값이다. 팀 검토 후 확정한다. 규칙별 배점과 정확도는 아직 검증되지 않았으며 점수는 악성 확률을 뜻하지 않는다.

API 목록:

- `POST /api/v1/analysis/url`: URL 한 개 분석 — F-101
- `POST /api/v1/analysis/message`: 문자와 포함 URL 분석 — F-102
- `POST /api/v1/analysis/qr`: QR 이미지 분석 — F-103
- `GET /api/v1/health`: DB를 포함한 서비스 준비 상태 확인
- `GET /api/v1/education/cases`: 공개 교육 사례 목록
- `GET /api/v1/education/cases/{id}`: 공개 사례와 예방수칙 상세
- `GET /api/v1/quizzes/current`: 현재 공개된 10문제 세트
- `POST /api/v1/quizzes/grade`: 해당 버전 답안 채점

요청 하나에서 분석을 완료하는 동기 API로 구성한다. 작업 조회·폴링 API와 결과 영구 저장은 제공하지 않는다.

## 2. 공통 통신 규칙

- 운영 서버 주소는 배포 후 확정하고 FE 환경변수로 관리한다.
- 운영 통신은 HTTPS, JSON은 UTF-8을 사용한다.
- JSON 요청은 `Content-Type: application/json`을 사용한다.
- QR 업로드는 `multipart/form-data`이며 브라우저가 boundary를 생성하게 한다.
- MVP는 인증 없이 이용한다. 외부 평판 서비스 키는 BE에만 둔다.
- 필드명은 `snake_case`, enum 값은 영문 소문자를 사용한다.
- 문서에 정의한 응답 필드는 생략하지 않는다. 해당 값이 없으면 명시된 `null` 또는 빈 배열을 사용한다.
- BE는 매 요청에 UUID 형식의 `request_id`를 생성하고 응답 본문과 `X-Request-ID` 헤더에 동일하게 반환한다.
- 시각은 UTC RFC 3339 문자열이다. 예: `2026-10-11T12:00:00Z`.
- 분석 응답과 오류 응답에 `Cache-Control: no-store`를 적용한다.
- CORS는 허용한 FE 출처에만 적용하고 `X-Request-ID`, `Retry-After`를 FE에 노출한다.
- FE는 알려지지 않은 추가 응답 필드를 무시한다. 필드 제거, 타입 변경, enum 변경은 계약 검토 대상이다.

## 3. 입력 제한

### 3.1 공통 검증

JSON 필수 필드 누락, 타입 오류, 정의하지 않은 필드는 `422 VALIDATION_ERROR`로 처리한다. JSON 문법 오류는 `400 INVALID_JSON`이다. 요청 본문 전체를 오류에 포함하지 않는다.

문자 수는 Unicode code point 기준이다. FE는 JavaScript 문자열의 단순 `.length` 대신 `Array.from(value).length`를 사용해 Python 측과 일치시킨다. 결합 문자와 이모지의 화면 표시 개수와는 다를 수 있다.

### 3.2 URL

- 앞뒤 공백 제거 후 1~2,048자.
- 절대 HTTP 또는 HTTPS URL이며 hostname이 필요하다. 프로토콜은 자동으로 추정하지 않는다.
- 사용자명·비밀번호가 포함된 URL과 제어문자를 거부한다.
- 정규화는 scheme·hostname 소문자화와 IDN hostname 변환에 한정한다. 경로·쿼리의 의미를 바꾸지 않는다.
- 분석 대상에 HTTP 요청, DNS 조회, 단축 URL 확장, 리디렉션 추적을 수행하지 않는다.
- 외부 평판 조회는 별도 승인된 제공자에 대한 요청이며 대상 사이트 방문과 구분한다.

### 3.3 문자

- 공백을 포함한 원본 기준 1~5,000자. 공백만 있는 입력은 거부한다.
- 원문 순서와 줄바꿈을 유지한다. 원문 전체는 응답에 복제하지 않는다.
- 명시적 `http://`·`https://` URL을 등장 순서대로 추출한다. 프로토콜 없는 주소는 MVP 추출 대상에서 제외한다.
- URL 정규화 후 중복을 제거하고 서로 다른 URL을 최대 5개 분석한다.
- 6개 이상이면 앞의 5개만 분석하며 `partial` 및 `URL_LIMIT_EXCEEDED` 경고를 반환한다.
- 유효하지 않은 URL 후보는 해당 항목만 실패 처리하고 문자 패턴 분석은 계속한다.

### 3.4 QR 파일

- 필드명 `file`, 파일 1개만 허용한다. 추가 파일·폼 필드는 거부한다.
- PNG/JPG/JPEG, MIME `image/png` 또는 `image/jpeg`.
- 파일 자체 최대 **10,000,000 bytes**. 기존 문서의 10MB를 이 값으로 해석한다.
- multipart 오버헤드를 포함한 요청 전체는 최대 11,000,000 bytes.
- 가로와 세로가 각각 1~4,096픽셀이어야 한다.
- 확장자, 선언 MIME, 실제 파일 형식 및 디코딩 결과를 서버에서 검증한다.
- 애니메이션 PNG, SVG, PDF, HEIC 및 손상 이미지를 거부한다.
- QR 하나만 지원한다. 복수 QR을 발견하면 임의로 선택하지 않고 다시 잘라서 제출하도록 안내한다.
- UTF-8 텍스트로 디코딩할 수 없는 QR은 지원하지 않는다.
- 이미지 디코딩 전에 가능한 헤더 크기 검사를 수행하고 디코더의 자원 한도를 적용한다.

## 4. 공통 성공 응답 모델

HTTP `200`의 JSON 구조는 다음과 같다. 정확한 타입은 11절에 정의한다.

- `request_id`: 요청 추적 식별자. 결과 재조회 토큰이 아니다.
- `data.type`: `url`, `message`, `qr`.
- `data.status`: `completed`, `partial`, `not_applicable`.
- `data.analyzed_at`: 분석 완료 시각.
- `data.analysis_version`: 적용한 규칙·배점 버전. 예: `rules-v0.1.0`.
- `data.risk`: 공통 위험 평가 또는 `null`.
- `data.reputation_status`: `disabled`, `completed`, `partial`, `failed`, `not_applicable`.
- `data.warnings`: 코드와 표시용 메시지 목록.
- `data.details`: 검사 유형별 상세 정보.

`completed`는 설정된 분석이 완료됐다는 뜻이다. 안전을 보증하거나 웹페이지 내용을 검사했다는 의미가 아니다. 외부 평판 기능 자체가 꺼져 있으면 `reputation_status=disabled`이고 분석은 `completed`가 될 수 있다.

`partial`은 문자 패턴 등 유효한 일부 결과가 있지만 URL 일부나 활성화된 외부 조회가 실패·생략된 경우다. FE는 결과 위에 분석 제한 경고를 표시한다.

`not_applicable`은 URL이 아닌 QR 데이터에 사용한다. `risk=null`이며 점수와 안전 등급을 표시하지 않는다. 기능 명세와 사용자 흐름에도 동일한 예외를 반영했다.

### 4.1 위험 평가

- `score`: 0~100 정수. 탐지 규칙 점수 합산 후 상한 100 적용.
- `base_level`: 점수에 따른 `safe`(0~29), `caution`(30~59), `danger`(60~100).
- `level`: 치명 규칙까지 반영한 최종 등급. FE가 재계산하지 않는다.
- `overridden`: 기본 등급을 치명 규칙으로 상향했을 때만 `true`.
- `critical_rule_ids`: 매칭된 치명 규칙 ID 목록. 원래부터 60점 이상이면 이 목록이 있어도 `overridden=false`일 수 있다.
- `summary`: 판정 근거 요약. 위험 신호가 없을 때도 검사 범위와 한계를 설명한다.
- `findings`: 매칭된 위험 규칙. 없으면 빈 배열이며 가짜 위험 근거를 생성하지 않는다.
- `recommendations`: 사용자 행동 코드와 설명. 최소 1개.

각 finding은 `rule_id`, `severity`(`info`, `warning`, `critical`), `description`을 포함한다. 원문·인증번호·계좌번호를 description에 복제하지 않는다.

문자는 문자 패턴 평가 점수와 성공한 URL 평가 점수 중 최댓값을 종합 점수로 사용한다. 치명 규칙은 성공한 모든 평가에서 합집합으로 반영한다. 동일 URL 반복으로 점수가 올라가지 않는다. 이 집계 정책과 규칙별 가중치는 서로 구분하며 배점은 별도 테스트 후 확정한다.

APK 언급이나 공식 서비스의 일반 로그인 안내만으로 치명 규칙을 확정하지 않는다. 문맥·도메인 등 구체적인 판정 조건은 규칙 정의에서 검증한다. 웹페이지 내부 폼 존재 여부는 MVP 분석 범위에 포함하지 않는다.

## 5. URL 분석

`POST /api/v1/analysis/url`

요청:

```json
{
  "url": "https://example.com/"
}
```

HTTP 200 예시:

```json
{
  "request_id": "65122d8f-7b10-4bb3-a936-0c0c3b837239",
  "data": {
    "type": "url",
    "status": "completed",
    "analyzed_at": "2026-10-11T12:00:00Z",
    "analysis_version": "rules-v0.1.0",
    "risk": {
      "score": 0,
      "base_level": "safe",
      "level": "safe",
      "overridden": false,
      "critical_rule_ids": [],
      "summary": "URL 문자열 검사에서 위험 신호가 발견되지 않았습니다. 웹페이지 내용은 검사하지 않았습니다.",
      "findings": [],
      "recommendations": [
        { "code": "VERIFY_OFFICIAL_CHANNEL", "message": "공식 앱이나 공식 연락처로 확인하세요." }
      ]
    },
    "reputation_status": "disabled",
    "warnings": [],
    "details": {
      "normalized_url": "https://example.com/",
      "hostname": "example.com",
      "scheme": "https"
    }
  }
}
```

예시 점수는 응답 형식을 설명하기 위한 값이며 탐지 정확도 검증 결과가 아니다. 응답 URL은 텍스트로 표시하고 자동 링크·미리보기로 만들지 않는다.

## 6. 문자 분석

`POST /api/v1/analysis/message`

요청:

```json
{
  "message": "배송 안내를 확인하세요. https://example.com/"
}
```

공통 응답에서 `type=message`를 사용한다. `details`의 형태는 다음과 같다.

```json
{
  "character_count": 34,
  "url_candidate_count": 1,
  "analyzed_url_count": 1,
  "omitted_url_count": 0,
  "matched_spans": [],
  "url_results": [
    {
      "index": 0,
      "url": "https://example.com/",
      "status": "completed",
      "score": 0,
      "level": "safe",
      "error_code": null
    }
  ]
}
```

`character_count`는 실제 요청 문자열에서 계산한다. 위 요청 예시는 공백 포함 34 code point다.

- `url_candidate_count`: 중복 제거 후 URL 후보 수.
- `analyzed_url_count`: 내부 URL 분석에 성공한 수. 외부 조회만 실패한 항목도 포함.
- `omitted_url_count`: 5개 제한 때문에 시도하지 않은 수.
- `url_results`: 시도한 URL 최대 5개. `index`는 0부터 시작하는 목록 인덱스다.
- URL 항목 `status`: `completed`, `partial`, `failed`. 실패 시 `score`와 `level`은 `null`, `error_code`는 오류 코드다.
- `matched_spans`: `{ "start": 0, "end": 2, "rule_id": "MSG_EXAMPLE" }` 형식. 원본 메시지의 code point 기준이며 시작 포함·끝 제외다. FE는 원본을 보유해 해당 범위를 강조한다.
- 원본을 정규화해서 검사하는 BE는 원본 오프셋으로 되돌려 반환해야 한다.

URL이 없으면 URL 관련 count는 0, 결과 배열은 빈 배열이며 문자 패턴만 평가한다. URL이 일부 실패하면 `status=partial`, `URL_ANALYSIS_FAILED` 경고와 유효한 문자 평가를 반환한다. 실패한 URL을 0점 안전으로 처리하지 않는다.

## 7. QR 분석

`POST /api/v1/analysis/qr`

요청은 multipart 필드 `file`에 이미지 바이너리를 담는다. 추가 JSON 입력은 없다.

HTTP 200에서 `type=qr`, `details`는 다음 형태다.

```json
{
  "content_type": "url",
  "decoded_text": "https://example.com/",
  "normalized_url": "https://example.com/",
  "image_width": 512,
  "image_height": 512
}
```

- `content_type`: `url` 또는 `text`.
- 유효한 HTTP(S) URL은 URL 분석과 동일한 위험 평가를 반환한다.
- 일반 텍스트와 HTTP(S) 이외의 URI는 `content_type=text`, `normalized_url=null`, `status=not_applicable`, `risk=null`, `reputation_status=not_applicable`을 반환한다. `QR_NON_URL` 경고로 URL 검사 대상이 아님을 안내한다.
- HTTP(S)로 시작하나 형식이 잘못된 데이터는 `422 INVALID_URL`이다.
- 디코딩은 됐으나 URL 분석 전체가 실패하면 성공으로 위장하지 않고 해당 5xx 오류를 반환한다.
- `decoded_text`는 HTML이 아닌 이스케이프된 일반 텍스트로 표시한다. 클릭, 자동 실행, 외부 리소스 로딩을 하지 않는다.
- 정상·예외 경로 모두 서버 임시 파일을 정리한다. 남은 파일은 업로드 시각으로부터 10분 이내 정리한다. 클라이언트 요청 취소만으로 서버 삭제 완료를 가정하지 않는다.

## 8. 상태 확인

`GET /api/v1/health`

내부 규칙·필수 QR 디코더 초기화와 PostgreSQL 연결·스키마 준비 확인이 모두 성공하면 HTTP 200을 반환한다.

```json
{
  "request_id": "947dc6aa-9116-42af-b591-de169493105d",
  "data": {
    "status": "ok",
    "api_version": "v1"
  }
}
```

준비되지 않았으면 `503 SERVICE_UNAVAILABLE`을 반환한다. DB 장애로 health가 실패해도 규칙 로딩이 끝난 분석 API는 계속 동작할 수 있다. 이 readiness 결과만으로 프로세스를 반복 재시작하지 않는다. 외부 평판 API 장애는 필수 내부 기능의 준비 상태에 영향을 주지 않는다. 비밀 설정·서버 경로·원문 데이터는 공개하지 않는다.

## 9. 오류와 부분 결과

### 9.1 공통 오류 구조

```json
{
  "request_id": "5b462ac3-86b1-4d26-80b2-cf26a2ce223d",
  "error": {
    "code": "INVALID_URL",
    "message": "HTTP 또는 HTTPS 형식의 올바른 URL을 입력하세요.",
    "field": "url",
    "retryable": false
  }
}
```

`field`는 `url`, `message`, `file`, `answers`, `quiz_version`, `id`, `page`, `page_size`, `category` 또는 `null`이다. `retryable=true`는 같은 입력으로 나중에 다시 요청할 수 있다는 의미다. FE 자동 재시도를 지시하지 않는다.

오류 목록:

- 400 `INVALID_JSON`: JSON 문법 오류. 재시도 불가.
- 404 `CONTENT_NOT_FOUND`: 없는/비공개 사례 또는 현재 문제 세트 없음. 재시도 불가.
- 409 `QUIZ_VERSION_UNAVAILABLE`: 채점할 버전이 폐기됨. 새 문제 조회 필요.
- 413 `PAYLOAD_TOO_LARGE`: 파일 또는 요청 크기 초과. 재시도 불가.
- 415 `UNSUPPORTED_MEDIA_TYPE`: 요청 Content-Type 또는 지원하지 않는 실제 이미지 형식. 재시도 불가.
- 422 `VALIDATION_ERROR`: 누락·빈 값·타입·길이·추가 필드·복수 파일 오류. 재시도 불가.
- 422 `INVALID_URL`: 지원하지 않는 scheme, 잘못된 URL, 사용자정보 포함 등. 재시도 불가.
- 422 `INVALID_IMAGE`: 손상 파일, 형식 위장, 디코딩 실패. 재시도 불가.
- 422 `IMAGE_DIMENSIONS_EXCEEDED`: 가로 또는 세로 제한 초과. 재시도 불가.
- 422 `QR_NOT_FOUND`: 인식 가능한 QR 없음. 재시도 불가.
- 422 `MULTIPLE_QR_CODES`: 복수 QR 발견. 재시도 불가.
- 422 `QR_CONTENT_UNSUPPORTED`: UTF-8 텍스트로 해석할 수 없음. 재시도 불가.
- 429 `RATE_LIMITED`: 요청 제한. 재시도 가능, `Retry-After` 정수 초 헤더 포함.
- 500 `INTERNAL_ERROR`: 예기치 않은 분석 실패. 재시도 가능.
- 503 `SERVICE_UNAVAILABLE`: 필수 분석 모듈 준비되지 않음. 재시도 가능.
- 504 `ANALYSIS_TIMEOUT`: 전체 분석 제한 시간 초과. 재시도 가능.

FastAPI 기본 검증 오류 역시 공통 구조로 변환한다. 프록시에서 발생하는 비JSON 응답과 네트워크 실패도 가능하므로 FE는 JSON 파싱 실패 시 일반 오류 안내를 제공한다.

### 9.2 성공 응답의 경고 코드

- `REPUTATION_UNAVAILABLE`: 활성화된 외부 조회가 실패함. `partial`.
- `URL_ANALYSIS_FAILED`: 문자 내 URL 일부 실패. `partial`.
- `URL_LIMIT_EXCEEDED`: 5개 초과 URL은 생략됨. `partial`.
- `QR_NON_URL`: URL이 아닌 QR. `not_applicable`.

경고는 `{ "code": "REPUTATION_UNAVAILABLE", "message": "외부 평판 조회에 실패하여 내부 분석 결과만 제공합니다." }` 형태다.

외부 조회가 비활성화되어 시도하지 않은 경우 실패 경고를 생성하지 않는다. 외부 조회 전부 성공은 `completed`, 일부 성공·일부 실패는 `partial`, 전부 실패는 `failed`로 `reputation_status`에 표현한다. 조회할 URL이 없으면 `not_applicable`이다.

## 10. 처리 시간과 FE 동작

다음은 배포 환경 검증 전 초기 설정안이다.

- BE 전체 분석 제한: 요청 본문 수신 완료 후 15초.
- 외부 평판 호출: 개별 3초, 전체 분석의 남은 시간 내에서만 수행, 자동 재시도 없음.
- FE 전체 요청 제한: 업로드 포함 30초. 초과 시 중단 안내와 수동 재시도 제공.
- 분석 API 합산 요청 제한: 클라이언트 IP별 고정 60초 구간에서 20회. 공유망 사용자 영향을 배포 테스트에서 확인한다.
- 프록시 IP 헤더는 신뢰한 프록시에서 온 값만 사용한다.
- FE는 중복 제출을 막고 실패 시 입력을 유지한다. 모든 POST 요청은 자동으로 재시도하지 않는다.
- 전체 제한 시간이 초과되어 반환 가능한 일관된 결과가 없으면 504. 부분 결과는 완료된 분석이 있을 때만 반환한다.
- `partial` 결과를 표시할 때는 점수보다 먼저 분석하지 못한 범위를 안내한다.
- QR `not_applicable`에서는 점수 카드를 숨기고 추출 내용과 검사 범위 안내를 표시한다.

## 11. FE 타입 계약

다음 타입은 필드와 분기를 설명하는 FE 구현 기준이다. 실제 BE 모델 및 생성 OpenAPI와 계약 테스트로 비교한다.

```typescript
type Level = 'safe' | 'caution' | 'danger';
type Warning = { code: string; message: string };
type Risk = {
  score: number;
  base_level: Level;
  level: Level;
  overridden: boolean;
  critical_rule_ids: string[];
  summary: string;
  findings: {
    rule_id: string;
    severity: 'info' | 'warning' | 'critical';
    description: string;
  }[];
  recommendations: { code: string; message: string }[];
};
type UrlDetails = {
  normalized_url: string;
  hostname: string;
  scheme: 'http' | 'https';
};
type MessageDetails = {
  character_count: number;
  url_candidate_count: number;
  analyzed_url_count: number;
  omitted_url_count: number;
  matched_spans: { start: number; end: number; rule_id: string }[];
  url_results: {
    index: number;
    url: string;
    status: 'completed' | 'partial' | 'failed';
    score: number | null;
    level: Level | null;
    error_code: string | null;
  }[];
};
type QrDetails = {
  content_type: 'url' | 'text';
  decoded_text: string;
  normalized_url: string | null;
  image_width: number;
  image_height: number;
};
type Meta = {
  analyzed_at: string;
  analysis_version: string;
  reputation_status:
    | 'disabled' | 'completed' | 'partial' | 'failed' | 'not_applicable';
  warnings: Warning[];
};
type Evaluated = {
  status: 'completed' | 'partial';
  risk: Risk;
};
type AnalysisData = Meta & (
  | (Evaluated & { type: 'url'; details: UrlDetails })
  | (Evaluated & { type: 'message'; details: MessageDetails })
  | (Evaluated & {
      type: 'qr';
      details: QrDetails & { content_type: 'url'; normalized_url: string };
    })
  | {
      type: 'qr'; status: 'not_applicable'; risk: null;
      details: QrDetails & { content_type: 'text'; normalized_url: null };
    }
);
type AnalysisResponse = { request_id: string; data: AnalysisData };
type ErrorResponse = {
  request_id: string;
  error: {
    code: string;
    message: string;
    field: 'url' | 'message' | 'file' | 'answers' | 'quiz_version' | 'id'
      | 'page' | 'page_size' | 'category' | null;
    retryable: boolean;
  };
};
```

## 12. DB 기반 교육·퀴즈 API

모든 응답은 request_id와 data 또는 error의 공통 envelope를 따른다. 아래 JSON은 data 내부 예시다. 조회·채점의 BE 제한은 5초, FE 제한은 10초를 초기값으로 제안한다. DB 연결·쿼리 실패는 503 SERVICE_UNAVAILABLE로 처리한다.

### 12.1 교육 목록

GET /api/v1/education/cases

쿼리: page(기본 1, 1 이상), page_size(기본 10, 1~50), category(선택). category enum은 DB 설계의 7개 코드와 같다. 잘못된 쿼리는 422다. 공개 행만 sort_order, id 순으로 제공한다. 범위를 벗어난 페이지는 빈 items다.

```json
{
  "items": [
    {
      "id": "c9025880-06a2-4b27-8f51-001d0e470aff",
      "category": "delivery",
      "title": "택배 사칭 사례",
      "summary": "배송 안내를 가장한 링크를 확인하는 방법"
    }
  ],
  "page": 1,
  "page_size": 10,
  "total": 1
}
```

### 12.2 교육 상세

GET /api/v1/education/cases/{id}

id는 UUID다. 문법 오류 422, 없는/비공개 사례 404. 목록의 네 필드에 example_message(string), suspicious_points(string[]), damage_process(string[]), prevention(string[]), response_steps(string[])를 추가해 반환한다. 예방수칙은 이 상세 응답에 포함한다.

### 12.3 현재 퀴즈

GET /api/v1/quizzes/current

```json
{
  "quiz_version": "quiz-v1",
  "title": "스미싱 예방 퀴즈",
  "questions": [
    {
      "id": "d397e264-8956-4981-94fd-f7dfb3028326",
      "position": 1,
      "kind": "ox",
      "prompt": "의심 링크는 공식 경로로 먼저 확인해야 한다.",
      "options": [
        { "id": "7304b273-46c4-4b35-9e93-34d08c3536b6", "label": "O" },
        { "id": "006bfdb5-3660-4432-8bdb-ff0b25c2be07", "label": "X" }
      ]
    }
  ]
}
```

위 예시는 한 문제만 발췌했다. 실제 questions는 position 순서로 정확히 10개며 options도 position 순이다. kind는 ox 또는 multiple_choice다. 정답·해설은 포함하지 않는다. 현재 공개 세트가 없으면 404 CONTENT_NOT_FOUND다.

### 12.4 퀴즈 채점

POST /api/v1/quizzes/grade — application/json

요청 형식(답안 1개 발췌):

```json
{
  "quiz_version": "quiz-v1",
  "answers": [
    {
      "question_id": "d397e264-8956-4981-94fd-f7dfb3028326",
      "option_id": "7304b273-46c4-4b35-9e93-34d08c3536b6"
    }
  ]
}
```

실제 요청은 해당 버전의 10문제마다 답안 하나씩 정확히 포함한다. 누락·중복·다른 세트의 문제·해당 문제에 속하지 않는 선택지는 422 VALIDATION_ERROR다. quiz_version은 1~64자다. 없는/폐기된 버전은 409 QUIZ_VERSION_UNAVAILABLE다. 이전 공개 버전은 보존된 동안 동일하게 채점한다.

응답 형식(결과 1개 발췌):

```json
{
  "quiz_version": "quiz-v1",
  "total_count": 10,
  "correct_count": 10,
  "score": 100,
  "results": [
    {
      "question_id": "d397e264-8956-4981-94fd-f7dfb3028326",
      "selected_option_id": "7304b273-46c4-4b35-9e93-34d08c3536b6",
      "correct_option_id": "7304b273-46c4-4b35-9e93-34d08c3536b6",
      "is_correct": true,
      "explanation": "문자 속 링크 대신 공식 경로로 확인합니다.",
      "related_case_id": "c9025880-06a2-4b27-8f51-001d0e470aff"
    }
  ]
}
```

실제 results는 문제 순서대로 10개다. 한 문제당 10점이며 related_case_id는 공개 사례가 없으면 null이다. 점수는 BE가 계산한다. 답안·점수는 저장하지 않고 응답에만 사용한다. 동일 버전과 답안의 재제출은 동일 결과를 반환한다.

교육·퀴즈 응답도 no-store를 적용한다. GET 교육·퀴즈와 채점 API를 합해 IP별 60초 고정 구간 60회 제한을 초기안으로 둔다. 429는 기존 공통 규격을 따른다.

추가 계약 테스트: 목록 페이지·유형 필터, 비공개 사례 404, 조회 응답 정답 미포함, 10문제 검증, 중복 답안 거부, 이전 버전 채점, DB 장애 503, 제출 기록 미저장.

## 13. 계약 검증과 후속 결정

FE 목 데이터와 BE 테스트는 동일한 fixture를 사용하도록 후속 작업으로 구성한다. 최소 검증 항목은 다음과 같다.

- URL 정상 결과, 잘못된 scheme, 2,048/2,049자 경계.
- 문자 5,000/5,001 code point 경계와 이모지 오프셋.
- URL 없음·중복·6개 이상·일부 분석 실패.
- 점수 29/30/59/60 및 60점 미만 치명 규칙 상향.
- 이미 danger인 결과의 `overridden=false`.
- QR URL·일반 텍스트·손상 이미지·미인식·복수 QR.
- 파일 10,000,000/10,000,001 bytes와 이미지 4,096/4,097픽셀 경계.
- 평판 비활성화·일부 실패·전체 실패 및 서버 타임아웃.
- 공통 오류 구조, 요청 ID 일치, 로그 원문 제외, 파일 삭제.

팀 검토 항목:

- 신규 제안값: URL 2,048자, 문자 URL 최대 5개, 15/30초 제한, 요청 제한.
- 문자 점수의 최댓값 집계와 규칙별 배점 검증.
- 비URL QR의 `risk=null` 예외 및 DB 콘텐츠·채점 계약의 통합 테스트.
- 배포 서버 주소와 실제 프록시 제한 설정.
- 외부 제공자 확정 전까지 평판 조회 기본 비활성화.

본 문서는 Markdown 계약 초안이다. 실행 가능한 OpenAPI 스키마나 BE 구현 완료를 의미하지 않는다. 구현 시 FastAPI 생성 스키마와 본 계약을 대조하고, 변경은 PM/Docs와 FE·BE가 함께 검토한다.
