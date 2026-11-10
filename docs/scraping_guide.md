# 상품 데이터 스크래핑 방법 안내

- 작성일: 2026-11-10
- 범위: 에이블리·지그재그·무신사의 데님 목록, 상품 기본 정보, 사이즈별 실측 수집
- 근거: 2026-11-09~10에 수행한 HTTP·브라우저 실험. 운영용 수집기는 아직 구현하지 않았다.
- 관련 문서: [상품 데이터 수집 검증 보고서](scraping_experiment.md)

## 1. 스크래핑 방식 및 추출 정보

| 사이트 | 방식 | 상품 목록 조회 준비 | 상품 ID |
| --- | --- | --- | --- |
| 에이블리 | 일반 HTTP 요청 | 익명 인증 토큰 + 카테고리 링크의 목록 토큰 | `sno` |
| 지그재그 | 일반 HTTP GraphQL 요청 | 카테고리 이동 URL → 초기 화면의 레이아웃·액션 | `product_id` |
| 무신사 | 브라우저로 세션 준비 후 종료, 독립 `APIRequestContext`로 HTTP 요청 | 카테고리 탐색·스크롤 → 세션 상태·요청 헤더 이전 | `goodsNo` |

세 사이트의 데이터를 연결할 때는 **사이트 + 상품 ID**를 상품 식별자로 사용한다. 상품명은 식별자로 쓰지 않는다. 사이즈명만으로도 옵션을 식별하지 않는다. 기장·계절·안감 등에 따라 같은 사이즈명이 반복될 수 있다.

수집 결과에는 다음 정보들을 추출하여 보존한다.

| 구분 | 정보 |
| --- | --- |
| 상품 | 사이트, 상품 ID, 상품명, 판매처·브랜드, 카테고리, 가격, 대표 이미지 URL, 원본 상품 페이지 URL |
| 상태 | 수집 시각, 요청 성공 여부, 판매 상태 |
| 실측 | 사이즈·옵션, 항목명·값, 추출 경로, 단위·측정 기준과 그 근거 |
| 품질 | 누락등의 이상 |

## 2. 에이블리

### 2.1. 준비 및 전체 흐름

일반 HTTP 클라이언트와 JSON·URL 파서가 필요하다. 검증에서는 사용자 쿠키나 브라우저 실행 없이 새 익명 토큰으로 조회했다.

```text
익명 인증 토큰 발급
→ 카테고리 목록에서 데님 링크 선택
→ 링크의 screen_name·next_token으로 상품 목록 조회
→ sno로 기본 정보와 실측 연결
→ 응답 최상위 next_token으로 다음 목록 조회
```

### 2.2. 익명 인증 토큰 발급

```http
GET https://api.a-bly.com/api/v2/anonymous/token/
```

응답의 `token`을 이후 요청 헤더에 넣는다.

```http
X-Anonymous-Token: <발급받은 token>
```

이 토큰과 `next_token`은 역할이 다르다. 인증 토큰은 API 이용을 위한 값이고, 목록 토큰은 조회할 화면·카테고리·위치를 지정한다. 실제 토큰을 코드나 문서에 고정하지 않는다.

웹 코드는 저장된 익명 토큰을 재사용하고 없을 때 발급한다. 매 상품마다 새 토큰을 발급하는 흐름은 사용하지 않는다.

### 2.3. 시작 목록 토큰 확보

```http
GET https://api.a-bly.com/api/v2/overview/category-index-list/
X-Anonymous-Token: <익명 인증 토큰>
```

1. 응답의 카테고리 항목을 순회한다.
2. `name`이 `데님`인 항목에서 `deeplink`를 읽는다.
3. URL 파서로 다음 쿼리 파라미터를 추출한다.

```text
ably://screens/?screen_name=SUB_CATEGORY_DEPARTMENT&next_token=<목록 토큰>
```

카테고리 객체의 `sno`와 링크 내부 카테고리 ID는 다를 수 있다. 관찰한 값은 각각 `174`, `501`이었다. 숫자를 서로 대입하거나 목록 토큰을 직접 조립하지 않고 반환된 링크를 사용한다.

### 2.4. 목록 조회 및 기본 정보 추출

