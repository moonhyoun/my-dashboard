# 데이터 구조 현황

> 지금 Supabase에 무엇이 어떻게 들어 있는지 적어둔 문서입니다.
> `RULES.md` 가 "어떻게 만들어야 하는가"라면 이 문서는 "지금 어떻게 되어 있는가"입니다.
> **구조를 바꾸면 반드시 이 문서도 함께 고칩니다.**
>
> 마지막 갱신: 2026-08-15 (금액 단위를 원으로 전환)

---

## 1. 한눈에 보기

**표 13개 · 뷰 6개**입니다. 금액은 모두 **원 단위**이고, 시간은 `year`·`month` 숫자 칼럼입니다.

| 영역 | 표 | 뷰 |
|---|---|---|
| 영업 | `sales_monthly` `sales_pipeline` `sales_rank` | `sales_pipeline_v` |
| 마케팅 | `marketing_channel` `marketing_budget` | — |
| 손익 | `pl_monthly` `pl_cost` `pl_margin` | `pl_monthly_v` `pl_cost_v` |
| 고객 | `customer_monthly` `customer_cohort` | `customer_cohort_v` `customer_ltv_v` |
| 인사 | `hr_monthly` `hr_team` `hr_tenure` `hr_overtime` | `hr_tenure_v` `hr_overtime_v` |
| 현금 | `cash_monthly` `receivable` `receivable_detail` `debt_monthly` | — |
| 경영 | `exec_target` | — |

모든 표에 `id bigint primary key generated always as identity` 가 있습니다. 입력할 때는 넣지 않습니다.

---

## 2. 뷰가 하는 일 — 여기가 이 구조의 핵심입니다

같은 숫자를 두 곳에 저장하지 않기 위해 뷰를 씁니다. **대시보드는 표가 아니라 뷰를 읽습니다.**

| 뷰 | 무엇을 합치는가 |
|---|---|
| `pl_monthly_v` | `pl_monthly`(목표) + `sales_monthly.revenue`(매출) |
| `pl_cost_v` | `pl_cost`(비용) + `marketing_channel` 광고비 합계를 '광고선전비' 행으로 |
| `sales_pipeline_v` | `sales_pipeline`(견적·협상·계약) + `marketing_channel` 전환 수를 '문의' 행으로 |
| `customer_ltv_v` | `customer_monthly` + `pl_monthly_v` + `pl_cost_v` → ARPU·공헌이익률·이탈률·LTV 계산 |
| `customer_cohort_v` | `customer_cohort` + 두 날짜를 빼서 `m_index` 계산 |
| `hr_tenure_v` | `hr_tenure` + 글자를 보고 `sort_order` 계산 |
| `hr_overtime_v` | `hr_overtime` + 실제 시간을 보고 `bucket` 계산 |

**따라서 이런 값은 저장하지 않습니다.**
손익의 매출과 광고선전비 · 인사의 인건비 · 영업 문의 단계의 광고·직접 문의 ·
LTV · `sort_order` · `m_index` · 초과근무 `bucket`

---

## 3. 표별 구조

### 영업

**`sales_monthly`** — 월 매출과 목표
`year` `month` `revenue`(원, 미래 달은 null) `target`(원) `deals`(계약 건수)
`unique (year, month)`

**`sales_pipeline`** — 유입 경로별 단계. 문의 단계는 저장하지 않음
`year` `month` `source` `stage` `count` `amount`(원) `won_amount`(계약 단계만)
`source in ('광고','직접 문의','소개·추천','영업 발굴')` — 실제 저장은 뒤 둘만
`stage in ('문의','견적','협상','계약')`
`unique (year, month, source, stage)`

**`sales_rank`** — 담당자·품목 순위
`year` `month` `kind` `name` `amount`(원)
`kind in ('person','product')`

### 마케팅

**`marketing_channel`** — 채널별 월 성과
`year` `month` `channel` `paid`(boolean) `spend`(원) `impressions` `clicks` `conversions` `revenue`(원)
`unique (year, month, channel)`

**`marketing_budget`** — 월 예산과 목표
`year` `month` `budget`(원) `target_roas`(%) `target_conversions` `margin_rate`(공헌이익률 %)
`unique (year, month)`
※ `ltv` 칼럼은 삭제됨. `customer_ltv_v` 에서 계산합니다.

### 손익

**`pl_monthly`** — 목표만 저장. 매출은 영업에서 옴
`year` `month` `target_profit`(원) `target_margin`(%)
`unique (year, month)`

**`pl_cost`** — 비용. 광고선전비는 저장하지 않음
`year` `month` `category` `item` `cost_type` `amount`(원)
`category in ('매출원가','판관비','기타')` · `cost_type in ('고정','변동')`
`unique (year, month, item)`

**`pl_margin`** — 상품별 공헌이익률
`year` `month` `product` `margin_rate`(%, 음수 가능)
`unique (year, month, product)`
※ `product` 는 `sales_rank` 의 품목명과 같아야 금액이 표시됩니다.

