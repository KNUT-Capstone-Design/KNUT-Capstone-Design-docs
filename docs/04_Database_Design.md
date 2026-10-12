# 데이터베이스 설계서

- **버전:** Draft v0.2
- **확정 사항:** MVP부터 사용자 OCI 서버에 Docker PostgreSQL 사용
- **설계 상태:** 데이터 모델과 운영 수치는 PM 제안, 팀 검토 필요
- **연계:** [아키텍처](03_System_Architecture.md), [API 명세](05_API_Specification.md)

## 1. 저장 범위

교육 사례·예방수칙·퀴즈 문제·선택지·정답·해설을 저장한다. FE는 BE API로만 접근한다. 탐지 규칙과 배점은 BE 파일로 버전 관리한다.

문자 원문, 검사 URL, QR 이미지 및 추출 내용, 검사 결과, 회원, 퀴즈 답안·점수·제출 이력 테이블은 MVP에서 만들지 않는다. 개인 회원과 기관 데이터 모델은 2차 확장안이며 MVP 마이그레이션에 포함하지 않는다. DB 도입만으로 수집 범위를 늘리지 않는다.

## 2. 논리 모델

- 교육 사례 하나는 제목, 유형, 요약, 예시 메시지, 의심 지점, 피해 과정, 예방수칙과 대응 방법을 갖는다.
- 퀴즈 세트 한 버전은 정확히 10문제를 갖는다.
- 문제 하나는 여러 선택지와 정확히 한 정답을 갖는다. OX는 선택지 두 개다.
- 문제는 추천 교육 사례 하나를 선택적으로 참조한다.
- 공개한 퀴즈 버전은 내용을 수정하지 않고 새 버전을 만든다. 진행 중인 이전 버전도 보존해 채점한다.

## 3. 테이블 초안

모든 시각은 UTC timestamptz, 콘텐츠 식별자는 UUID를 사용한다. 필드는 별도 표시가 없으면 NOT NULL이다.

### education_cases

- id: uuid PK
- category: text, 아래 category 코드 중 하나
- title: varchar(120)
- summary: varchar(300)
- example_message: text, 실제 개인정보를 제거한 교육 예시
- suspicious_points: jsonb, 설명 문자열 배열
- damage_process: jsonb, 단계 설명 문자열 배열
- prevention: jsonb, 예방수칙 문자열 배열
- response_steps: jsonb, 피해 대응 문자열 배열
- published: boolean, 기본 false
- sort_order: integer, 기본 0
- created_at, updated_at: timestamptz

category는 delivery, invitation_obituary, public_agency, finance, investment, impersonation, apk_install 중 하나다. JSON 배열은 문자열 원소와 빈 배열 허용 여부를 seed 검증에서 검사한다. 교육 내용은 일반 텍스트로 제공하고 실행 가능한 HTML을 저장하지 않는다.

인덱스는 공개 목록용 (published, sort_order, id), 유형 필터용 (published, category, sort_order, id)를 제안한다.

### quiz_sets

- version: varchar(64) PK. 예: quiz-v1
- title: varchar(120)
- published: boolean, 기본 false
- is_current: boolean, 기본 false
- created_at: timestamptz
- published_at: timestamptz, nullable

is_current=true인 행은 최대 하나만 존재하도록 부분 unique index를 둔다. 현재 세트는 반드시 공개 상태여야 한다. 최초 seed는 현재 공개 세트를 하나 생성해야 한다.

### quiz_questions

- id: uuid PK
- quiz_version: varchar(64) FK → quiz_sets.version
- position: integer, 1~10
- kind: text, multiple_choice 또는 ox
- prompt: text
- explanation: text
- related_case_id: uuid FK → education_cases.id, nullable

UNIQUE(quiz_version, position). 공개 시 세트에 정확히 10문제가 있는지 트랜잭션 안에서 검증한다. 세트와 문제의 삭제는 FK RESTRICT로 보호한다.

### quiz_options

- id: uuid PK
- question_id: uuid FK → quiz_questions.id
- position: integer, 1 이상
- label: text
- is_correct: boolean

UNIQUE(question_id, position). question_id별 is_correct=true 행에 부분 unique index로 정답을 최대 하나로 제한한다. 정답이 최소 하나인지, OX는 2개·객관식은 2~4개 선택지인지 공개 검증에서 확인한다.

정답은 BE 채점에만 사용하며 문제 조회 응답에서 is_correct와 explanation을 제외한다.

## 4. API와 데이터 연결

- 사례 목록: 공개 행만 페이지 단위 조회, sort_order와 id 순서.
- 사례 상세: 공개된 id 한 개 조회, 없거나 비공개면 404.
- 현재 퀴즈: 현재 공개 세트의 문제·선택지 조회, 정답·해설 제외.
- 채점: 제출 버전의 10문제와 답안을 검증한 뒤 정답 및 해설 반환. 답안을 DB에 INSERT하지 않는다.
- 이전 공개 버전은 MVP 운영 기간 동안 보존한다. 폐기한 버전의 채점은 409로 새 퀴즈를 안내한다.
- 추천 사례가 비공개이면 채점 응답의 related_case_id는 null로 처리한다.

## 5. 스키마 변경과 초기 데이터