```http
GET https://api.a-bly.com/api/v2/screens/{screen_name}/?next_token=<목록 토큰>
X-Anonymous-Token: <익명 인증 토큰>
```

URL 파서에서 읽은 토큰 값을 HTTP 클라이언트의 쿼리 파라미터 기능으로 전달한다. `%3D` 같은 URL 인코딩이 중복 적용되지 않도록 한다. GET 본문은 사용하지 않았다.

상품 위치: `components[].entity.item_list[].item`

| 정보 | 상품 객체 내부 필드 |
| --- | --- |
| ID·상품명 | `sno`, `name` |
| 대표 이미지 URL | `image` |
| 원본 상품 페이지 URL | `remote_deeplink`에서 쿼리를 제거한 짧은 웹 URL |
| 판매처 | `market_sno`, `market_name` |
| 카테고리 | `category_name` |
| 가격·할인율 | `price`, `discount_rate` |

컴포넌트가 여러 종류이므로 상품 객체인지 확인한 뒤 읽고, `sno`로 중복 제거한다. 실측 데이터가 기본 정보의 상품 ID와 연결되도록 유지한다. 판매 상태 필드는 별도 확인이 필요하며 가격이 있다는 이유만으로 판매 중이라고 판단하지 않는다.

에이블리 상품 이동에는 `remote_deeplink`의 쿼리 파라미터를 제거한 짧은 링크를 사용한다. URL 파서로 scheme·host·상품 경로를 유지하고 query를 비운다. 다른 호스트로 바꾸거나 상품 ID를 수정하지 않는다.

```text
응답 링크: https://m.a-bly.com/goods/76893452?list_id=...&first_page_rendering=...
사용 링크: https://m.a-bly.com/goods/76893452
```

짧은 URL을 사용자 이동용 상품 링크로 저장·표시하고, 응답의 원문 URL은 수집 출처 확인용으로 별도 보관할 수 있다. 검증에서는 `m.a-bly.com`이 `mobile.a-bly.com`으로 이동했다.

카테고리 링크의 `screen_name`·`next_token`, API 요청의 인증값·목록 토큰은 제거하지 않는다.

### 2.5. 상품 ID로 실측 조회

```http
GET https://api.a-bly.com/api/v3/goods/{sno}/information/
X-Anonymous-Token: <익명 인증 토큰>
```

반드시 두 배열을 모두 확인한다.

```text
pieces[].measurement.sizes[]
pieces[].measurement.extracted_sizes[]
```

두 배열에서 확인한 행 구조는 같다.

| 정보 | 사이즈 행 내부 필드 |
| --- | --- |
| 사이즈명 | `name` |
| 실측 항목명 | `measure_values[].spot.name` |
| 실측 값 | `measure_values[].value` |

- `pieces`, `measurement`가 null이거나 배열이 비어 있을 수 있다.
- `sizes`가 비어도 `extracted_sizes`에 값이 있을 수 있다. 반대 경우도 있다.
- 두 배열이 모두 채워지면 출처를 보존하고 사이즈·옵션·항목을 대조한다. 우선순위를 임의로 정해 덮어쓰지 않는다.
- `extracted_sizes`라는 이름만으로 추출 방식이 OCR·AI라고 확정하지 않는다.
- `spot.top`, `spot.left`는 실측 값으로 사용하지 않는다. 
- `fashion_model.size`는 모델 정보이므로 의류 실측에서 제외한다.
- `product_info.options[].choices[]`의 속성을 읽을 때는 `is_chosen: true`만 선택한다.
- 에이블리는 단면, cm 측정을 사용함을 확인했다.

### 2.6. 다음 목록과 종료 처리

응답 최상위 `next_token`을 같은 화면 API의 다음 요청에 전달한다. 중첩 컴포넌트의 같은 이름 필드와 혼동하지 않는다. 페이지 사이에서도 `sno`를 중복 제거한다.

전체 목록의 끝까지 순회한 검증은 아직 없다. 향후 구현에서는 토큰 부재·반복, 신규 상품 ID 없음 등을 각각 기록하고 무한 요청을 막는 상한을 둔다. 이것을 플랫폼의 확정된 종료 규약으로 해석하지 않는다.

