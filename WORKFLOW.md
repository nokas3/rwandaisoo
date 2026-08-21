# 월드비전 후원 결과보고서 → 웹페이지 제작 가이드

이 문서는 `rwandaisoo` 리포지토리에서 진행한 작업 방식을 정리한 것입니다. 앞으로 다른
후원 결과보고서(PPT 산출물)를 같은 방식으로 웹페이지화할 때 이 문서를 참고해 동일한
파이프라인과 스타일을 재현하세요.

## 1. 전체 흐름

1. 사용자가 결과보고서 PPT(또는 이미 변환된 단일 HTML)를 하나 제공한다.
2. PPT 내용을 **이미지·텍스트를 전부 인라인으로 포함한 단일 HTML 파일**로 만든다
   (외부 리소스 의존 없이 그 파일 하나로 완전히 렌더링되어야 함 — 폰트 CDN 정도만 예외).
3. 그 폴더를 새 git 저장소로 만들고 GitHub에 푸시한다.
4. Vercel에 GitHub 저장소를 연결해 `main` 브랜치 푸시 시 자동 배포되게 한다.
5. 이후 사용자가 요청하는 문구/스타일/사진 수정은 `index.html`을 직접 편집 → 커밋 → 푸시
   순서로 반영한다 (Vercel이 자동으로 프로덕션에 재배포함).

## 2. 파일 구조 규칙

- 사이트의 루트에서 열리는 파일은 반드시 **`index.html`** 이어야 한다 (Vercel이 정적 사이트로
  서빙할 때 루트 경로가 이 파일을 찾음).
- 원본 산출물 이름을 살린 파일(예: `2026 르완다 ... 최종보고서_전이수 후원자님.html`)도
  함께 유지한다. **수정할 때마다 두 파일을 항상 동일하게 유지**해야 한다:
  ```bash
  cp index.html "원본 파일명.html"
  ```
  편집은 `index.html`에만 하고, 커밋 직전에 위 명령으로 복사해서 동기화한다.
- 사진/로고는 전부 `<img src="data:image/...;base64,...">` 형태로 파일 안에 인라인 포함한다.
  외부 이미지 파일을 따로 두지 않는다 (단일 파일 배포 원칙).

## 3. Git / Vercel 배포 파이프라인

### 최초 1회 설정
```bash
git init
git config user.email "<사용자 이메일>"
git config user.name "<github 사용자명>"
git add -A
git commit -m "..."
git branch -M main
git remote add origin https://github.com/<user>/<repo>.git
git push -u origin main
```
- GitHub 저장소는 미리 비어있는 상태로 생성되어 있어야 한다 (`git ls-remote`로 확인 가능).
- Vercel 배포는 Vercel MCP 커넥터(`mcp__<id>__*` 툴들, `deploy_to_vercel` / `create_git_project`
  / `list_projects` / `list_deployments` / `get_project`)로 진행한다.
  - `create_git_project`로 GitHub 저장소를 Vercel 프로젝트에 연결하면 이후 `main` 브랜치
    푸시마다 **자동으로 프로덕션 배포**된다.
  - 만약 "GitHub 연동을 설치해야 한다"는 에러(`Install GitHub App`)가 나오면, 사용자에게
    `https://github.com/apps/vercel` 에서 앱을 설치하고 해당 저장소 접근 권한을 부여해
    달라고 요청한 뒤 `create_git_project`를 다시 호출한다.
  - 연동 전에 이미 `deploy_to_vercel`(파일 직접 업로드 방식)로 프로젝트가 만들어졌다면,
    git 연동 후에도 기존 프로젝트가 재사용된다(`Reused project ...`).

### 매 수정 후
```bash
cp index.html "<원본 파일명>.html"
git add -A
git commit -m "<변경 요약>"
git push origin main
```
푸시 후 `list_deployments`로 최신 배포가 `state: READY`, `target: production`인지 확인하고,
필요하면 Browser 툴로 실제 사이트(`https://<project>.vercel.app`)를 열어 육안 확인한다.

## 4. 브랜드 / 디자인 규칙 (World Vision Korea)

- 색상: 오렌지 `#FF5515`(`--orange`), 진한 오렌지 `#E32C13`(`--orange-900`), 미드나이트
  `#111222`, 그레이 스케일 `--grey-800~500`.
