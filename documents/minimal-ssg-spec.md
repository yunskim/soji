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

## 0. 설계 철학

- **미니멀 코어**: 빌드 계약 3종 + `chrome` + 레지스트리 + 배포가 전부다. 기능은 실제 마찰이 생긴 뒤에만 추가한다.
- **단일 파이프라인 가정을 버린다**: 자체 데이터 수명주기·빌드 툴체인을 가진 아이템은 변환기 바깥에 둔다(아래 불변식).
- **설정 주도**: 코어는 사이트 중립적이다. 사이트별 값은 전부 설정·콘텐츠·테마에 있다.

## 1. 핵심 개념 (데이터 모델)

### 1.1 콘텐츠 아이템

사이트의 모든 공개 단위는 하나의 **콘텐츠 아이템**이다. 정체성(note/app)으로 나누지 않고,
아이템마다 **빌드 계약** 두 필드로 기술한다.

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

이렇게 분리하면 (1) 필드의 진실원본이 한 곳에 모이고, (2) 사이트마다 자기 필드를 추가할 수 있고(예: 젠트리피케이션 연구의 `stage`), (3) `new` 스캐폴드와 frontmatter 검증이 같은 스키마에서 파생된다.

주의: 이를 *아이템 타입별 스키마*로 키우지 않는다 — 그건 폐기한 정체성 이분법의 재림이다. 단일 스키마 + (필요 시) 컬렉션별 기본값 오버라이드까지만. 타입·검증 DSL은 실제 마찰 전엔 도입하지 않는다.

### 1.4 레지스트리

모든 아이템의 진실원본. 다만 모든 항목을 손으로 적지 않는다.

- **converter 아이템**: 콘텐츠 디렉터리 스캔으로 **자동 발견**. frontmatter가 메타데이터(스키마에 정의된 필드)를 제공한다.
- **prebuilt/command 아이템**: 콘텐츠 디렉터리 밖에 있고 빌드 지시가 필요하므로 레지스트리에 **명시 선언**한다.
- 레지스트리는 콘텐츠 렌더링 데이터가 아니라 **빌드 메타데이터**다.

## 2. 사이트 설정 (`site.yml`)

코어를 사이트 중립적으로 유지하는 단일 설정 파일. 최소 키:

- `title`, `base_url`
- `content_dir`(converter 아이템 소스), `output_dir`(빌드 산출물, 예: `site/`)
- `theme`(테마 이름 또는 경로)
- `item`(아이템 메타데이터 스키마 — 1.3. 필드별 `required`/`default`와 사이트 고유 필드)
- `features`(예: `math: katex` on/off 등 선택적 기능 토글)
- `nav`(정렬·그룹 규칙 등 전역 네비 설정)
- `deploy`(배포 전략과 그 파라미터 — 10장)
- `registry`(prebuilt/command 아이템 선언 — 인라인 또는 별도 파일 경로)

도구를 *이 설정 + 콘텐츠 디렉터리*에 겨누면 한 사이트가 빌드된다. 같은 코어로 다른
`site.yml`을 겨누면 다른 사이트가 나온다 — 이것이 일반화의 본질이다.

## 3. 콘텐츠 작성 규약 (converter 아이템)

저자가 무엇을 쓰는가. 변환기가 그걸 어떻게 처리하는지는 4장.

- 표준 Markdown + YAML frontmatter. frontmatter 필드는 §2의 `item` 스키마에서 정의된다(필수 `title`; 나머지는 default 보유).
- Tufte 요소: 사이드노트/마진노트, 전폭 그림은 변환기·테마가 약속한 구문(예: 각주 → 사이드노트)으로 쓴다.
- 수식: `features.math`가 켜져 있으면 인라인·블록 수식을 쓸 수 있다.
- **고정 사이드카 자산**: 같은 폴더에 고정 JSON/CSV/JS를 두고, 본문에서 `<script>`로 불러 D3 등으로 그릴 수 있다(1.2의 정적 범위 한정). 데이터가 샤딩·자체 갱신으로 바뀌면 `prebuilt`로 넘어간다.
- 드래프트: `draft: true`이거나 `_drafts/` 폴더의 파일은 빌드에서 제외된다.