## 3. 지그재그

### 3.1. 준비 및 쿼리 정의 확보

일반 HTTP 클라이언트와 JSON·URL 파서가 필요하다. 검증한 요청은 쿠키·인증 토큰 없이 성공했다.

```text
카테고리 탐색 API → 데님팬츠 landing_url
→ GetCnvPage → layout_id·목록 조회 액션
→ GetCnvPageAction → 상품 카드
→ product_id로 GetSizeInfo
→ 응답 PAGINATION 액션으로 다음 목록 조회
```

GraphQL은 엔드포인트만으로 호출할 수 없다. `query`와 `variables`를 JSON으로 전달하고, 쿼리가 참조하는 모든 fragment 정의도 포함해야 한다. `Content-Type: application/json`을 사용하며 `Content-Length`는 클라이언트에 맡긴다.

현재 웹이 사용하는 목록 쿼리 정의를 확인하는 방법:

1. `https://zigzag.kr/categories` 및 카테고리 이동 페이지의 공개 JavaScript 참조를 확인한다. 브라우저에서는 Network의 요청 Payload에서도 해당 전체 쿼리를 확인할 수 있다.
2. `GlobalNavigationPage`, `GetCnvPage`, `GetCnvPageAction` 작업의 전체 쿼리와 의존 fragment를 확보한다.
3. 초기 화면 쿼리·액션 쿼리의 응답 필드가 아래 경로를 포함하는지 확인한다. 공개 코드에서 초기 쿼리는 연결된 canvas 모듈에 있었다.
4. 쿼리 정의는 코드 배포에 따라 갱신하되, 목록 커서·레이아웃 ID 등 실행 시 값은 새 응답에서 받는다.

아래 목록 요청 예시는 본문 구조 설명용이다. `<전체 쿼리와 fragment 정의>`를 실제 정의로 채워야한다. 실측 쿼리는 3.6절에 전체 본문을 제공한다.

### 3.2. 카테고리 이동 URL 확보

```http
POST https://api.zigzag.kr/api/2/graphql/GlobalNavigationPage
Content-Type: application/json
```

```json
{
  "query": "<GlobalNavigationPage 전체 쿼리와 fragment 정의>",
  "variables": {}
}
```

응답에서 다음 항목을 찾는다.

```text
data.global_navigation_page.department_category_list[]
  .related_item_list[].item_list[]
```

`text.text == 데님팬츠`인 항목의 `landing_url`을 읽는다. 확인한 주소는 다음과 같으며, 실제 실행에서는 반환된 주소를 사용한다.

```text
https://zigzag.kr/pages/srp-clp-category?category_id=547&sub_category_id=2796
```

`547`은 팬츠, `2796`은 데님팬츠 선택에 사용됐다. 다른 카테고리까지 같은 값으로 고정하지 않는다.

### 3.3. 초기 화면에서 레이아웃·목록 액션 확보

```http
POST https://api.zigzag.kr/api/2/graphql/GetCnvPage
Content-Type: application/json
```

공개 웹 코드에서 확인한 규칙에 따라 경로의 `srp-clp-category`를 `srp_clp_category`로 바꾸고, 이동 URL의 쿼리를 `DEEPLINK_ENTRY.option`으로 전달한다. 아래 카테고리 값은 예시다.

```json
{
  "query": "<GetCnvPage 전체 쿼리와 fragment 정의>",
  "variables": {
    "input": {
      "page_key": "srp_clp_category",
      "render_param_list": [{
        "type": "DEEPLINK_ENTRY",
        "option": {
          "category_id": "547",
          "sub_category_id": "2796",
          "page_id": "srp_clp_category"
        }
      }]
    }
  }
}
```

1. `data.result.layout_id`를 읽는다.
2. `data.result.module_spec_list[].meta.action_list[]`에서 상품 목록을 불러오는 `MODULE_REFETCH`·`PRELOAD` 액션을 찾는다. 검증한 해당 모듈의 `body.type`은 `SKELETON`이었다.
3. 해당 액션의 `type`, `target`, `option`을 유지한다. `target.module_slot_id`가 목록 영역 식별자다.