- 폰트: Pretendard(CDN) 본문, Noto Serif KR은 인용구(`.serif`)에만.
- 섹션 번호(01, 02 …)는 굵게(`font-weight:800`) + 브랜드 오렌지(`--orange`, `--orange-900`
  아님 — 사용자가 명시적으로 "월드비전 주황색"을 요청하면 `--orange`를 쓴다).
- 로고 이미지 원본 PNG는 투명 여백이 매우 크게(전체의 80%+) 포함되어 있는 경우가 많다.
  `<img>`에 `height`만 키우면 여백까지 같이 커져서 상단바가 불필요하게 비대해진다 →
  **반드시 알파채널 바운딩 박스를 찾아 크롭한 버전을 별도로 만들어 써야 한다** (5장 참고).

## 5. 이미지 처리 (base64 인라인 + PowerShell)

이 환경(Windows)에는 Python/ImageMagick이 없다. **PowerShell + System.Drawing**으로 모든
이미지 작업을 처리했다.

### 5-1. base64 추출 / 삽입
파일 안의 이미지 한 줄이 수십~수백 KB(때로 200KB+)라서 `Read` 툴로 직접 읽으면 토큰 한도
초과 에러가 난다. **절대 Read로 읽으려 하지 말고** 다음 패턴을 쓴다:

```bash
# 추출
awk 'NR==<라인번호>' index.html | sed -e 's/.*base64,//' -e 's/".*//' | base64 -d > out.png

# 교체 (head/tail로 해당 줄만 통째로 교체 — Edit 툴 대신 이 방식이 안전)
head -n <N-1> index.html > index_new.html
printf '      <img src="data:image/jpeg;base64,%s" alt="...">\n' "$(cat new.b64)" >> index_new.html
tail -n +<N+1> index.html >> index_new.html
diff <(awk '{print NR": "length($0)}' index.html) <(awk '{print NR": "length($0)}' index_new.html)  # 해당 줄만 바뀌었는지 검증
mv index_new.html index.html
```

### 5-2. 로고 여백 크롭
```powershell
Add-Type -AssemblyName System.Drawing
$img = [System.Drawing.Bitmap]::FromFile($src)
# 알파>10인 픽셀의 bounding box를 찾아 그 영역만 크롭 (여백 제거)
```
크롭 후 CSS `height`는 "크롭 전 렌더 높이 × (크롭 후 높이/원본 높이)" 비율로 재계산해서
시각적 크기가 유지되도록 한다.

### 5-3. 현장 사진 위 라벨/화살표(주석) 그래픽
사업 현장 사진에 건물명 라벨 박스 + 화살표를 얹은 조감도 이미지가 자주 필요하다.
**가장 좋은 방법은 이 작업을 직접 하지 않는 것**이다:

- 사용자에게 라벨/화살표가 이미 그려진 이미지 파일을 만들어서 저장소 폴더에 직접 넣어달라고
  하면, 다음 세션에서 `git status`의 untracked 파일로 나타난다. 그 파일을 그대로 리사이즈/
  크롭만 해서 쓰면 된다.
- 그런 이미지가 없고 텍스트만 고쳐야 하는 경우(예: 라벨의 영문 코드 접미사 삭제, 오탈자
  수정): **흰색 배경 라벨 박스는 손대기 쉽고, 초록/수풀 배경 위의 화살표선은 지우고 다시
  그리기 까다롭다.**
  - 라벨 박스 텍스트 교체: 박스의 흰색 내부 영역을 flood-fill로 자동 탐지 →
    `FillRectangle`로 흰색 덮어쓰기 → `DrawString`으로 새 텍스트를 중앙 정렬해 다시 그린다.
    폰트는 `Malgun Gothic Bold`가 원본 스타일과 잘 어울렸다.
  - 회전된(세로로 눕힌) 라벨은 `Graphics.TranslateTransform` + `RotateTransform`으로 박스
    중심을 기준으로 회전시킨 좌표계 안에서 동일하게 채우기+텍스트를 그린다. 회전각은 박스의
    axis-aligned bounding box 크기와 실제 폭/높이 관계식(`AABB_w = L·sinθ + W·cosθ` 등)으로
    역산하거나, 크롭 이미지를 보고 코너 좌표를 눈대중으로 재서 구한다.
  - 화살표 경로가 꺾여 있어(직각 elbow) "자연스럽게" 펴 달라는 요청을 받으면: 배경이 수풀
    처럼 텍스처가 있는 영역은 주변의 **깨끗한(주황색 없는) 같은 텍스처 패치를 찾아 그대로
    복제(clone-stamp)해서 기존 화살표 위에 덮어씌운 뒤**, 새 좌표로 직선을 다시 그린다.
    복제할 소스 영역은 반드시 스캔으로 주황색 픽셀이 없는지 사전 확인한다.