BE 저장소에서 마이그레이션과 seed를 버전 관리한다. Python/FastAPI를 유지하면 SQLAlchemy와 Alembic 사용을 제안하되 도구 버전은 BE 초기화 시 확정한다.

1. 빈 개발 DB에 마이그레이션을 적용한다.
2. 공개 사례 최소 7개와 퀴즈 10문제를 넣는다.
3. 관계·정답·문항 수를 검증한 뒤 공개한다.
4. 같은 seed 재실행으로 중복 행이 생성되지 않게 고정 식별자를 사용한다.
5. 운영의 기존 공개 퀴즈 버전을 seed가 덮어쓰지 않게 한다.
6. API와 새 스키마의 호환성을 확인하고 배포한다.

앱 시작마다 자동 스키마 변경을 실행하지 않는다. 운영 마이그레이션은 배포 작업에서 한 번 수행하고 실패 시 배포를 중단한다.

## 6. OCI와 Docker 운영안

- 사용자의 기존 OCI 서버에 PostgreSQL 컨테이너를 별도로 구성한다.
- BE와 같은 Compose 네트워크를 기본안으로 하며 FE에는 DB 자격증명을 전달하지 않는다.
- DB 포트는 외부에 공개하지 않는다. 관리 접속 방식은 SSH 터널 등으로 별도 정한다.
- PostgreSQL 공식 이미지의 구체적인 버전을 고정한다. 메이저 버전과 이미지별 데이터 볼륨 경로는 실제 설치 시 확인한다.
- 영속 볼륨을 사용한다. 컨테이너 재생성과 데이터 삭제를 구분하며 운영에서 볼륨 삭제 명령을 배포 절차에 포함하지 않는다.
- 앱 계정은 MVP에 필요한 SELECT 권한만, 마이그레이션·seed는 별도 계정을 사용한다.
- 비밀번호·접속 문자열은 저장소에 커밋하지 않는다.
- 메모리·디스크·연결 풀 크기는 OCI 인스턴스 자원을 확인해 결정한다.
- DB 장애 시 교육·퀴즈 API는 503과 재시도를 제공한다. 분석 API는 파일 기반 규칙으로 동작 가능하다.

## 7. 백업과 복구

초기 운영 목표는 하루 1회 논리 백업, 최근 7일 보관이다. 이는 제안값이며 데이터 중요도와 저장 공간을 확인해 조정한다.

- 마이그레이션 전에도 백업한다.
- 백업은 컨테이너 및 동일 데이터 볼륨 밖에 보관하고 서버와 분리된 위치에도 복사한다.
- 백업 접근 권한을 제한하고 자격증명이 로그에 노출되지 않도록 한다.
- 최초 배포 전 별도 DB로 실제 복원하여 테이블 수, 콘텐츠 수, 퀴즈 채점을 확인한다.
- 백업 실패·디스크 부족·DB 연결 실패를 운영 점검 항목으로 둔다.
- 메이저 업그레이드는 컨테이너 태그만 바꾸지 않고 검증된 이전 절차와 복원 계획을 마련한다.

## 8. 개발 착수 및 배포 전 확인

- 사용자 확정: PostgreSQL / OCI / Docker.
- 팀 검토: 테이블 상세, ORM·마이그레이션 도구, 백업 보관 기준.
- 서버 확인: 아키텍처·RAM·디스크·기존 컨테이너·포트 충돌·백업 위치.
- 검증: 빈 DB 설치, seed 재실행, DB 재시작 후 데이터 유지, 이전 퀴즈 채점, 공개/비공개 접근, DB 장애 처리, 백업 복원.

## 9. 2차 회원·기관 데이터 모델

기관용 교육 SaaS를 개발할 때 아래 테이블을 별도 마이그레이션으로 추가한다. 필드와 관계는 인증 방식과 기관 운영 정책을 확정한 뒤 상세화한다.

- `users`: 개인 계정, 인증 식별자, 계정 상태, 생성·탈퇴 시각
- `organizations`: 기관 정보, 서비스 상태, 생성 시각
- `organization_members`: 기관·사용자 연결, 학습자·관리자 역할, 초대·활성 상태
- `courses`: 공개 또는 기관 전용 교육 과정
- `course_items`: 과정에 포함되는 사례와 퀴즈의 순서
- `assignments`: 기관이 구성원 또는 그룹에 배정한 과정과 마감일
- `learning_progress`: 사용자별 교육 진행률, 퀴즈 결과 요약, 이수 시각
- `saved_result_summaries`: 사용자가 명시적으로 선택해 저장한 검사 유형·등급·시각·비식별 요약

2차 설계에서도 문자 원문, 전체 URL, QR 이미지와 추출 원문은 저장하지 않는다. 모든 기관 소유 데이터에는 `organization_id`를 두고 API에서 기관 소속과 역할을 함께 검증한다. 통계는 최소 집계 인원 기준을 적용해 개인을 추정할 수 있는 결과를 숨긴다. 계정 탈퇴, 기관 계약 종료, 학습 기록의 보관 기간과 삭제 절차는 구현 전에 별도 정책으로 확정한다.

본 문서는 설계안이며 OCI 설치 또는 운영 검증을 수행했다는 의미는 아니다.