관찰한 `layout_id=593`, `module_slot_id=wWrwdWGYBX6e`를 고정하지 않는다. 액션이 여러 개라면 목록 영역과 필터 상태를 확인해 선택한다. 레이아웃 데이터는 검증 당시 HTML `pageProps` 자체에는 없었다.

### 3.4. 상품 목록 요청

```http
POST https://api.zigzag.kr/api/2/graphql/batch/GetCnvPageAction
Content-Type: application/json
```

이 batch 요청은 객체 배열을 본문으로 보낸다.

```json
[
  {
    "operationName": "GetCnvPageAction",
    "query": "<GetCnvPageAction 전체 쿼리와 fragment 정의>",
    "variables": {
      "page_key": "<현재 page_key>",
      "layout_id": "<초기 응답의 layout_id>",
      "input_list": [{
        "action_id": "<새 UUID>",
        "type": "MODULE_REFETCH",
        "target": {"module_slot_id": "<반환된 슬롯 ID>"},
        "option": "<반환된 option 객체를 실제 JSON 객체로 삽입>"
      }]
    }
  }
]
```

예시의 `option` 자리에는 문자열이 아닌 반환된 객체를 넣는다. `target`의 유효 입력 필드는 보존하되 응답 전용 `target.__typename`은 제거한다. 검증 중 이 필드 때문에 HTTP 200 안에 GraphQL 입력 오류가 반환된 적이 있다. `trigger_type`을 입력 액션에 그대로 복사하지 않는다.

응답 상품 블록 위치:

```text
$[].data.result.module_spec_update_list[].module_spec_list[].body.product_card
```

아래 경로는 이 블록 기준이다. 내부에 `product_card`가 한 번 더 있으므로 구분한다.

| 정보 | 블록 내부 필드 |
| --- | --- |
| 상품 ID | `product_card.product_id` |
| 상품명 | `product_card.product.title` |
| 대표 이미지 URL | `product_card.product.image.normal` |
| 원본 상품 페이지 URL | `action_list[]` 중 `type == DEEP_LINK`의 `option.landing_url` |
| 판매처 | `product_card.shop.name`, `product_card.meta.shop.shop_id` |
| 가격 | `product_card.price.final_price` |
| 판매 상태 | `product_card.meta.state.sales_status` |

같은 이동 액션의 `option.server_option.landingUrl`에서도 웹 링크를 확인했다. `zigzag://` 앱 전용 링크와 웹 링크를 구분하며, 상품명·링크는 같은 블록에서 연결한다. 상품 카드가 중복될 수 있으므로 `product_id`로 중복 제거한다.

### 3.5. 다음 목록 요청

1. 결과 모듈 중 `body.type == LOAD_MORE`를 찾는다.
2. `meta.action_list[]`에서 `type == PAGINATION`인 액션을 읽는다.
3. `action_id`는 새 UUID로 만들고, 이 액션의 `type`, `target`, `option`을 3.4절의 `input_list`에 넣는다.
4. 현재 `page_key`, `layout_id`, 전체 액션 쿼리를 유지해 같은 batch API로 보낸다.

`option.server_option`에는 `cursors`, `moduleSlotId`, `renderedModuleOffset`, `renderParams` 등이 있었다. 이 값을 직접 조립하거나 다른 필터 상태의 값과 섞지 않는다. 커서에 방문자 식별값이 포함될 수 있으므로 전체 커서 로그도 피한다.

HTTP 200이어도 `errors`가 있으면 성공으로 처리하지 않는다.

### 3.6. 상품 ID로 실측 조회

```http
POST https://api.zigzag.kr/api/2/graphql/GetSizeInfo
Content-Type: application/json
```

다음은 기존에 검증한 전체 실측 쿼리 본문이다. `catalog_product_id`를 목록의 `product_card.product_id`로 교체한다.

```json
{
  "query": "query GetSizeInfo($catalog_product_id: ID!) { pdp_base_info(catalog_product_id: $catalog_product_id) { catalog_product { trait_list { type } managed_category_list { key } } } pdp_size_info(catalog_product_id: $catalog_product_id) { item_list { image_url value_list description size_category_id is_size_comparable is_edit_request_clicked is_ai_image_zoom_button_visible } has_size_tab_content } }",
  "variables": {"catalog_product_id": "172896101"}
}
```