### 5-4. 흰 여백 잘라내기
사용자가 제공하는 조감도 이미지 하단에 흰 여백이 남아있는 경우가 잦다. 여러 x좌표에서
아래→위로 스캔해 흰색이 아닌 첫 행(y)을 찾고, 그 경계가 전체 폭에서 일정한지 확인한 뒤
그 높이까지만 크롭한다.

## 6. 문서 섹션 구성 (참고용 목차)

01 사업개요 · 02 사업성과 요약(애니메이션 카운터) · 03 주요 사업성과 · 04 사업 현장 사진
(조감도 + 카테고리별 사진 그리드) · 05 사업계획 대비 변동사항 · 06 변화 스토리 ·
07 사업장 팀장 감사인사 · 08 진행일정(타임라인 표) · 09 결산 · 10 SDGs

## 7. 결산(예산) 표 표기 규칙 (이번 세션에서 확정된 스타일)

- 총액 표시는 **"한화 [금액]원"** 형식으로, "약"이라는 단어는 쓰지 않는다.
- **달러(USD) 금액은 전부 제거**한다 — 표 헤더 컬럼도, 총계 행의 `<span class="usd">`도 삭제.
- 총계 행 라벨은 그냥 **"총계"** — 환율 설명 괄호(`FY25 고정환율 ...`)는 넣지 않는다.
- 표 안의 모든 원화 숫자 뒤에는 **"원"을 직접 붙인다** (예: `63,998,749원`).
- 상단 큰 총액 숫자(`.budget-total .amount`)는 처음엔 40px로 너무 크므로 26px 정도로
  줄인다.

## 8. 알아두면 좋은 버그 / 주의사항

- **`.cnt` 클래스 충돌 버그**: 애니메이션 숫자 카운터(`data-target` 속성 있는 것)와 사진
  개수 배지("01 / 04" 같은 정적 텍스트)가 같은 클래스명(`cnt`)을 쓰면, 스크롤 시
  `IntersectionObserver` + `animate()`가 두 곳 모두에 실행되어 `data-target`이 없는 요소는
  `parseInt(undefined)` → `NaN`으로 텍스트가 깨진다. **애니메이션 카운터 클래스와 정적
  배지 클래스는 반드시 다른 이름을 쓸 것** (예: `cnt` vs `pcnt`).
- 상단바(topbar) 레이아웃: 로고 → 문서 타이틀(`.doc-label`) → `margin-left:auto`가 걸린
  `<nav>`(우측 정렬 탭) 순서로 마크업해야 "타이틀은 로고 옆, 탭은 우측 끝" 배치가 된다.
- git identity(`user.name`/`user.email`)가 전역에 설정되어 있지 않을 수 있으니 리포지토리
  로컬로 설정한다.
- `git commit`/`push` 전에는 반드시 `git status`로 의도치 않은 파일이 섞이지 않았는지 확인.

## 9. 새 보고서 작업 시 체크리스트

1. PPT/원본 자료를 받아 `index.html` 하나로 변환 (인라인 base64, 반응형, 위 브랜드 규칙 적용).
2. 로고는 투명 여백을 크롭한 버전으로 준비.
3. 새 GitHub 저장소 생성 요청 → `git init`부터 진행 → push.
4. Vercel MCP로 `create_git_project` → 자동 배포 확인.
5. 이후 문구/스타일/사진 수정 요청은 `index.html` 편집 → 원본 파일명 동기화 → commit → push
   → (자동배포) → Browser 툴로 실사이트 확인, 순서로 반복.
6. 큰 이미지 base64 줄은 절대 Read로 열지 말고 head/tail 재구성 패턴을 쓸 것.