### 고객

**`customer_monthly`**
`year` `month` `new_count` `churn_count` `active_count` `new_revenue`(원)
`unique (year, month)`

**`customer_cohort`** — 가입 시점과 확인 시점을 각각 저장
`cohort_year` `cohort_month` `year` `month` `retained`
`unique (cohort_year, cohort_month, year, month)`

### 인사

**`hr_monthly`** — 인건비는 손익에서 옴
`year` `month` `headcount` `hired` `resigned`
`unique (year, month)`

**`hr_team`** — 금액이 아니라 배분 비중을 저장
`year` `month` `team` `headcount` `pay_weight`(%, 합이 100)
`unique (year, month, team)`

**`hr_tenure`**
`year` `month` `bucket` `count`
`bucket in ('3년 이상','1~3년','6개월~1년','6개월 미만')`
`unique (year, month, bucket)`

**`hr_overtime`** — 구간이 아니라 실제 시간을 저장
`year` `month` `team` `hours` `count`
`unique (year, month, team, hours)`

### 현금

**`cash_monthly`** — `opening + inflow − outflow = closing` 이 맞아야 함
`year` `month` `opening` `inflow` `outflow` `closing` (모두 원)
`unique (year, month)`

**`receivable`** — 미수금 총액
`year` `month` `balance`(원) `overdue`(원)
`unique (year, month)`

**`receivable_detail`** — 거래처별. `days_due` 가 음수면 아직 기한 전
`year` `month` `customer` `amount`(원) `days_due`(일)
`unique (year, month, customer)`

**`debt_monthly`**
`year` `month` `balance` `repayment` `monthly_due` (모두 원)
`unique (year, month)`

### 경영

**`exec_target`** — 신호등 기준. 목표값이 null이면 원천 표에서 가져옴
`sort_order` `area` `metric`(수정 금지) `target`(null 가능) `green` `yellow` `direction`
`direction in ('up','down')` · `check (green >= yellow)`

`metric` 값: `revenue` `roas` `new_customers` `profit` `payroll_share` `runway`

---

## 4. 지금 들어 있는 기간

| 표 | 기간 |
|---|---|
| `sales_monthly` | 2025-01 ~ 2026-12 (2026-09 이후는 목표만) |
| `sales_pipeline` `sales_rank` | 2025-01 ~ 2026-07 |
| `marketing_channel` `marketing_budget` | 2025-01 ~ 2026-08 |
| `pl_monthly` `pl_cost` `pl_margin` | 2025-01 ~ 2026-07 |
| `customer_monthly` | 2025-01 ~ 2026-07 |
| `customer_cohort` | 2026년만 |
| `hr_*` | 2025-01 ~ 2026-07 (근속·초과근무는 2026년만) |
| `cash_monthly` `receivable` | 2025-01 ~ 2026-07 |
| `receivable_detail` `debt_monthly` | 2026년 일부 |

**영역마다 최신 달이 다릅니다.** 마케팅은 진행 중인 달이 있고 회계는 마감된 달까지입니다.
화면은 각 영역의 최신 달을 따로 찾아 표시합니다.

**지금 값은 전부 샘플입니다.** 회사 규모는 매출 월 84,700,000원 · 인원 5명 기준으로 맞춰져 있습니다.

---

## 5. 구조를 다시 뽑는 법

바뀔 때마다 `schema-sql.txt` 를 SQL Editor에서 실행하고, 결과를 이 문서에 반영합니다.

- ① 테이블·뷰의 칼럼 목록
- ② 정해진 값만 받는 칸 (check 제약)
- ③ 중복을 막는 조합 (unique)
- ④ 뷰가 무엇을 계산하는지
- ⑤ 각 표에 데이터가 어느 기간까지 있는지

⑤번만 따로 실행해도 **어느 영역이 밀렸는지** 바로 알 수 있습니다.

---

## 6. 다른 AI에게 넘길 때

| 파일 | 왜 필요한가 |
|---|---|
| `RULES.md` | 지켜야 할 규칙과 원칙 |
| `SCHEMA.md` | 지금 표와 뷰가 어떻게 생겼는지 |
| `pl/index.html` | 화면 패턴의 기본형 |
| 작업할 영역의 `index.html` | 고칠 대상 (새로 만드는 경우엔 불필요) |

### 요청 문구

> 첨부한 `RULES.md` 와 `SCHEMA.md` 를 지켜서 [영역] 대시보드를 [만들어 / 고쳐]줘.
> `pl/index.html` 은 화면 패턴 참고용이고, [영역]/index.html 이 고칠 대상이야.
> 나는 코딩을 모르니 완성된 index.html 전체 코드를 한 번에 주고, 설명은 짧게.
> 파일 올리는 방법도 이름 바꾸기까지 포함해서 단계별로 알려줘.
> SQL이 필요하면 실행 순서까지 함께 알려줘.