실측 위치: `data.pdp_size_info.item_list[]`

| 필드 | 처리 |
| --- | --- |
| `value_list` | 2차원 배열. 첫 행은 항목명, 이후 행은 사이즈별 값 |
| `description` | 단위·측정 오차·AI 추출 안내 등 원문 보존 |
| `is_size_comparable` | 플랫폼의 비교 가능 여부 표시를 품질 정보로 보존 |
| `image_url` | 측정 도식 등 실측 관련 이미지. 대표 상품 이미지와 구분 |

숫자도 문자열로 반환된다. 헤더 이름으로 열을 연결하고 고정 열 순서를 가정하지 않는다. `item_list`가 여러 개이면 구분해 읽는다. `has_size_tab_content`가 참이어도 단일 숫자 실측이 있다는 뜻은 아니다.

최근 표본에서 확인한 예외:

- 복수 기장 `102.5/108.5`, `96/100/104`: 기장 옵션과 연결하기 전 하나의 값으로 자르거나 평균내지 않는다.
- 밴딩 `31~`, `30.5~(히든밴딩)`: 단일 고정 허리 치수로 단정하지 않는다.
- `기본`, `롱` 같은 추가 열: 실측 항목과 옵션별 기장을 구분하는 추가 해석이 필요하다.
- 허벅지·밑위 열 자체가 빠진 표: 누락 상태를 남긴다.
- AI 추출 안내와 `is_size_comparable: false`: 숫자가 있어도 검수 전 비교 가능 데이터로 확정하지 않는다.
- `허리단면` 헤더 아래 큰 숫자: 단면·둘레 기준을 확인하고 임의로 2로 나누지 않는다.

## 4. 무신사

### 4.1. 준비 및 전체 흐름

무신사는 **Playwright 브라우저로 세션을 준비한 뒤 독립 `APIRequestContext`로 이전하고, Chrome을 종료한 상태에서 목록·실측 API를 직접 호출한다.** 브라우저는 초기 준비에 사용하며 상품마다 페이지를 열거나 스크롤하지 않는다.

```text
새 브라우저 컨텍스트
→ 데님 카테고리 탐색·스크롤
→ 목록 API URL·JSON·실제 요청 헤더 확보
→ 브라우저 세션 상태를 독립 APIRequestContext에 이전
→ 독립 컨텍스트로 목록·실측 호출 성공 확인
→ Chrome 종료
→ nextPageUrl로 목록 조회, goodsNo로 실측 조회
→ 작업 종료 시 API 컨텍스트와 Playwright 런타임 정리
```

검증 환경은 headless Chrome `156.0.8078.81`, 새 비영구 컨텍스트, locale `ko-KR`, viewport `1280 × 900`이었다. 최초 쿠키는 0개였고 사용자 프로필·제공받은 쿠키를 사용하지 않았다.

페이지 탐색 중 새 쿠키가 생성되는 것은 정상이다. 시작 시 쿠키가 없다는 말은 쿠키 저장을 막는다는 뜻이 아니다. 일반 HTTP 요청 실패 원인을 특정 쿠키 하나로 확정하거나 사용자 쿠키 복사를 필수 절차로 두지 않는다.

Chrome 종료 후에도 Playwright 런타임·드라이버와 독립 API 요청 컨텍스트는 유지한다. 브라우저 없이 최초 세션을 만드는 방식은 검증하지 않았다. 이 구성은 일반 Python `requests`나 Java HTTP 클라이언트로의 완전 이전과도 다르다.

### 4.2. 초기 목록 수신과 세션 이전

페이지 이동·스크롤 **전에** 네트워크 응답 관찰을 등록한다. 응답 URL의 호스트·경로·카테고리를 확인해 목록 요청을 식별한다.

```text
탐색 페이지: https://www.musinsa.com/category/003002/goods?gf=A
목록 API:   https://api.musinsa.com/api2/dp/v2/plp/goods
```

