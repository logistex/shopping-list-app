# 쇼핑 리스트 Supabase 연동 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 쇼핑 리스트 앱(`index.html`)의 저장소를 `localStorage`에서 Supabase 데이터베이스(`shopping_items` 테이블)로 교체하고, 실시간 동기화를 붙인다.

**Architecture:** Supabase를 단일 진실 공급원으로 삼는다. 클라이언트는 쓰기 요청만 Supabase에 보내고, 화면 갱신은 Supabase Realtime 구독이 트리거하는 재조회(refetch)로만 이루어진다. 로컬 배열을 직접 조작하는 낙관적 업데이트는 하지 않는다.

**Tech Stack:** 순수 HTML/CSS/JS (빌드 도구 없음), `@supabase/supabase-js` v2 (CDN, ESM), Supabase Postgres + Realtime.

## Global Constraints

- 프로젝트: `shopping-list-app` (`project_id` = `rkkebybdtoqfbplshrvq`, region `ap-northeast-2`)
- 프로젝트 URL: `https://rkkebybdtoqfbplshrvq.supabase.co`
- Publishable key: `sb_publishable_MtwMHuEA_z3xKmi_Fx1bIw_gjOmrXTC`
- 인증 없이 전체 공개 접근 (RLS는 켜두되 정책으로 전체 허용)
- 로그인 기능 추가하지 않음, 기존 localStorage 데이터 마이그레이션하지 않음, 오프라인 폴백 없음 (설계 문서 `docs/superpowers/specs/2026-08-03-supabase-shopping-list-design.md` 참조)

---

### Task 1: `shopping_items` 테이블, RLS, Realtime 설정

**Files:** 없음 (Supabase MCP를 통한 원격 DB 변경. 이 리포지터리는 Supabase CLI 마이그레이션 폴더를 쓰지 않으므로 로컬 파일은 만들지 않는다.)

**Interfaces:**
- Produces: `public.shopping_items(id uuid, text text, checked boolean, created_at timestamptz)` 테이블. RLS 활성화 + select/insert/update/delete 전체 허용 정책 4개. `supabase_realtime` publication에 테이블 등록됨.

- [ ] **Step 1: 마이그레이션 적용**

Supabase MCP `apply_migration` 도구를 다음 인자로 호출한다.
- `project_id`: `rkkebybdtoqfbplshrvq`
- `name`: `create_shopping_items_table`
- `query`:
```sql
create table shopping_items (
  id uuid primary key default gen_random_uuid(),
  text text not null,
  checked boolean not null default false,
  created_at timestamptz not null default now()
);

alter table shopping_items enable row level security;

create policy "public select" on shopping_items for select using (true);
create policy "public insert" on shopping_items for insert with check (true);
create policy "public update" on shopping_items for update using (true) with check (true);
create policy "public delete" on shopping_items for delete using (true);

alter publication supabase_realtime add table shopping_items;
```

- [ ] **Step 2: 테이블/컬럼 확인**

Supabase MCP `list_tables` 도구를 `project_id: rkkebybdtoqfbplshrvq`, `schemas: ["public"]`, `verbose: true`로 호출한다.

Expected: 결과에 `shopping_items` 테이블이 있고, 컬럼이 `id (uuid, pk)`, `text (text, not null)`, `checked (boolean, not null, default false)`, `created_at (timestamptz, not null, default now())` 이어야 한다.

- [ ] **Step 3: RLS 정책 확인**

Supabase MCP `execute_sql` 도구를 다음 쿼리로 호출한다.
```sql
select policyname, cmd from pg_policies where tablename = 'shopping_items' order by policyname;
```

Expected: 4개 행 반환 (`public delete`/DELETE, `public insert`/INSERT, `public select`/SELECT, `public update`/UPDATE).

- [ ] **Step 4: Realtime 등록 확인**

