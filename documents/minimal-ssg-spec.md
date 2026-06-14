---
meta-description: 변환된 Markdown과 자체 빌드 앱을 하나의 빌드 계약 모델로 섞어 다루는 미니멀 정적 사이트 생성기(SSG)의 명세.
meta-viewport: width=device-width, initial-scale=1
title: 미니멀 정적 사이트 생성기 — 명세
date: 2026-06-04
tags: [ssg, infra, spec]
draft: false
---

# 미니멀 정적 사이트 생성기 — 명세

특정 사이트(`yunskim.github.io`)를 위해 다듬어 온 *변환기 + 레지스트리 + 빌드 계약 + 배포*
시스템을 일반화한 명세다. 도메인·아이템 목록·테마처럼 사이트에 박혀 있던 값을 **설정으로
빼내** 임의의 사이트를 만들 수 있게 한다. 기능을 더하지 않는다 — 일반화란 하드코딩된 가정을
설정으로 옮기는 것이지 능력을 늘리는 것이 아니다.

이 SSG의 차별점은 단 하나다: 대부분의 SSG가 "모든 콘텐츠는 하나의 변환 파이프라인을
통과한다"고 가정하는 반면, 이 도구는 **변환된 Markdown과 자체 데이터·빌드를 가진 앱을
같은 레지스트리·배포 아래에서 깨끗이 공존**시킨다. 그 경계를 명시적 빌드 계약으로 다룬다.

도구 명령은 `ssg`(교체 가능한 기본 이름), 단일 콘텐츠 개념은 `item`이라 부른다.

## 0. 설계 철학

- **미니멀 코어**: 빌드 계약 3종 + `chrome` + 레지스트리 + 배포가 전부다. 기능은 실제 마찰이 생긴 뒤에만 추가한다.
- **단일 파이프라인 가정을 버린다**: 자체 데이터 수명주기·빌드 툴체인을 가진 아이템은 변환기 바깥에 둔다(아래 불변식).
- **설정 주도**: 코어는 사이트 중립적이다. 사이트별 값은 전부 설정·콘텐츠·테마에 있다.

## 1. 핵심 개념 (데이터 모델)

### 1.1 콘텐츠 아이템

사이트의 모든 공개 단위는 하나의 **콘텐츠 아이템**(이하 *item*)이다. 정체성(note/app)으로
나누지 않고, 아이템마다 **빌드 계약** 두 필드로 기술한다.

- `build:` — 어떻게 빌드되는가. `converter`(Markdown → HTML, 고정 사이드카 자산 동봉) / `prebuilt`(미리 빌드된 산출물 수집) / 빌드 명령 문자열.
- `chrome:` — 테마의 공유 외형(레이아웃·네비·스타일)을 입힐 것인가. `true` / `false`.

두 필드는 대체로 직교하되, `chrome: true`는 빌드가 본문 프래그먼트를 내줄 때만 가능하다(5장).

### 1.2 load-bearing 불변식: 빌드 시점 고정 자산 vs 자체 데이터 수명주기

경계는 "런타임에 데이터를 불러오느냐"가 **아니다** — 그건 너무 넓다. 고정된 JSON 하나를
브라우저에서 `fetch`해 그리는 페이지도, 그 JSON이 빌드에 함께 실리는 **고정 자산**이라면
정적 문서이며 변환기가 다룬다.

변환기 바깥(`prebuilt`/명령)으로 밀어내야 하는 것은 **자체 데이터 수명주기 또는 자체 빌드
툴체인을 가진** 아이템이다: 샤딩 데이터셋을 커스텀 JS로 스티칭하거나, 콘텐츠와 무관한 자체
주기로 갱신되는 데이터를 쓰거나, 별도 번들러·프레임워크로 빌드돼야 하는 경우. 변환기가
이를 소유하려는 순간 데이터 생산·갱신 주기까지 떠안게 되고, 그게 SSG 비대화다.

**판정 테스트 (아이템별 한 줄):**

> 이 아이템에 필요한 모든 것(HTML + 데이터 파일)을 빌드 시점에 고정된 자산으로 함께 실어 보낼 수 있는가?

- 예 → `build: converter` (고정 JSON·CSV를 런타임에 읽어도 무방)
- 아니오 (자체 데이터 수명주기·샤딩 / 자체 빌드 툴체인) → `build: prebuilt` 또는 빌드 명령