## 4. 변환기 기능 상세 (`build: converter`)

변환기가 converter 아이템 하나를 처리할 때 수행하는 일:

1. **frontmatter 파싱·검증**: YAML frontmatter를 `item` 스키마로 검증하고 default를 채운다(필수 누락은 빌드 중단).
2. **Markdown → HTML**: 표준 Markdown + 확장(표, 각주, 코드펜스). 각주는 Tufte 사이드노트로 매핑.
3. **Tufte 요소 매핑**: 사이드노트/마진노트, 전폭 그림(figure), 표를 테마가 기대하는 마크업으로 변환.
4. **수식 렌더**: `features.math` 시 렌더. **빌드 시점 렌더를 기본**으로(무 JS에서도 보임), 클라이언트 렌더는 폴백.
5. **코드 하이라이팅**: 빌드 시점 하이라이팅(선택적 기능).
6. **고정 사이드카 자산·passthrough**: 같은 폴더의 고정 JSON/CSV/JS를 출력으로 복사, 원시 HTML과 `<script>` 임베드를 통과(1.2의 정적 범위 한정).
7. **head 메타데이터 생성**: `title`, `meta-description`, `meta-viewport`, canonical(`base_url`+slug), 파비콘, 테마 CSS/JS 링크.
8. **chrome(테마 레이아웃) 적용**: `chrome: true`면 본문을 테마 셸(헤더·네비·푸터·컨테이너)로 감싼다(5장 경계).
9. **링크·자산 경로 해석**: 상대 링크·자산을 `output_dir/<slug>/` 구조에 맞게 해석.
10. **출력 + 인덱스 입력 제공**: `output_dir/<slug>/index.html`(+자산)을 기록하고, 인덱스·네비 생성을 위한 메타데이터(title/date/tags)를 방출.

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

**선택적 일관성 레이어 (비침습, opt-in).** 시각적 일관성이 필요하면, 테마가 베이스 CSS(와
선택적으로 헤더/푸터 프래그먼트)를 **안정적인 URL로 노출**하고, prebuilt 앱이 *자기 빌드
시점에 스스로* 가져다 쓸 수 있다. 도구가 앱 문서에 주입하지 않으므로 독립성은 유지된다.
이 공유 자산 export를 v1에 넣을지는 미해결(12장).

## 6. 테마 / 템플릿

`chrome`이 입히는 공유 외형은 **테마**가 제공한다. 특정 스타일을 코어에 박지 않는다.

- 테마 = 디렉터리. 제공물: 페이지 템플릿, 목록/인덱스 템플릿, 네비 파셜, 정적 자산 번들(CSS/JS).
- 코어는 변환된 본문을 테마 템플릿에 끼워 넣고, 테마 자산을 출력에 복사한다.
- **기본 테마 = Tufte**(Tufte CSS + 선택적 KaTeX)를 함께 배포한다. 교체 가능하다.
- `chrome: false` 아이템에는 테마를 입히지 않는다(불투명 처리). 5장의 opt-in 공유 자산은 테마가 노출만 한다.

## 7. 빌드 파이프라인

빌드 명령이 설정·레지스트리를 읽어 수행한다. **외부 SSG 의존성 0.**

1. **발견**: `content_dir`를 스캔해 converter 아이템 수집 + 레지스트리에서 prebuilt/command 아이템 수집. 각 아이템을 `item` 스키마로 검증·기본값 채움.
2. **변환기 아이템 빌드**(4장): Markdown → HTML + 테마 적용(`chrome: true`) + 고정 사이드카 자산 복사 → `output_dir/<slug>/`. 드래프트 제외.
3. **자체 빌드 아이템 처리**(5장): `prebuilt`는 산출물 디렉터리를 그대로 복사. 빌드 명령이 선언된 경우만 실행 후 복사. 내부는 건드리지 않는다.
4. **인덱스 생성**: 컬렉션별 날짜 역순 목록 페이지 + 루트 인덱스를 생성.
5. **조립**: 레지스트리·설정에서 상단 네비를 생성해 `chrome: true` 아이템에 주입.
6. **멱등성**: 같은 입력은 항상 같은 `output_dir`를 만든다.
7. **증분 정책**: 일상 빌드는 자체 수명주기를 가진 아이템 산출물을 재빌드하지 않는다(독립 갱신 보존).

