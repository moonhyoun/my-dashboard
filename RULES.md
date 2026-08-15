# 대시보드 프로젝트 규칙

> 새 대시보드를 만들 때 이 문서를 AI에게 먼저 붙여넣으세요.
> 마지막 수정: 2026-08-15 · 영업 대시보드 완성 시점

---

## 1. 이 프로젝트가 뭔지

사내 대시보드 6개를 **웹페이지로** 만들어 링크로 공유합니다.
숫자는 Supabase에서 고치면 웹페이지에 자동 반영됩니다.

```
GitHub (my-dashboard)  →  코드 보관
Supabase               →  숫자 보관
Vercel                 →  웹페이지로 공개 (GitHub에 올리면 1~2분 뒤 자동 반영)
```

## 2. 폴더와 주소

| 파일 위치 | 접속 주소 | 상태 |
|---|---|---|
| `index.html` | `/` | 완료 — 6개 링크 모음 |
| `sales/index.html` | `/sales` | 완료 — 영업 |
| `marketing/index.html` | `/marketing` | 예정 |
| `customer/index.html` | `/customer` | 예정 |
| `pl/index.html` | `/pl` | 예정 (손익) |
| `hr/index.html` | `/hr` | 예정 (인사) |
| `exec/index.html` | `/exec` | 예정 (경영) |

**파일 이름은 항상 `index.html`.** 바뀌는 건 폴더 이름뿐입니다.

## 3. 절대 지켜야 할 것

- **파일 하나로 완성.** HTML·CSS·JS를 한 `index.html`에 다 넣습니다.
- **빌드·설치 과정 없음.** GitHub에 올리면 그걸로 끝이어야 합니다.
- **차트 라이브러리 금지.** 그래프는 순수 SVG 또는 HTML/CSS로 직접 그립니다.
- **`type="module"` 금지.** Supabase는 아래 방식으로만 불러옵니다.

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script>
  const SUPABASE_URL = 'https://abgpkybrwtpzflkchchu.supabase.co';
  const SUPABASE_KEY = 'sb_publishable_4LUZJYwgyDYPb-EzzpDHRw_FwEUvJ1e';

  const { createClient } = supabase;
  const db = createClient(SUPABASE_URL, SUPABASE_KEY, {
    auth: { persistSession: false, autoRefreshToken: false }
  });
</script>
```

## 4. 이미 한 번 데인 것들 — 반복 금지

**변수 이름을 `URL`로 쓰면 안 됩니다.**
`const URL = '...'` 은 브라우저 내장 기능을 덮어써서 Supabase가 주소를 해석하지 못합니다.
증상: `Invalid supabaseUrl: Provided URL is malformed`
→ 반드시 `SUPABASE_URL`, `SUPABASE_KEY`로 씁니다.

**`persistSession: false`를 넣습니다.**
없으면 브라우저가 저장소 접근을 막으면서 콘솔에 경고가 쌓입니다.

**RLS를 안 열면 데이터가 안 옵니다.**
테이블을 만들 때마다 읽기 정책을 함께 만듭니다. 잊으면 화면이 예시 값으로 나옵니다.

**GitHub에서 덮어쓸 땐 해당 폴더 안으로 들어간 뒤 업로드합니다.**
루트에서 올리면 새 파일이 하나 더 생길 뿐 교체되지 않습니다.

**파일을 더블클릭해서 여는 건 테스트용으로 부적합합니다.**
`file:` 주소는 보안 정책이 달라 데이터를 못 불러올 수 있습니다. Vercel 주소로 확인하세요.

## 5. Supabase 규칙

**테이블 이름** — `영역_내용` 형식. 예: `sales_monthly`, `marketing_channel`

**금액 단위는 만원으로 통일.** 화면에서 만원/백만원/원으로 바꿔 보여주되, DB에는 항상 만원으로 저장합니다. 영역이 달라도 이 규칙은 같습니다.

**시간은 `year`, `month` 숫자 칼럼으로.**
`sort_order`로 순번을 매기지 않습니다. 중간에 한 줄 끼워 넣을 때마다 뒤 번호를 전부 고쳐야 하기 때문입니다.
`sort_order`는 **순서에 의미가 있지만 시간이 아닌 것**에만 씁니다. 예: 영업 단계(문의 → 견적 → 협상 → 계약)

**아직 지나지 않은 기간은 비워둡니다(null).** 0을 넣으면 실적이 0인 것과 구분이 안 됩니다.

**새 테이블마다 읽기 권한을 엽니다.**

```sql
alter table 테이블명 enable row level security;
create policy "read" on 테이블명 for select to anon using (true);
```

**키는 공개돼도 안전합니다.** `sb_publishable_...` 키는 읽기 전용이라 코드에 그대로 넣어도 됩니다. 수정은 Supabase에 로그인한 사람만 할 수 있습니다.

## 6. 디자인 규칙

- 흰 배경, **포인트 색은 파랑 하나만** (`#1D4ED8`)
- 강약이 필요하면 다른 색을 추가하지 말고 **파랑의 농도**로 구분합니다
- 폰트: Pretendard CDN
- 카드·패널: 테두리 `#E7EAF0`, 모서리 14px
- 회색 글자: `#64748B`
- 휴대폰에서 안 깨져야 합니다

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.min.css">
```

## 7. 공통으로 넣는 기능

영업 대시보드에서 자리를 잡은 것들입니다. 다른 대시보드에도 같은 방식으로 넣습니다.

- **연도 버튼** — 데이터에 있는 연도를 자동으로 읽어 버튼을 만들고, 기본은 올해
- **단위 버튼** — 만원 / 백만원 / 원
- **왼쪽 위 `← 대시보드 목록` 링크** — `href="/"`
- **데이터를 못 불러오면** 화면을 비우지 말고 주황색 안내줄 + 예시 값 표시
- **개수·기간 토글** — 순위처럼 목록이 길어지는 곳에는 표시 개수와 기간(당월/누적) 선택을 붙임

## 8. 코드에 개수를 박아두지 않기

담당자가 3명이든 20명이든, 단계가 4개든 7개든 **화면이 데이터에 맞춰 늘어나야** 합니다.
Supabase에서 행을 추가하는 것만으로 반영되어야 하고, 그때 코드를 고쳐야 한다면 잘못 만든 것입니다.

## 9. 새 대시보드 만드는 순서

1. 어떤 숫자를 볼지 정한다
2. 테이블을 설계하고 SQL로 만든다 (RLS 포함)
3. `index.html`을 만든다
4. GitHub `+` → Create new file → `폴더명/index.html` → 붙여넣기 → Commit
5. `주소/폴더명` 으로 접속해 확인
6. 루트 `index.html`의 해당 카드를 클릭 가능하게 바꾼다

## 10. 아직 안 정한 것

- 카드 4개 구조를 모든 영역에 쓸지 (마케팅 만들어보고 결정)
- 실제 데이터를 매달 넣는 방식 — 현재는 Supabase Table Editor 직접 입력, CSV 가져오기 병행 검토
- `/exec` 경영 대시보드가 다른 5개 데이터를 어떻게 모아올지

---

## 다른 AI에게 넘길 때

이 문서 + 참고할 기존 `index.html` 파일을 함께 주고 이렇게 요청하세요.

> 첨부한 규칙 문서를 지켜서 [영역] 대시보드를 만들어줘.
> 나는 코딩을 모르니 완성된 index.html 전체 코드를 한 번에 주고, 설명은 짧게.
> 보고 싶은 숫자는 [...] 이고, Supabase 테이블은 [있음 / 없어서 설계부터 필요] 상태야.