### 1.3 아이템의 구조: 핵심 필드 vs 선언된 스키마

아이템의 필드는 두 층으로 나뉜다.

- **핵심(구조) 필드 — 도구 소유, 설정 불가:** `slug`, `build`. 도구의 *동작*이 이 값에 분기하므로 코어가 의미를 정의한다. (`build`는 converter 아이템에선 콘텐츠 디렉터리 위치로 함의되고, prebuilt/command 아이템에선 명시된다.)
- **메타데이터 스키마 — `site.yml`에서 선언:** `title`·`date`·`tags`·`nav`·`order`·`draft`·`chrome` 등 서술 필드의 *required/default*, 그리고 **사이트 고유 필드**. 이 스키마가 converter 아이템의 frontmatter와 선언 아이템의 레지스트리 항목 **양쪽에 동일하게** 적용된다.

검증 범위는 **required/default + default 리터럴에서 추론되는 최소 타입**(date=날짜, tags=리스트,
bool=불리언)까지다. 도구가 정렬·필터를 하려면 이미 필요한 수준이다. 선언적 type/enum DSL
(예: `stage ∈ {1..4}`)은 두지 않는다(§12).

이렇게 분리하면 (1) 필드의 진실원본이 한 곳에 모이고, (2) 사이트마다 자기 필드를 추가할 수 있고(예: 젠트리피케이션 연구의 `stage`), (3) `new` 스캐폴드와 frontmatter 검증이 같은 스키마에서 파생된다.

주의: 이를 *아이템 타입별 스키마*로 키우지 않는다 — 그건 폐기한 정체성 이분법의 재림이다. 단일 스키마 + 컬렉션별 기본값 오버라이드(§9)까지만.

### 1.4 레지스트리

모든 아이템의 진실원본. 다만 모든 항목을 손으로 적지 않는다.

- **converter 아이템**: 콘텐츠 디렉터리 스캔으로 **자동 발견**. frontmatter가 메타데이터(스키마에 정의된 필드)를 제공한다.
- **prebuilt/command 아이템**: 콘텐츠 디렉터리 밖에 있고 빌드 지시가 필요하므로 레지스트리에 **명시 선언**한다.
- 레지스트리는 콘텐츠 렌더링 데이터가 아니라 **빌드 메타데이터**다.

## 2. 사이트 설정 (`site.yml`)

코어를 사이트 중립적으로 유지하는 단일 설정 파일.

**정체성·URL**
- `title`, `base_url`
- `path_prefix` — 서브경로 호스팅 시 링크·자산 접두사(§4.9). apex면 `/`.
- `lang` — `<html lang>` 기본값.
- `title_template` — 페이지 타이틀 조립(예: `"{title} · yunskim"`).
- `author`, `description` — head 메타 기본값.

**경로**
- `content_dir` — converter 아이템 소스.
- `output_dir` — 빌드 산출물(예: `site/`).
- `static_dir` — 출력 루트로 **그대로 복사**되는 정적 자산(favicon, robots.txt, CNAME, .nojekyll, 루트 이미지 등).

**렌더링**
- `theme` — 내장 테마 이름 또는 디렉터리 경로(§6).
- `date_format`, `timezone` — 인덱스 정렬·표시(§7·§9)의 파싱/표시 기준.
- `features` — 선택적 기능 토글. 예: `math: katex`, `raw_html: false`(원시 HTML/`<script>` passthrough; 기본 off).

**빌드 동작** (top-level `build:`는 `item.build`와 혼동되므로 플랫 키로 둔다)
- `clean` — 빌드 전 `output_dir`를 비운다(멱등성, §7).
- `include_drafts` — 드래프트 포함 여부(기본 false; `serve`에서 보통 true로 오버라이드).
- `publish_future` — 미래 날짜 글 포함 여부(기본 false).

**구조**
- `item` — 아이템 메타데이터 스키마(§1.3).
- `collections` — 컬렉션 정의(§9).
- `nav` — 정렬·그룹 규칙 등 전역 네비 설정.
- `deploy` — 배포 전략과 파라미터(§10).
- `registry` — prebuilt/command 아이템 선언(인라인 또는 별도 파일 경로).

