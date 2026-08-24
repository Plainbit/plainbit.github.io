# CLAUDE.md — GRABBIT Analyzer(구 FACT_commercial) 배포 하네스

이 저장소(`plainbit.github.io`)는 제품군의 업데이트 메타데이터와 설치 파일을
GitHub Pages / GitHub Release로 서빙합니다.

> **브랜드 변경 (2026-08-24, 이슈 #19).** 분석기 제품명이 `FACT Commercial` → **`GRABBIT Analyzer`** 로 바뀌었다.
> 실행 파일명이 `FACT_commercial.exe` → **`GrabbitAnalyzer.exe`** 로 바뀌고,
> 신규 클라이언트가 보는 메타데이터 경로가 **`GrabbitAnalyzer/version.json`** 으로 바뀌었다
> (`UpdateService.BaseUrl = https://plainbit.github.io/GrabbitAnalyzer`).
> **기존 설치본은 여전히 `FACT_commercial/version.json` 을 폴링한다 — 두 파일을 같은 내용으로 갱신한다.**
> 저장소명(`plainbit.github.io`·`FACT_Expert_Commercial`)과 빌드 폴더 경로는 바뀌지 않았다.
> **이미 배포된 과거 릴리스의 asset 이름(`FACT_commercial.exe`)과 그 downloadUrl 은 손대지 않는다.**

## 배포 트리거 (두 종류)

이 폴더에서 사용자가 말하는 의도에 따라 **서로 다른 두 배포**를 수행합니다.
**대상 exe / 소스 폴더 / 태그 저장소 / version.json 갱신 위치가 모두 다르므로 절대 혼동하지 않는다.**

| 트리거 문구 | 대상 | 소스 exe | 태그 저장소 | version.json 갱신 |
|---|---|---|---|---|
| **"분석기 배포"**, **"메인 배포"**, **"GRABBIT Analyzer 배포"**(구 "FACT_commercial 배포") | 분석기 = 메인 앱 | `GrabbitAnalyzer.exe` | `plainbit.github.io`(공개 asset) + `FACT_Expert_Commercial`(기록) | **top-level 필드 — `GrabbitAnalyzer/`·`FACT_commercial/` 두 version.json 모두** |
| **"수집기 배포"** | 수집기(Collector) | `FACT_Standard_Commercial.exe` | `Plainbit/plainbit.github.io` | **`assets[]` 의 `collector` 항목** |
| **"수집기·분석기 같이 배포"**, **"둘 다 배포"** | 분석기 + 수집기 | 위 두 exe 모두 | 두 저장소 모두 | **top-level + `collector` 모두** |
| **"BitCollector_CLI 배포"** | BitCollector CLI (exe 2종) | `BitCollector_CLI.exe` + `BitCollector_sds_CLI.exe` | `Plainbit/bitCollector_cli` | **갱신 없음** |

> - "분석기" = 메인 앱(`GrabbitAnalyzer`) = A 섹션. "수집기" = B 섹션. "BitCollector_CLI" = D 섹션(독립 배포).
> - 어느 배포인지 헷갈리면 자동 진행하지 말고 사용자에게 확인한다.
> - 함께 배포하는 경우 아래 **C. 동시 배포** 절차를 따른다.

## 공통 규칙

- 배포 버전은 항상 **exe 의 FileVersion** 에서 읽는다.
  - PowerShell: `(Get-Item $exe).VersionInfo.FileVersion`
- `sha256`/`size` 는 반드시 해당 폴더의 **`version_info.json`** 에서 읽는다. 임의로 계산·추정하지 않는다.
  ```json
  { "sha256": "...", "size": 449655752 }
  ```
  > 검증: `version_info.json.size` 는 실제 exe 파일 크기와 일치해야 한다.
  > 불일치하면 배포를 중단하고 사용자에게 알린다 (빌드 stale 또는 version_info.json 미갱신).
- 되돌리기 어려운 외부 작업(GitHub 릴리스 생성, push) **실행 전** 최종 요약(버전, sha256, size, changelog)을 보여주고 확인을 받는다.
- 파일 삭제/덮어쓰기 금지. 기존 버전 파일·릴리스는 보존한다.
- 버전 불일치나 이미 존재하는 태그 발견 시 자동 진행하지 말고 사용자에게 보고한다.

---

# A. 메인 앱 배포 ("GRABBIT Analyzer 배포")

## 입력 소스
빌드 폴더:
`C:\Project\FACT_commercial\FACT_commercial\bin\Release\net9.0\win-x64\publish`
- `GrabbitAnalyzer.exe` — FileVersion 을 배포 버전 `<VER>` 로 사용
  (빌드 폴더 경로는 그대로다 — 프로젝트 폴더명은 리네임하지 않았다)
- `version_info.json` — `sha256`, `size`

## 절차 (`<VER>` 예: `1.1.2.1`)

> **배포 방식 변경 (1.1.2.2 이후 버전부터 적용):**
> 분석기 설치 파일은 더 이상 `Download/` 폴더에 커밋하지 않는다.
> `plainbit.github.io`(public) 의 **GitHub Release asset** 으로 서빙한다 (수집기와 동일 방식).
> `downloadUrl` 도 릴리스 asset 을 가리킨다.
> (기존 `Download/` 의 과거 버전 파일과 그 다운로드 URL 은 그대로 보존한다.)

### 0. 사전 확인
- `publish\GrabbitAnalyzer.exe` FileVersion = `<VER>` 확인.
- `version.json` top-level `version` 과 비교 (같으면 재배포 여부 확인).
- 이미 `<VER>` 태그 존재 여부 확인 (두 저장소 모두):
  - `gh api repos/Plainbit/FACT_Expert_Commercial/tags --jq '.[].name'`
  - `gh api repos/Plainbit/plainbit.github.io/tags --jq '.[].name'`
- **changelog 문구를 사용자에게 물어본다** (자동 생성 금지).

### 1. version.json top-level 갱신 — **두 경로 모두**

`GrabbitAnalyzer/version.json`(신규 클라이언트)과 `FACT_commercial/version.json`(기존 설치본)을 **같은 내용으로** 갱신한다.
한쪽만 고치면 그쪽 클라이언트만 업데이트를 받는다.
- `version` = `<VER>`
- `releaseDate` = 오늘 (`YYYY-MM-DD`)
- `downloadUrl` = `https://github.com/Plainbit/plainbit.github.io/releases/download/<VER>/GrabbitAnalyzer.exe`
- `sha256` / `size` = `version_info.json` 값
- `changelog` = 사용자가 준 문구
- `assets[]`, `licenseKey` 등 나머지는 그대로 유지.

### 2. GitHub 릴리스 — 공개 서빙 asset (`Plainbit/plainbit.github.io`)
- 태그명 = 릴리스명 = `<VER>`, 본문 없음. asset = `GrabbitAnalyzer.exe`.
```bash
gh release create <VER> \
  -R Plainbit/plainbit.github.io \
  --title "<VER>" \
  --notes "" \
  --target master \
  "C:/Project/FACT_commercial/FACT_commercial/bin/Release/net9.0/win-x64/publish/GrabbitAnalyzer.exe"
```
> 여기 asset 이 `downloadUrl` 의 실제 다운로드 대상이다.
> `Download/` 폴더에는 복사하지 않는다.

### 3. GitHub 릴리스 — 제품 저장소 기록/변경이력 (`Plainbit/FACT_Expert_Commercial`)
- 기존과 동일하게 유지 (태그 + changelog compare 링크, exe asset).
- `<PREV_VER>` = 직전 릴리스 태그 (`gh release list -R Plainbit/FACT_Expert_Commercial -L 1`)
```bash
gh release create <VER> \
  -R Plainbit/FACT_Expert_Commercial \
  --title "<VER> 배포" \
  --notes "**Full Changelog**: https://github.com/Plainbit/FACT_Expert_Commercial/compare/<PREV_VER>...<VER>" \
  "C:/Project/FACT_commercial/FACT_commercial/bin/Release/net9.0/win-x64/publish/GrabbitAnalyzer.exe"
```

### 4. 커밋 & 푸시 (`plainbit.github.io`)
- 스테이징: `GrabbitAnalyzer/version.json` + `FACT_commercial/version.json` **만** (Download exe 없음).
- 커밋 메시지: `<VER> 배포`
- `git push origin master`

---

# B. 수집기 배포 ("수집기 배포")

## 입력 소스
빌드 폴더:
`C:\Project\FACT_Standard-CSharp-version-\src\FACT_Standard_Commercial_V_CShap\bin\Release\net10.0-windows\win-x64\publish`
- `FACT_Standard_Commercial.exe` — FileVersion 을 배포 버전 `<VER>` 로 사용
- `version_info.json` — `sha256`, `size`

> 메인 앱과 달리 `Download/` 폴더에 복사하지 않는다.
> 수집기는 `plainbit.github.io` 의 **GitHub Release asset** 으로 서빙된다.

## 절차 (`<VER>` 예: `0.0.0.5`)

### 0. 사전 확인
- `publish\FACT_Standard_Commercial.exe` FileVersion = `<VER>` 확인.
- `version.json` 의 `assets[]` 중 `id == "collector"` 항목의 현재 `version` 과 비교 (같으면 재배포 여부 확인).
- 이미 `<VER>` 태그 존재 여부 확인:
  `gh api repos/Plainbit/plainbit.github.io/tags --jq '.[].name'`

### 1. GitHub 태그 + 릴리스 (`Plainbit/plainbit.github.io`)
- 태그명 = 릴리스명 = `<VER>`, 본문 없음 (기존 `0.0.0.4` 릴리스 규칙과 동일)
```bash
gh release create <VER> \
  -R Plainbit/plainbit.github.io \
  --title "<VER>" \
  --notes "" \
  "C:/Project/FACT_Standard-CSharp-version-/src/FACT_Standard_Commercial_V_CShap/bin/Release/net10.0-windows/win-x64/publish/FACT_Standard_Commercial.exe"
```
> `gh release create` 가 로컬에서 태그를 만들려면 대상 저장소에 대응하는 커밋이 필요할 수 있다.
> 문제가 생기면 `--target master` 를 붙인다.

### 2. version.json 의 `collector` 자산 갱신 — **두 경로 모두**
`GrabbitAnalyzer/version.json` 과 `FACT_commercial/version.json` 에서
`assets[]` 중 `id == "collector"` 항목만 수정 (top-level 및 다른 자산은 그대로):

> 수집기 exe(`FACT_Standard_Commercial.exe`)는 **이번 리네임 대상이 아니다.** 이슈 #19 의
> `FACT_standard → GRABBIT Collector` 는 아직 반영되지 않았다 — 파일명을 임의로 바꾸지 마라.
- `version` = `<VER>`
- `downloadUrl` = `https://github.com/Plainbit/plainbit.github.io/releases/download/<VER>/FACT_Standard_Commercial.exe`
- `sha256` / `size` = `version_info.json` 값

### 3. 커밋 & 푸시 (`plainbit.github.io`)
- 스테이징: `GrabbitAnalyzer/version.json` + `FACT_commercial/version.json`
- 커밋 메시지: `수집기 <VER> 배포`
- `git push origin master`

---

# C. 동시 배포 (수집기 + 분석기 함께)

수집기와 분석기가 같은 시점에 함께 배포될 수 있다. 이때는 A·B 를 각각 수행하되,
**`version.json` 을 한 번에 갱신하고 커밋도 한 번만** 하도록 아래 순서로 진행한다.

### 0. 사전 확인 (둘 다)
- 각 exe 의 FileVersion 확인 → 분석기 `<VER_A>`, 수집기 `<VER_B>`.
- 각 `version_info.json` 의 `size` 가 실제 exe 크기와 일치하는지 확인.
- 두 저장소에 각 태그가 이미 있는지 확인.
- **분석기 changelog 문구를 사용자에게 물어본다** (수집기는 changelog 없음).
- 최종 요약(두 버전, 각 sha256/size, changelog)을 보여주고 확인을 받는다.

### 1. GitHub 릴리스 생성 (분석기 2건 + 수집기 1건)
- 분석기 공개 asset: A-2 대로 `plainbit.github.io` 에 `<VER_A>` 릴리스 + `GrabbitAnalyzer.exe`.
- 분석기 제품 기록: A-3 대로 `FACT_Expert_Commercial` 에 `<VER_A>` 릴리스 + `GrabbitAnalyzer.exe`.
- 수집기: B-1 대로 `plainbit.github.io` 에 `<VER_B>` 릴리스 + `FACT_Standard_Commercial.exe`.

### 2. version.json 한 번에 갱신 — **두 경로 모두**
- top-level: A-1 대로 (`version`=`<VER_A>`, `releaseDate`, `downloadUrl`(공개 asset), `sha256`, `size`, `changelog`).
- `assets[].collector`: B-2 대로 (`version`=`<VER_B>`, `downloadUrl`, `sha256`, `size`).
- (분석기 설치 파일은 `Download/` 에 복사하지 않는다.)

### 3. 커밋 & 푸시 (한 번)
- 스테이징: `GrabbitAnalyzer/version.json` + `FACT_commercial/version.json` **만**.
- 커밋 메시지: `분석기 <VER_A> · 수집기 <VER_B> 배포`
- `git push origin master`

---

# D. BitCollector_CLI 배포 ("BitCollector_CLI 배포")

**독립 배포다.** version.json(두 경로 모두) 을 갱신하지 않고, `plainbit.github.io` 로의
커밋/푸시도 없다. `Plainbit/bitCollector_cli`(private) 저장소에 **새 릴리스만 생성**하고
exe 2종을 업로드한다.

## 입력 소스 (exe 2종)
- `C:\Project\bitCollector_cli\BitCollector_CLI\x64\Release\BitCollector_CLI.exe`
- `C:\Project\bitCollector_cli\BitCollector_CLI\x64\Release_SDS\BitCollector_sds_CLI.exe`

> **공통 규칙 예외:**
> - 이 exe 들의 FileVersion 은 `1.0.0.0` 으로 고정되어 있어 **배포 버전으로 쓸 수 없다.**
>   배포 버전 `<VER>` 은 **사용자에게 직접 물어본다.**
> - 이 폴더에는 `version_info.json` 이 없다 → `sha256`/`size` 는 다루지 않는다.

## 절차 (`<VER>` 예: `1.0.1.14`)

### 0. 사전 확인
- exe 2종이 각 경로에 존재하는지 확인.
- **배포할 버전 `<VER>` 를 사용자에게 물어본다.**
- 이미 `<VER>` 태그가 있는지 확인: `gh api repos/Plainbit/bitCollector_cli/tags --jq '.[].name'`.
- `<PREV_VER>` = 직전 릴리스 태그 (`gh release list -R Plainbit/bitCollector_cli -L 1`).

### 1. GitHub 태그 + 릴리스 (`Plainbit/bitCollector_cli`)
- 태그명 = `<VER>`, 릴리스명 = `<VER> 배포`, 기본 브랜치 `main`.
- 본문 = compare 링크. asset = **exe 2종 모두**.
```bash
gh release create <VER> \
  -R Plainbit/bitCollector_cli \
  --title "<VER> 배포" \
  --notes "**Full Changelog**: https://github.com/Plainbit/bitCollector_cli/compare/<PREV_VER>...<VER>" \
  "C:/Project/bitCollector_cli/BitCollector_CLI/x64/Release/BitCollector_CLI.exe" \
  "C:/Project/bitCollector_cli/BitCollector_CLI/x64/Release_SDS/BitCollector_sds_CLI.exe"
```
> 태그가 로컬에서 안 만들어지면 `--target main` 을 붙인다.

### 2. 검증
- `gh release view <VER> -R Plainbit/bitCollector_cli --json assets` 로 두 asset 업로드 확인.
- (`plainbit.github.io` 커밋/푸시 없음, version.json 변경 없음.)

---

## 참고 경로 / 저장소
- 메인 빌드: `C:\Project\FACT_commercial\FACT_commercial\bin\Release\net9.0\win-x64\publish`
- 수집기 빌드: `C:\Project\FACT_Standard-CSharp-version-\src\FACT_Standard_Commercial_V_CShap\bin\Release\net10.0-windows\win-x64\publish`
- 메타데이터: `GrabbitAnalyzer/version.json`(신규 클라이언트) + `FACT_commercial/version.json`(기존 설치본) — **둘을 같은 내용으로 유지**
- 메인 설치 파일: 신규 버전은 `plainbit.github.io` 릴리스 asset (`Download/` 는 1.1.2.2 이하 과거 버전 보관용)
- 메인 태그/릴리스: `https://github.com/Plainbit/plainbit.github.io`(공개 서빙 asset) + `https://github.com/Plainbit/FACT_Expert_Commercial`(제품 기록/changelog)
- 수집기 태그/릴리스 & Pages(origin): `https://github.com/Plainbit/plainbit.github.io`
- BitCollector_CLI 빌드: `...\bitCollector_cli\BitCollector_CLI\x64\Release`(BitCollector_CLI.exe) + `...\x64\Release_SDS`(BitCollector_sds_CLI.exe)
- BitCollector_CLI 태그/릴리스: `https://github.com/Plainbit/bitCollector_cli` (private, version.json 갱신 없음)
