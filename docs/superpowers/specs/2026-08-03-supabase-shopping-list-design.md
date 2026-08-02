# 쇼핑 리스트 Supabase 연동 설계

## 배경

쇼핑 리스트 앱(`index.html`)은 순수 HTML/JS로 작성된 단일 파일 앱으로, 로그인 기능 없이 아이템(`{id, text, checked}`)을 브라우저 `localStorage`에 저장한다. 이를 Supabase 데이터베이스(`shopping_items` 테이블)에 저장하도록 변경한다.

## 요구사항 결정 사항

- **인증**: 로그인 기능을 추가하지 않는다. Supabase 접근은 인증 없이 전체 공개로 허용한다. anon key는 클라이언트 코드에 노출되지만, 이는 anon key의 정상적인 사용 방식이며 실제 보호는 RLS 정책이 담당한다.
- **동기화**: 여러 탭/기기에서 동시에 열어두었을 때 Supabase Realtime을 통해 즉시 반영되어야 한다.
- **오프라인/에러 처리**: 네트워크 실패 시 별도 폴백(localStorage 캐시 등) 없이 간단한 에러 메시지만 표시한다.
- **기존 데이터**: 기존 localStorage 데이터는 마이그레이션하지 않는다. 학습용 프로젝트로 실 데이터가 없다고 판단했다.

## 스키마

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

- `id`는 클라이언트 생성 문자열 대신 DB가 생성하는 `uuid`를 사용한다.
- `created_at` 오름차순으로 정렬해 기존 "추가한 순서대로 표시" 동작을 유지한다.
- RLS는 활성화한 상태로 두고, 인증 없이도 select/insert/update/delete를 모두 허용하는 정책을 둔다.

## 클라이언트 아키텍처

**단일 진실 공급원(single source of truth) = Supabase.** 로컬 배열을 직접 조작하지 않고, 모든 변경은 Supabase에 쓴 뒤 realtime 구독이 전달하는 이벤트로만 화면을 갱신한다. 로컬 상태와 서버 상태가 어긋나는 문제를 구조적으로 피하기 위함이다.

- **라이브러리**: `@supabase/supabase-js` v2를 CDN(`https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2`)으로 로드한다. 빌드 과정이 없는 정적 HTML 앱이므로 CDN 방식이 적합하다.
- **초기화**: 프로젝트 URL과 publishable(anon) key를 코드 내 상수로 둔다.
- **초기 로드**: `select('*').order('created_at')`로 목록을 가져와 렌더링한다.
- **실시간 구독**: `shopping_items` 테이블의 INSERT/UPDATE/DELETE 이벤트를 구독하고, 이벤트 수신 시 로컬 `items` 배열을 갱신 후 재렌더링한다.
- **쓰기 함수** (`addItem`, `toggleItem`, `deleteItem`, 체크된 항목 일괄 삭제)는 Supabase에 insert/update/delete 요청만 보내며, 화면 갱신은 realtime 이벤트에 위임한다 (낙관적 업데이트를 하지 않는다).
- **에러 처리**: 요청 실패 시 짧은 에러 메시지를 표시한다. 로컬 상태를 미리 바꾸지 않으므로 롤백 로직이 불필요하다.
- `localStorage` 관련 코드(`STORAGE_KEY`, `loadItems`, `saveItems`)는 완전히 제거한다.

## 테스트 및 배포 계획

1. Supabase MCP `apply_migration`으로 위 스키마를 프로젝트에 적용.
2. 브라우저에서 `index.html`을 열어 추가/체크/삭제/일괄삭제 동작 확인.
3. 탭 두 개를 열어 한쪽 변경이 다른 쪽에 실시간 반영되는지 확인.
4. Supabase MCP로 테이블을 직접 조회해 실제 저장 여부 확인.
5. 확인 완료 후 `git commit` 및 `git push`로 GitHub(`github.com/logistex/shopping-list-app`)에 반영.

## 범위 제외

- 로그인/인증 기능 추가
- 기존 localStorage 데이터 마이그레이션
- 오프라인 폴백/캐시