도구를 *이 설정 + 콘텐츠 디렉터리*에 겨누면 한 사이트가 빌드된다. 같은 코어로 다른
`site.yml`을 겨누면 다른 사이트가 나온다 — 이것이 일반화의 본질이다.

## 3. 콘텐츠 작성 규약 (converter 아이템)

저자가 무엇을 쓰는가. 변환기가 그걸 어떻게 처리하는지는 4장.

- 표준 Markdown + YAML frontmatter. frontmatter 필드는 §2의 `item` 스키마에서 정의된다(필수 `title`; 나머지는 default 보유).
- Tufte 요소: 사이드노트/마진노트, 전폭 그림은 변환기·테마가 약속한 구문(예: 각주 → 사이드노트)으로 쓴다.
- 수식: `features.math`가 켜져 있으면 인라인·블록 수식을 쓸 수 있다.
- **고정 사이드카 자산**: 같은 폴더에 고정 JSON/CSV/JS를 두고, 본문에서 `<script>`로 불러 D3 등으로 그릴 수 있다 — 단 `features.raw_html`이 켜져 있어야 한다(1.2의 정적 범위 한정). 데이터가 샤딩·자체 갱신으로 바뀌면 `prebuilt`로 넘어간다.
- 드래프트: `draft: true`이거나 `_drafts/` 폴더의 파일은 `include_drafts`가 false면 빌드에서 제외된다.

## 4. 변환기 기능 상세 (`build: converter`)

변환기가 converter 아이템 하나를 처리할 때 수행하는 일:

1. **frontmatter 파싱·검증**: YAML frontmatter를 `item` 스키마로 검증하고 default를 채운다(필수 누락은 빌드 중단).
2. **Markdown → HTML**: 표준 Markdown + 확장(표, 각주, 코드펜스). 각주는 Tufte 사이드노트로 매핑.
3. **Tufte 요소 매핑**: 사이드노트/마진노트, 전폭 그림(figure), 표를 테마가 기대하는 마크업으로 변환.
4. **수식 렌더**: `features.math` 시 렌더. **빌드 시점 렌더를 기본**으로(무 JS에서도 보임), 클라이언트 렌더는 폴백.
5. **코드 하이라이팅**: 빌드 시점 하이라이팅(선택적 기능).
6. **고정 사이드카 자산·passthrough**: 같은 폴더의 고정 JSON/CSV/JS를 출력으로 복사. 원시 HTML·`<script>` 임베드 통과는 `features.raw_html` 토글에 따른다(기본 off; 1.2의 정적 범위 한정).
7. **head 메타데이터 생성**: `lang`, `title_template`로 만든 타이틀, `meta-description`(없으면 사이트 `description`), `author`, `meta-viewport`, canonical(`base_url`+slug), 파비콘, 테마 CSS/JS 링크.
8. **chrome(테마 레이아웃) 적용**: `chrome: true`면 본문을 테마 셸(헤더·네비·푸터·컨테이너)로 감싼다(5장 경계).
9. **링크·자산 경로 해석**: 상대 링크·자산을 `path_prefix` + `output_dir/<permalink>` 구조에 맞게 해석.
10. **출력 + 인덱스 입력 제공**: 퍼머링크(§9)에 따라 `output_dir/.../index.html`(+자산)을 기록하고, 인덱스·네비 생성을 위한 메타데이터(title/date/tags)를 방출.

## 5. prebuilt / command 경계 — 변환기 기능이 어디까지 적용되나

핵심 질문: prebuilt 아이템은 "아예 따로"인가, 변환기 기능을 일부 물려받는가. 결정한다.

**문서 조립 기능은 prebuilt에 적용하지 않는다.** 본문 렌더(4장 2·3), 수식·하이라이팅(4·5),
head 주입(7), chrome 래핑(8)은 *완결된 HTML 문서*를 가진 prebuilt 앱에는 적용하지 않는다.
변환기가 앱의 문서를 다시 쓰지 않기 때문이고, "자체 빌드 아이템은 도구 없이도 독립 동작"
원칙(11장)과 직결된다.

**적용되는 것은 사이트 수준 비침습 기능뿐이다.** 레지스트리 메타데이터(title/date/description)는
검증되어 **루트 인덱스·네비 목록에 아이템을 포함**시키는 데 쓰인다. 이는 앱 문서를 건드리지 않는다.