1. 카테고리 페이지로 이동하고 상품 요소가 나타나는지 확인한다.
2. 스크롤해 사이트 코드가 추가 목록을 요청하도록 한다.
3. 해당 API 응답의 HTTP 상태, JSON, `meta.result`를 확인한다.
4. `data.list[]`를 읽고 `goodsNo`로 중복 제거한다.

검증에서는 2페이지 JSON을 수신했다. 첫 페이지 상품은 HTML의 `__NEXT_DATA__`에도 있으므로 XHR 응답만 읽으면 첫 페이지를 놓칠 수 있다. 첫 페이지도 포함하려면 다음 위치를 확인한다.

```text
props.pageProps.dehydratedState.queries[]
  .state.data.pages[].data.list[]
```

이 경로는 `__NEXT_DATA__` 스크립트의 JSON 기준이다. 여러 쿼리 중 카테고리 키와 목록 구조를 확인해 선택한다. 배열 인덱스를 고정하지 않는다.

브라우저를 종료하기 전에 다음 준비를 완료한다.

1. 관찰한 목록 URL 또는 응답의 `nextPageUrl`과, 목록에서 얻은 `goodsNo`의 실측 URL을 준비한다.
2. 같은 페이지의 브라우저 `fetch`로 목록·실측 API가 성공하는지 확인한다. 검증에서는 `credentials: include`, `cache: no-store`, `Accept: application/json`을 사용했다.
3. 실제 성공 요청의 일반 헤더를 관찰해 대상 API별로 메모리에 보관한다. 검증에서는 `Cookie`, `Host`, `Connection`, `Content-Length`와 HTTP pseudo-header를 복사 대상에서 제외했다. 쿠키는 다음 단계의 저장 상태로 전달한다.
4. `browser_context.storage_state()`를 파일 경로 없이 호출해 저장 상태를 메모리에서 얻는다.
5. `playwright.request.new_context(storage_state=state)`로 **독립** `APIRequestContext`를 만든다. 각 요청에는 앞서 관찰한 대상 API의 일반 헤더를 전달한다.
6. 독립 컨텍스트에서 목록·실측 응답이 성공하고 기준 데이터와 일치하는지 확인한 뒤 `browser.close()`로 Chrome을 종료한다.
7. 독립 API 컨텍스트를 유지해 이후 수집을 진행한다. 이 단계에서 Playwright 런타임을 먼저 종료하지 않는다.

`browser_context.request`는 브라우저 컨텍스트와 쿠키를 공유하는 기능이다. 브라우저 종료 후 수집에 사용할 객체는 이것이 아니라 `playwright.request.new_context(...)`로 별도 생성한 컨텍스트다. 독립 컨텍스트 생성 이후 두 쿠키 저장소가 자동 동기화된다고 가정하지 않는다.

실제 토큰·쿠키·저장 상태·서명이 포함된 전체 URL은 문서나 로그에 남기지 않는다. 위 절차는 준비된 세션의 유효성을 확인한 흐름이며, 어떤 쿠키나 헤더가 필수인지 최소 조건을 분리한 검증은 아니다.

### 4.3. 기본 정보와 다음 목록

아래는 API 응답의 `data.list[]` 내부 필드다.

| 정보 | 필드 |
| --- | --- |
| ID·상품명 | `goodsNo`, `goodsName` |
| 대표 이미지 URL | `thumbnail` |
| 원본 상품 페이지 URL | `goodsLinkUrl` |
| 브랜드 | `brand`, `brandName` |
| 가격 | `price`, `finalPrice` |
| 품절 여부 | `isSoldOut` |

다음 목록 정보는 `data.pagination`의 `page`, `hasNext`, `nextPageUrl`에 있다. Chrome 종료 후에는 **응답의 `nextPageUrl`을 독립 API 컨텍스트의 `get`에 그대로 전달**한다. 페이지 이동이나 추가 스크롤은 필요하지 않았다.

종료 후 3페이지의 상품 60개를 재조회하고, 그 응답의 다음 URL로 4페이지의 새로운 상품 60개도 받았다. 쿠키·관찰 헤더를 옮긴 일반 Python `requests`는 목록·실측 모두 403이었으므로, 독립 Playwright HTTP 요청의 성공을 모든 HTTP 클라이언트에 일반화하지 않는다.

