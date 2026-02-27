# 학급 학생 기초조사 자동화 시스템

GitHub Pages 폼으로 학생 기초자료를 수집하여 n8n을 통해 자동으로 Notion DB에 저장하는 시스템입니다.

---

## 시스템 구조

```
GitHub Pages (폼) → n8n Webhook → Data Table → Code 노드 → HTTP Request → Notion DB
```

---

## 구성 요소

### 1. GitHub Pages (프론트엔드)
- 단계별 모바일 최적화 설문 폼
- Step 1: 기본 정보 (번호, 이름, 성별, 생년월일, 주소, 전화번호)
- Step 2: 신학기 기초자료 (희망진로, 걱정, 친구관계, 건강, 가정전달사항 등)
- 제출 시 n8n Webhook으로 POST 요청

### 2. n8n 워크플로우
총 4개 노드로 구성됩니다.

#### ① Webhook 노드
- Method: POST
- Path: `class-survey`
- Production URL: `https://n8n.도메인/webhook/class-survey`

#### ② Data Table 노드 (Insert)
- 테이블명: `class-survey`
- 모든 폼 데이터를 저장 (백업 용도)
- Map Each Column Manually 방식으로 매핑
- 컬럼 매핑 예시:
  - `num` → `{{ $json.body['번호'] }}`
  - `name` → `{{ $json.body['이름'] }}`
  - `birthday` → `{{ $json.body['생년월일'] }}`
  - `gender` → `{{ $json.body['성별'] }}`
  - `address` → `{{ $json.body['주소'] }}`
  - `parent1` → `{{ $json.body['보호자1휴대폰'] }}`
  - `parent2` → `{{ $json.body['보호자2휴대폰'] }}`
  - `myphone` → `{{ $json.body['학생휴대폰'] }}`

#### ③ Code 노드 (JavaScript)
Data Table 출력 데이터를 Notion API 형식으로 변환합니다.

```javascript
const data = $input.first().json;

// 생년월일 변환: 19840923 → 1984-09-23
const bday = data.birthday ? String(data.birthday) : null;
const formattedBday = bday ? `${bday.slice(0,4)}-${bday.slice(4,6)}-${bday.slice(6,8)}` : null;

const body = {
  parent: { database_id: "여기에_DB_ID" },
  template: {
    type: "template_id",
    template_id: "여기에_템플릿_ID"
  },
  properties: {
    "학생명": { title: [{ text: { content: data.name || "" } }] },
    "번호": { number: Number(data.num) },
    "성별": { select: { name: data.gender } },
    "생일": { date: { start: formattedBday } },
    "주소": { rich_text: [{ text: { content: data.address || "" } }] },
    "전화(부)": { phone_number: data.parent1 || null },
    "전화(모)": { phone_number: data.parent2 || null },
    "전화(학생)": { phone_number: data.myphone || null }
  }
};

return [{ json: body }];
```

> **Mode**: Run Once for All Items  
> **Language**: JavaScript

#### ④ HTTP Request 노드
Notion API를 직접 호출하여 페이지 생성 + 템플릿 적용을 합니다.

- Method: `POST`
- URL: `https://api.notion.com/v1/pages`
- Authentication: Predefined Credential Type → Notion API
- Headers:
  - `Notion-Version`: `2025-09-03`
- Body Content Type: `JSON`
- Specify Body: `Using JSON`
- JSON: `{{ $json }}`

---

## Notion 설정

### DB 스키마 (주요 속성)
| 속성명 | 타입 |
|--------|------|
| 학생명 | Title |
| 번호 | Number |
| 성별 | Select (남자/여자) |
| 생일 | Date |
| 주소 | Text |
| 전화(부) | Phone Number |
| 전화(모) | Phone Number |
| 전화(학생) | Phone Number |

### Integration 설정
1. [Notion Integrations](https://www.notion.so/my-integrations)에서 Integration 생성
2. 학생 DB 페이지 → Connections → Integration 연결
3. n8n Notion API 크레덴셜에 Integration Secret 등록

### 템플릿 설정
- Notion DB에서 템플릿 페이지 생성 (출결 DB, 학생상담, 학부모상담 인라인 DB 포함)
- 템플릿 ID는 URL에서 추출하거나 Notion API로 조회

```
GET https://api.notion.com/v1/data_sources/{data_source_id}/templates
```

---

## ID 찾는 방법

### Database ID
Notion DB URL에서 추출:
```
https://notion.so/workspace/제목-{database_id}?v=...
```

### Template ID
템플릿 페이지 URL에서 추출:
```
https://notion.so/{template_id}
```

---

## 주의사항

- Notion API 버전 `2025-09-03` 이상 필요 (템플릿 적용 기능)
- `phone_number` 속성은 빈 문자열(`""`) 불가 → `null` 처리 필수
- 생년월일은 8자리 숫자(`YYYYMMDD`)로 받아서 Code 노드에서 `YYYY-MM-DD`로 변환
- n8n Data Tables 기능은 `N8N_DATA_TABLES_ENABLED=true` 환경변수 필요 (v1.113+)
- 성별 Select 옵션은 Notion DB와 폼 값이 정확히 일치해야 함 (예: `남자`, `여자`)

---

## 관련 링크

- [Notion API - Creating pages from templates](https://developers.notion.com/guides/data-apis/creating-pages-from-templates)
- [n8n Notion node documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.notion/)