**chrome의 진짜 분기선 = 프래그먼트냐 완결 문서냐.** `chrome: true`는 빌드 단계가
**본문 프래그먼트**를 내줄 때만 가능하다.

- `converter` → 항상 프래그먼트 → `chrome` 선택 가능(보통 true).
- `command` → 프래그먼트를 내주면 `chrome: true` 가능, 완결 트리를 내주면 `chrome: false`.
- `prebuilt`(완결 자체 문서) → `chrome: false` 고정.

**선택적 일관성 레이어 (비침습, opt-in) — v1 비포함, 후속 확장점.** 시각적 일관성이 필요해지면,
테마가 베이스 CSS(와 선택적 헤더/푸터 프래그먼트)를 안정 URL로 노출하고 prebuilt 앱이 *자기
빌드에서 스스로* 가져다 쓰게 한다(도구가 주입하지 않으므로 독립성 유지). **도입 트리거:
어떤 prebuilt 앱이 실제로 변환 페이지와 시각적 일관성을 요구할 때.** 그 전엔 만들지 않는다.

## 6. 테마 / 템플릿

`chrome`이 입히는 공유 외형은 **테마**가 제공한다. 특정 스타일을 코어에 박지 않는다.

- 테마 = **디렉터리 규약**(매니페스트 없음). 고정 레이아웃:

  ```
  themes/<name>/
    templates/page.html    # converter 본문을 감싸는 셸 (chrome)
    templates/index.html   # 루트·컬렉션 인덱스
    partials/nav.html      # 상단 네비
    partials/head.html     # (선택) head 조각
    assets/                # css/js/폰트 → 출력으로 복사
  ```

- `theme:`는 내장 이름(`tufte`) 또는 위 레이아웃을 따르는 디렉터리 경로.
- 코어는 변환된 본문을 `templates/page.html`에 끼워 넣고, `assets/`를 출력에 복사한다.
- **기본 테마 = Tufte**(Tufte CSS + 선택적 KaTeX)를 함께 배포한다. 교체 가능.
- `chrome: false` 아이템에는 테마를 입히지 않는다(불투명 처리).
- 테마 옵션 파라미터가 필요해지면 그때 작은 `theme.yml`을 추가한다(현재 비포함, §12).

## 7. 빌드 파이프라인

빌드 명령이 설정·레지스트리를 읽어 수행한다. **외부 SSG 의존성 0.**

1. **클린**: `clean`이면 `output_dir`를 비운다(잔재 제거, 멱등성 보장).
2. **정적 패스스루**: `static_dir`를 출력 루트로 그대로 복사(`CNAME`·`.nojekyll`·favicon·robots.txt 등). 생성물과 충돌하면 경고.
3. **발견**: `content_dir` 스캔(컬렉션 매칭 포함) + 레지스트리에서 선언 아이템 수집. 각 아이템을 `item` 스키마로 검증·기본값 채움. 드래프트·미래글은 `include_drafts`/`publish_future`에 따라 포함/제외.
4. **변환기 아이템 빌드**(4장): Markdown → HTML + 테마 적용(`chrome: true`) + 고정 사이드카 자산 복사 → 퍼머링크 경로(§9).
5. **자체 빌드 아이템 처리**(5장): `prebuilt`는 산출물 디렉터리를 그대로 복사. 빌드 명령이 선언된 경우만 실행 후 복사. 내부는 건드리지 않는다.
6. **인덱스 생성**: 루트 인덱스 + 컬렉션별 인덱스(날짜 역순).
7. **조립**: 레지스트리·설정에서 상단 네비를 생성해 `chrome: true` 아이템에 주입.
8. **멱등성·증분**: 같은 입력은 같은 `output_dir`를 만든다. 일상 빌드는 자체 수명주기를 가진 아이템 산출물을 재빌드하지 않는다(독립 갱신 보존).

## 8. CLI / 명령

`build.py`/`deploy.sh`를 서브커맨드를 가진 도구 `ssg`로 일반화한다.

- `ssg build` — 7장 파이프라인 실행, `output_dir` 생성.
- `ssg serve` — 빌드(드래프트 포함) 후 `output_dir`를 정적 서빙. **v1은 watch·라이브 리로드 없음**(후속, §12).
- `ssg deploy` — `build` 후 설정의 배포 전략 실행(10장).
- `ssg new` — 새 converter 아이템 스캐폴드. **`item` 스키마(1.3)의 required 필드 + default로 frontmatter 골격을 생성**한다.