요청에는 `page`, `seen`, `hmacId` 등이 포함될 수 있다. 서명·쿠키를 직접 만들거나 고정하지 않는다. 후속 구현에서는 `hasNext`, 새 응답 유무, 신규 ID, 제한 시간·수집 상한을 함께 다룬다.

### 4.4. 상품 ID로 실측 API 직접 호출

1. 목록의 `goodsNo`와 `goodsLinkUrl`을 함께 보관한다.
2. `goodsNo`로 다음 실측 URL을 구성하고 준비된 독립 API 컨텍스트의 `get`으로 호출한다. 상품 페이지에 이동하거나 상품별 스크롤을 하지 않는다.

```text
GET https://goods-detail.musinsa.com/api2/goods/{goodsNo}/actual-size
```

3. 요청의 상품 ID와 실측을 연결할 목록 상품 ID가 같은지 확인한다. 다른 상품의 응답을 재사용하지 않는다.
4. HTTP 200, `meta.result == SUCCESS`, `data`의 존재와 실측 행을 별도로 검사한다.

실측 경로:

```text
data.sizes[].name                 사이즈명
data.sizes[].items[].name         항목명
data.sizes[].items[].value        값
```

`sequence`는 표시 순서이며 `recommendSizeRange`는 실측 값이 아니다. `mySize`도 상품 실측과 구분한다. `mobileImage`, `webImage`는 측정 도식이므로 목록의 `thumbnail`을 대체하지 않는다.

### 4.5. 세션 수명과 실패 처리

API 수집 단계에서는 네트워크 응답 이벤트나 스크롤을 기다리는 대신 독립 컨텍스트의 HTTP 요청 완료와 JSON 검증을 기준으로 처리한다. 각 요청에는 타임아웃과 작업별 요청 횟수·시간 상한을 둔다. 초기 브라우저 준비의 스크롤 횟수·고정 대기는 성능 목표가 아니다.

| 상태 | 기록·처리 방향 |
| --- | --- |
| 초기 카테고리 탐색 실패 | 세션 준비 실패로 기록 |
| 초기 페이지는 열렸지만 목록 응답 없음 | 준비 단계의 응답 미수신으로 기록 |
| 독립 API 요청 타임아웃 | 요청 실패로 기록. 실측 미제공으로 확정하지 않음 |
| 401·403 또는 차단 HTML | 세션·접근 실패로 기록. 응답을 파싱해 실측 없음으로 처리하지 않음 |
| API 200, 실측 배열 없음 | 구조화된 실측 미확보로 기록 |
| API 200, 전 항목 0 | 통신은 성공, 비교용 실측은 없음 |

세션이 만료되거나 차단되면 무한 재시도하지 않는다. 유한 재시도·중단·브라우저 세션 재준비의 기준은 추가 설계하고 검증한다. 재준비 시에는 새 저장 상태로 독립 API 컨텍스트를 다시 생성하고 이전 컨텍스트를 정리한다.

검증에서는 Chrome 종료 직후와 약 5초 뒤에도 목록·실측이 성공했고 브라우저 기준 데이터와 일치했다. **장시간 유지·만료 후 복구·프로세스 재시작 후 복원·배포 서버의 성공은 미검증**이다. 목록 직접 순회와 실측 다수 호출의 장기 부하도 아직 확인하지 않았다. 브라우저 준비 비용이 줄어드는 효과는 기대할 수 있지만 속도·메모리 개선량을 측정한 것은 아니다.

자원 정리 순서는 세션 이전·검증 후 Chrome 종료, 수집 완료 후 독립 컨텍스트 `dispose()`, 마지막으로 Playwright 런타임 종료다. 준비나 수집 중 오류가 나도 각 자원을 정리한다. 브라우저 컨텍스트를 종료한 뒤 그 컨텍스트의 `request` 객체를 계속 사용하는 방식으로 구현하지 않는다.

## 5. 실측 정규화와 비교 대상 판정

다음은 향후 구현에 적용할 처리 지침이다. 서비스의 상세 검색·순위 정책을 이미 구현하거나 확정했다는 의미는 아니다.