## 8. CLI / 명령

`build.py`/`deploy.sh`를 서브커맨드를 가진 도구로 일반화한다.

- `build` — 7장 파이프라인 실행, `output_dir` 생성.
- `serve` — 로컬 미리보기(가능하면 라이브 리로드). 선택적.
- `deploy` — `build` 후 설정의 배포 전략 실행(10장).
- `new` — 새 converter 아이템 스캐폴드. **`item` 스키마(1.3)의 required 필드 + default로 frontmatter 골격을 생성**한다.

도구 이름은 미정(12장).

## 9. 사이트 구조·인덱스·네비게이션

- URL 규약: `output_dir/<slug>/...`. 컬렉션은 하위 경로(예: `<collection>/<date>-<slug>/`).
- 상단 네비: `nav: true` 아이템을 `order`대로 생성, `chrome: true` 아이템에 주입. 추가·삭제가 한 곳에서 끝난다.
- 루트 인덱스: 레지스트리·발견 결과에서 생성.
- `chrome: false` 아이템은 자체 내부 네비를 유지. 코어는 루트→아이템, 아이템→홈 링크 수준만 보장.

## 10. 배포 (플러그블)

코어의 책임은 `output_dir`까지다. 배포는 설정으로 고른 **전략**이 수행한다. 한 전략을
레퍼런스로 명시한다.

**레퍼런스 전략 — GitHub Pages (private → public rsync):**

- 소스·드래프트·빌드 스크립트는 비공개 저장소, 빌드된 `output_dir`만 공개 저장소(Pages 호스팅)로 내보낸다.
- `output_dir`를 공개 저장소의 완전한 진실원본으로 둔다. `CNAME`, `.nojekyll`까지 그 안에 포함시켜 `--delete` 동기화가 안전하도록 한다.
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
- 아이템 타입별 스키마, 타입·검증 DSL(현재 `item` 스키마는 required/default 수준)
- 플러그인 API의 일반화(현재는 `features` 토글 수준)
- (레지스트리·고정 사이드카 자산 복사·테마 적용·`item` 스키마는 코어에 포함된다.)

**미해결 결정 사항:**

- 도구·단일 개념의 이름(중립적 `entry`/`item` vs 관용어).
- prebuilt용 **opt-in 공유 자산 export**(테마 베이스 CSS·헤더/푸터 프래그먼트)를 v1에 넣을지(5장).
- `item` 스키마 검증을 어디까지 둘지(현재 required/default만; 타입·열거형 보류).
- 테마 패키징 방식(디렉터리 규약 vs 설정 매니페스트).
- `serve` 라이브 리로드 포함 여부(선택).

---

## 부록 A: 예시 인스턴스 — yunskim.github.io

코어는 사이트를 모른다. 아래는 이 SSG의 **한 인스턴스**일 뿐이다.

### A.1 site.yml

```yaml
title: yunskim
base_url: https://yunskim.github.io
content_dir: content/
output_dir: site/
theme: tufte                # 기본 테마
features:
  math: katex
item:                       # 아이템 메타데이터 스키마 (1.3)
  fields:
    title:       { required: true }
    date:        { required: false }
    description: { required: false }
    tags:        { default: [] }
    nav:         { default: false }
    order:       { default: 100 }
    draft:       { default: false }
    chrome:      { default: true }    # 구조적 의미, 사이트 기본값만 설정
    stage:       { required: false }  # 사이트 고유 필드(예: 젠트리피케이션 1~4기)
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

# 1) 빌드: 발견 + converter 변환(+고정 자산) + prebuilt 수집 + 인덱스·네비
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