`ssg`는 교체 가능한 기본 이름이다(설계와 무관).

## 9. 사이트 구조·인덱스·네비게이션

- **기본 퍼머링크**: `path_prefix` + `<slug>/`. 출력은 `output_dir/<slug>/index.html`.
- **컬렉션**: `content_dir`의 서브셋을 묶는 단위. `collections.<name>`에 정의한다.
  - `source` — 컬렉션 소스 디렉터리.
  - `permalink` — 퍼머링크 패턴(토큰 `{slug}`, `{date}`, `{year}`/`{month}` 등). 예: `garosu-gil/log/{date}-{slug}/`.
  - `index` — 컬렉션 자체 인덱스 페이지(날짜 역순) 생성 여부.
  - `defaults` — (선택) 이 컬렉션 아이템에 적용할 스키마 기본값 오버라이드(예: `nav: false`).
- **상단 네비**: `nav: true` 아이템을 `order`대로 생성, `chrome: true` 아이템에 주입. 추가·삭제가 한 곳에서 끝난다.
- **루트 인덱스**: 레지스트리·발견 결과에서 생성.
- `chrome: false` 아이템은 자체 내부 네비를 유지. 코어는 루트→아이템, 아이템→홈 링크 수준만 보장.

## 10. 배포 (플러그블)

코어의 책임은 `output_dir`까지다. 배포는 설정으로 고른 **전략**이 수행한다. 한 전략을
레퍼런스로 명시한다.

**레퍼런스 전략 — GitHub Pages (private → public rsync):**

- 소스·드래프트·빌드 스크립트는 비공개 저장소, 빌드된 `output_dir`만 공개 저장소(Pages 호스팅)로 내보낸다.
- `CNAME`·`.nojekyll`은 `static_dir`를 통해 `output_dir`에 들어가므로(§7.2), `output_dir`가 공개 저장소의 완전한 진실원본이 된다. `--delete` 동기화가 안전하다.
- 사전 빌드 HTML이므로 `.nojekyll`로 Jekyll 처리를 끈다.
- rsync 시 `.git`은 반드시 제외(공개 저장소 히스토리 보존).
- `build → rsync → commit/push`를 단일 스크립트로 묶어, 공개 저장소는 손으로 건드리지 않는다.
- 후속 자동화: 비공개 저장소의 CI가 공개 저장소로 푸시(deploy key/PAT)하도록 이전 가능.

다른 전략(임의의 디렉터리·호스트로 rsync/복사 등)도 같은 인터페이스로 끼울 수 있다.

## 11. 비기능 요구사항

- 모바일 반응형은 테마 책임(`meta-viewport` 등).
- 의존성 최소화. 외부 CDN 사용은 설정/테마에서 정책 명시.
- 합리적 빌드 속도, 빌드 실패·링크 깨짐 검출(최소한 빌드 중단으로 표면화).
- 결합도 최소화: 자체 빌드 아이템은 도구 없이도 독립 동작 가능해야 한다(5장의 근거).
- 이식성: 코어는 사이트별 값을 담지 않는다(전부 설정·콘텐츠·테마).

## 12. 범위 경계 (v1 비포함)

실제 마찰이 생긴 뒤에 추가한다. 코어를 작게 유지하기 위한 의도적 제외다.

- 백링크/위키링크, 클라이언트 검색, RSS/Atom 피드
- 상태 대시보드, 콘텐츠 데이터 파일 렌더링, 콘텐츠 타입별 템플릿 분리, 시리즈 네비게이션
- prebuilt 문서로의 chrome/head 주입(5장: 비침습 원칙상 영구 제외)
- prebuilt용 opt-in 공유 자산 export(5장: 트리거 충족 시 후속)
- 아이템 타입별 스키마, 선언적 type/enum 검증 DSL(현재 required/default + 추론 최소 타입까지)
- 테마 매니페스트·옵션 파라미터(`theme.yml`) — 디렉터리 규약으로 충분할 때까지
- `serve` watch·라이브 리로드
- 플러그인 API의 일반화(현재는 `features` 토글 수준)
- 후속(정적/생성물로 처리 가능): sitemap.xml, 404 페이지, 자산 minify·fingerprint, 리다이렉트/alias, OG·소셜 이미지 기본값
- (레지스트리·고정 사이드카 자산 복사·테마 적용·`item` 스키마·`static_dir`·컬렉션은 코어에 포함된다.)