Supabase MCP `execute_sql` 도구를 다음 쿼리로 호출한다.
```sql
select tablename from pg_publication_tables where pubname = 'supabase_realtime' and tablename = 'shopping_items';
```

Expected: 1개 행(`shopping_items`) 반환.

- [ ] **Step 5: 커밋**

이 태스크는 원격 DB 변경만 있고 로컬 파일 변경이 없으므로 커밋할 것이 없다. 다음 태스크로 진행한다.

---

### Task 2: `index.html`을 Supabase 연동으로 교체

**Files:**
- Modify: `index.html:139-252` (body 내 footer 아래 에러 메시지 엘리먼트 추가, `<script>` 블록 전체 교체)

**Interfaces:**
- Consumes: Task 1에서 만든 `shopping_items` 테이블, Global Constraints의 프로젝트 URL/키
- Produces: 없음 (최종 사용자 화면)

- [ ] **Step 1: 에러 메시지 엘리먼트 추가**

`index.html`의 `.footer` div 바로 다음(151번째 줄, `</div>` 뒤)에 아래 엘리먼트를 추가한다.

```html
  <p id="error-message" style="display:none; margin-top:12px; padding:10px 14px; border-radius:10px; background:rgba(255,59,48,0.12); color:var(--danger); font-size:13px; text-align:center;"></p>
```

- [ ] **Step 2: 기존 `<script>` 블록(152번째 줄 `<script>`부터 250번째 줄 `</script>`까지)을 아래 내용으로 완전히 교체**

```html
<script type="module">
  import { createClient } from "https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/+esm";

  const SUPABASE_URL = "https://rkkebybdtoqfbplshrvq.supabase.co";
  const SUPABASE_KEY = "sb_publishable_MtwMHuEA_z3xKmi_Fx1bIw_gjOmrXTC";
  const supabase = createClient(SUPABASE_URL, SUPABASE_KEY);

  let items = [];

  const listEl = document.getElementById("list");
  const formEl = document.getElementById("add-form");
  const inputEl = document.getElementById("item-input");
  const countEl = document.getElementById("count");
  const clearBtn = document.getElementById("clear-checked");
  const errorEl = document.getElementById("error-message");

  function showError(message) {
    errorEl.textContent = message;
    errorEl.style.display = "block";
    setTimeout(() => {
      errorEl.style.display = "none";
    }, 3000);
  }

  function render() {
    listEl.innerHTML = "";

    if (items.length === 0) {
      const empty = document.createElement("li");
      empty.className = "empty";
      empty.textContent = "리스트가 비어 있습니다.";
      listEl.appendChild(empty);
    } else {
      items.forEach((item) => {
        const li = document.createElement("li");
        if (item.checked) li.classList.add("checked");

        const checkbox = document.createElement("input");
        checkbox.type = "checkbox";
        checkbox.checked = item.checked;
        checkbox.addEventListener("change", () => toggleItem(item.id, !item.checked));

        const text = document.createElement("span");
        text.className = "item-text";
        text.textContent = item.text;

        const deleteBtn = document.createElement("button");
        deleteBtn.type = "button";
        deleteBtn.className = "delete-btn";
        deleteBtn.textContent = "✕";
        deleteBtn.addEventListener("click", () => deleteItem(item.id));

        li.appendChild(checkbox);
        li.appendChild(text);
        li.appendChild(deleteBtn);
        listEl.appendChild(li);
      });
    }

    const total = items.length;
    const checked = items.filter((i) => i.checked).length;
    countEl.textContent = total === 0 ? "" : `${checked} / ${total}개 완료`;
  }

  async function loadItems() {
    const { data, error } = await supabase
      .from("shopping_items")
      .select("*")
      .order("created_at", { ascending: true });

    if (error) {
      showError("목록을 불러오지 못했습니다.");
      return;
    }

    items = data;
    render();
  }

  async function addItem(text) {
    const { error } = await supabase.from("shopping_items").insert({ text, checked: false });
    if (error) showError("추가에 실패했습니다.");
  }

  async function toggleItem(id, checked) {
    const { error } = await supabase.from("shopping_items").update({ checked }).eq("id", id);
    if (error) showError("변경에 실패했습니다.");
  }

  async function deleteItem(id) {
    const { error } = await supabase.from("shopping_items").delete().eq("id", id);
    if (error) showError("삭제에 실패했습니다.");
  }

  async function clearChecked() {
    const { error } = await supabase.from("shopping_items").delete().eq("checked", true);
    if (error) showError("삭제에 실패했습니다.");
  }

  formEl.addEventListener("submit", (e) => {
    e.preventDefault();
    const value = inputEl.value.trim();
    if (!value) return;
    addItem(value);
    inputEl.value = "";
    inputEl.focus();
  });

  clearBtn.addEventListener("click", () => {
    clearChecked();
  });

  supabase
    .channel("shopping_items_changes")
    .on("postgres_changes", { event: "*", schema: "public", table: "shopping_items" }, () => {
      loadItems();
    })
    .subscribe();

  loadItems();
</script>
```