### 5.1. 항목 이름과 측정 기준

| 공통 항목 | 관찰한 항목명 예 |
| --- | --- |
| 총장 | 총장, 총기장 |
| 허리 | 허리, 허리단면 |
| 힙 | 힙단면, 엉덩이단면 |
| 허벅지 | 허벅지단면 |
| 밑위 | 밑위, 밑위길이 |
| 밑단 | 밑단단면 |

이 표는 이름을 연결하는 출발점이다. 이름을 통일해도 측정 기준까지 같아지는 것은 아니다. 단위, 단면/둘레, 앞/뒤밑위가 불명확하면 미확인 상태를 남기고 자동 환산하지 않는다.

사이즈명·실측 출처·옵션 원문을 유지하고, 총장이 다르다는 이유만으로 같은 사이즈의 값을 덮어쓰지 않는다. 복수 기장을 옵션별 행으로 분리하려면 실제 옵션과의 대응 근거가 필요하다.

### 5.2. 0·누락·복합 값 처리

| 응답 형태 | 비교용 처리 |
| --- | --- |
| 모든 사이즈의 모든 항목이 0 | 상품을 실측 미제공 상태로 분류하고 실측 비교에서 제외 |
| 특정 사이즈만 전 항목 0 | 그 사이즈만 제외. 다른 사이즈까지 버리지 않음 |
| 일부 항목만 0 또는 누락 | 해당 항목을 결측으로 취급하고 검색 필수 항목 충족 여부 판단 |
| null·빈 배열 | 확인한 경로에서 실측 미확보. 상품 설명 전체의 실측 부재로 확대하지 않음 |
| 범위·복수 기장·밴딩 문자열 | 원문 보존, 해석 필요 상태. 임의의 단일 숫자로 변환하지 않음 |
| 양수 숫자 | 형식상 후보 값. 측정 기준·이상값·정확성 검사는 별도로 필요 |

실제 관찰한 예는 무신사 `5367138`의 전 사이즈 0, `5128887`의 XS만 전 항목 0, `7047636`의 힙 0이다. 전부 0인 첫 상품은 실제로 실측 미제공 제품임을 확인했다.

0을 거리 계산에 넣거나 평균값으로 채우지 않는다. 원시 0과 정규화된 결측을 구분해 보존한다. 통신 성공과 실측 품질은 서로 다른 상태로 관리한다. 상품의 기본 정보는 남길 수 있지만 실측 없는 상품을 유효한 핏 추천처럼 표시하지 않는다.

### 5.3. 수집·품질 상태 구분

| 구분 | 예 |
| --- | --- |
| 요청 실패 | 타임아웃, HTTP 403, GraphQL 오류, 실측 응답 미수신 |
| 응답 성공·실측 미확보 | null, 빈 배열, 전 항목 0 |
| 부분 실측 | 일부 항목만 양수, 다른 항목 누락·0 |
| 해석 필요 | 다중 기장, 밴딩, 옵션 대응 불명확, AI 추출·비교 제한 |
| 비교 후보 | 검색에 필요한 치수와 측정 기준을 충족한 사이즈 |

API에 값이 없는 상황은 즉시 반복 요청으로 해결되지 않을 수 있다.

## 6. 구현 전 확인 사항과 유지보수

- 상품 단위 중복 제거와 사이즈·옵션 단위 품질 검사를 함께 적용한다.
- 에이블리·지그재그의 이미지 URL과 상품 링크는 일부 표본에서 HTTP 접근을 확인했다. 실제 서비스 도메인의 브라우저에서 이미지 표시와 URL의 장기 유효성은 별도 확인한다. 무신사 표시용 URL의 별도 접근 검증도 남아 있다.
- 공개 웹 코드·GraphQL fragment·API 스키마 변경을 감지하면 기존 파서의 성공 판정을 중단하고 응답 구조를 재확인한다. 빈 결과를 곧바로 상품 삭제로 해석하지 않는다.
- 외부 서비스 노출 전 수집·저장·이미지 사용·데이터 노출 관련 플랫폼 정책을 확인한다.