**결정 로그 (이전 미해결 항목 → 확정):**

- **이름**: 단일 개념 = `item`, 도구 = `ssg`(교체 가능한 기본값).
- **opt-in 공유 자산 export**: v1 비포함. 트리거 충족 시 후속(5장).
- **스키마 검증**: required/default + default 리터럴 추론 최소 타입까지. 선언적 type/enum DSL 후속.
- **테마 패키징**: 디렉터리 규약, 매니페스트 없음(6장).
- **`serve`**: 빌드 후 정적 서빙(드래프트 포함), 라이브 리로드 없음.

---

## 부록 A: 예시 인스턴스 — yunskim.github.io

코어는 사이트를 모른다. 아래는 이 SSG의 **한 인스턴스**일 뿐이다.

### A.1 site.yml

```yaml
title: yunskim
base_url: https://yunskim.github.io
path_prefix: /
lang: ko
title_template: "{title} · yunskim"
author: yunskim
description: 연구 노트와 인터랙티브 시각화.

content_dir: content/
output_dir: site/
static_dir: static/          # favicon, robots.txt, CNAME, .nojekyll 등 → 출력 루트로 복사

theme: tufte
date_format: "YYYY-MM-DD"
timezone: Asia/Seoul
features:
  math: katex
  raw_html: true             # 고정 JSON + <script> 시각화를 허용

clean: true
include_drafts: false        # serve에서 true로 오버라이드
publish_future: false

item:                        # 아이템 메타데이터 스키마 (1.3)
  fields:
    title:       { required: true }
    date:        { required: false }
    description: { required: false }
    tags:        { default: [] }
    nav:         { default: false }
    order:       { default: 100 }
    draft:       { default: false }
    chrome:      { default: true }
    stage:       { required: false }   # 사이트 고유 필드(젠트리피케이션 1~4기)

collections:
  log:                        # 가로수 진행 로그
    source: content/garosu-gil/log/
    permalink: "garosu-gil/log/{date}-{slug}/"
    index: true
    defaults: { nav: false }

deploy:
  strategy: github-pages-rsync
  public_repo: ../yunskim.github.io/
registry: registry.yml
```

### A.2 registry.yml (prebuilt/command 아이템만 명시)

```yaml
items:
  - slug: boxoffice
    title: Box Office
    description: 영화진흥위원회 일별 박스오피스 흐름.
    build: prebuilt          # 샤딩·독립 갱신 데이터 → 판정 테스트 "아니오"
    chrome: false            # 완결 문서 → chrome 고정 false (5장)
    nav: true
    order: 10
  - slug: apartments
    title: Apartments
    description: 아파트 실거래를 점 이벤트로 본 Hawkes process.
    build: prebuilt
    chrome: false
    nav: true
    order: 40
# garosu-gil, notebook 등 converter 아이템은 content/ 스캔으로 자동 발견.
# 고정 JSON 하나를 fetch해 그리는 단순 시각화는 converter/chrome:true 로 둔다(prebuilt 아님).
```

### A.3 deploy (github-pages-rsync 전략 구현)

```bash
#!/usr/bin/env bash
set -euo pipefail

SRC="site/"                      # output_dir
DEST="../yunskim.github.io/"     # public_repo

# 1) 빌드: 클린 + 정적 패스스루 + 발견 + converter 변환 + prebuilt 수집 + 인덱스·네비
ssg build

# 2) 동기화: output_dir 내용을 공개 저장소로, 삭제분 반영, .git 보존
rsync -av --delete --exclude='.git' "$SRC" "$DEST"

# 3) 커밋·푸시
cd "$DEST"
git add -A
git commit -m "site: rebuild $(date +%F)" || echo "변경 없음, 커밋 생략"
git push
```

`SRC`의 후행 슬래시는 "폴더가 아니라 내용물을 복사"를 뜻한다. `--delete`는 삭제·개명된
아이템의 잔재까지 정리하고, `CNAME`·`.nojekyll`이 `site/` 안에 있어 함께 보존된다.