참고: 화면 갱신은 각 쓰기 함수가 아니라 Realtime 구독 콜백에서 `loadItems()`를 다시 호출하는 방식으로 이루어진다. 이벤트 종류(INSERT/UPDATE/DELETE)별로 배열을 부분 패치하는 대신 매번 전체를 재조회하는데, 이 앱 규모에서는 이 편이 더 단순하고 정렬/중복 버그 여지가 없다.

- [ ] **Step 3: 브라우저에서 기본 동작 확인**

`mcp__Claude_Browser__preview_start`로 `index.html`을 열고(파일 경로를 `file://`로 직접 열거나 정적 서버로 연다), 다음을 확인한다.
1. "테스트 아이템"을 입력하고 추가 버튼 클릭 → 목록에 나타나는지 확인
2. 체크박스 클릭 → 취소선이 적용되고 "1 / 1개 완료"로 바뀌는지 확인
3. Supabase MCP `execute_sql`로 `select id, text, checked from shopping_items;` 실행 → 방금 추가/체크한 행이 실제로 반영되어 있는지 확인
4. 삭제 버튼(✕) 클릭 → 목록에서 사라지고, `execute_sql`로 해당 행이 DB에서도 삭제됐는지 확인

Expected: 4단계 모두 화면과 DB 상태가 일치해야 한다.

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "Replace localStorage with Supabase for shopping list persistence"
```

---

### Task 3: 실시간 동기화 확인

**Files:** 없음 (동작 검증만)

**Interfaces:**
- Consumes: Task 2의 `index.html`

- [ ] **Step 1: 탭 두 개에서 실시간 반영 확인**

브라우저 탭을 두 개 열어 `index.html`을 각각 로드한다. 탭 A에서 아이템을 추가하고, 새로고침 없이 탭 B에도 나타나는지 확인한다. 탭 A에서 체크박스를 클릭하고, 탭 B의 체크 상태가 자동으로 바뀌는지 확인한다.

Expected: 탭 B가 새로고침 없이 탭 A의 변경을 몇 초 내로 반영해야 한다.

- [ ] **Step 2: 커밋**

검증만 하는 태스크로 코드 변경이 없다. 커밋할 것 없이 다음 태스크로 진행한다.

---

### Task 4: GitHub에 반영

**Files:** 없음 (git 작업만)

- [ ] **Step 1: 원격 저장소로 푸시**

```bash
git push origin main
```

Expected: `main` 브랜치가 `origin/main`으로 성공적으로 푸시됨 (Task 1의 스펙 커밋과 Task 2의 `index.html` 커밋 포함).

- [ ] **Step 2: 푸시 확인**

```bash
git log origin/main -1 --oneline
```

Expected: 로컬 `git log -1 --oneline`과 동일한 커밋 해시가 출력되어야 한다.
