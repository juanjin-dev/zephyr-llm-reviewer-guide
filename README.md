# Zephyr 로컬 리뷰 워크플로 — AI 리뷰어용

이 문서는 AI 코딩 에이전트(Claude/DeepSeek 등)를 **Zephyr PR 리뷰어**로 쓰기 위한 로컬
워크플로를 정리합니다. 대상 트리는 `zephyrproject-rtos/zephyr`이고, 아래 모든 경로는
Zephyr 저장소 루트 기준입니다.

---

## 0. 역할과 원칙

AI는 **리뷰 초안을 생성**하고, **사람이 검증·책임**을 집니다. `doc/contribute/reviewer_expectations.rst`
의 관련 조항:

- raw LLM 출력을 그대로 PR 코멘트로 게시하지 말 것. 모든 AI 피드백은 사람이 검증하고
  문맥을 맞춘 뒤 올립니다.
- 리뷰는 **가능하면 한 번에 전체**를. 부분 리뷰는 어느 부분을 봤는지 명시합니다.
- trivial/non-functional 요청은 **Non-blocking**으로 표시하고, non-blocking만 남으면
  approve할 수 있습니다.
- **스타일 지적은 반드시 해당 가이드라인/규칙을 인용**해야 합니다. 문서화되지 않은
  스타일 선호는 non-blocking으로만 지적할 수 있고 request changes 사유가 될 수 없습니다.
- **CI가 자동으로 잡는 것**(checkpatch, gitlint, compliance, CI 빌드)은 request changes
  사유로 쓰지 않습니다.
- PR 범위를 넓히는 신기능 요청을 하지 않습니다. RFC에 제안으로 남깁니다.
- PR이 **건드리지 않은 줄의 스타일**은 지적하지 않습니다.

---

## 1. 환경 준비

Zephyr는 west 워크스페이스 안에서만 빌드됩니다.

```sh
# 저장소 부모에서
west init -l <zephyr-dir>
west update
```

원격 구성:

```sh
git remote -v   # upstream = zephyrproject-rtos/zephyr, origin = 본인 포크
```

리뷰 도구 준비:

```sh
pip install -r scripts/requirements-compliance.txt
gh auth login        # PR/리뷰 조회·등록용
```

`git worktree`에서 작업하면 `export ZEPHYR_BASE=<worktree>`가 필요합니다. 그렇지 않으면
워크스페이스에 등록된 체크아웃이 조용히 빌드됩니다.

---

## 2. PR 재현 — 리뷰의 출발점

**CI는 PR을 현재 `main`에 리베이스한 상태로 돕니다.** 로컬에서도 동일하게 재현해야
"main에 있다/없다"를 정확히 말할 수 있습니다.

```sh
# 두 번 fetch: multi-ref fetch 후 FETCH_HEAD는 첫 번째 ref입니다
git fetch upstream main
git fetch upstream pull/<N>/head
git checkout --detach FETCH_HEAD
git rebase upstream/main
```

또는 간단히:

```sh
gh pr checkout <N>
git rebase upstream/main
```

`main`의 현재 상태가 궁금하면:

```sh
git fetch upstream main
git log upstream/main --oneline -10
```

---

## 3. 도구 레퍼런스

### 3.1 gh — PR 조회/리뷰

```sh
gh pr view <N>                                  # 제목·본문·상태·체크
gh pr diff <N>                                  # 통합 diff
gh pr checks <N>                                # CI 상태
gh pr list --author <user> --state open         # 작성자별 PR 목록

# 인라인 리뷰 코멘트 목록
gh api repos/zephyrproject-rtos/zephyr/pulls/<N>/comments

# 일반(이슈) 코멘트 목록
gh api repos/zephyrproject-rtos/zephyr/issues/<N>/comments

# 리뷰 등록 — 사람이 검증한 뒤에만
gh pr review <N> --comment --body-file review.md
gh pr review <N> --request-changes --body-file review.md
```

코멘트 본문은 파일로 작성해 넘깁니다(`--body-file`). 셸 인용 문제를 피하고, 사람이 파일을
한 번 더 읽고 수정할 수 있게 합니다.

### 3.2 로컬 검사

```sh
# checkpatch — 반드시 format-patch로. git show를 파이프하면 BAD_SIGN_OFF 오탐 발생
git format-patch -1 --stdout <sha> | ./scripts/checkpatch.pl -

# 커밋 범위 전체 compliance (원격 main 기준)
./scripts/ci/check_compliance.py --parallel -c upstream/main..HEAD

# 사용 가능한 체크 목록
./scripts/ci/check_compliance.py -l

# 단일 체크
./scripts/ci/check_compliance.py -m Checkpatch -c upstream/main..HEAD
```

`-l`이 출력하는 체크: BinaryFiles, BoardYml, CMakeStyle, Checkpatch, ClangFormat,
DeviceAPI, DevicetreeBindings, DevicetreeLinting, Gitlint, Kconfig, KconfigBasic,
KeepSorted, LicenseAndCopyrightCheck, MaintainersFormat, Nits, Ruff, SphinxLint,
TestMetadataFilename, YAMLLint 등. **이들이 잡는 것은 사람 리뷰에서 다시 요구하지
않습니다.**

### 3.3 메인테이너 확인

```sh
./scripts/get_maintainer.py path <files...>
./scripts/get_maintainer.py commits <base>..<head>
./scripts/get_maintainer.py areas
```

리뷰어가 맞는지, 리뷰 대상이 어느 area인지 확인하는 데 씁니다.

### 3.4 twister — 테스트/샘플 실행

```sh
west twister -p native_sim -T <test-dir>       # 디렉터리 전체
west twister -s <test-dir>/<scenario-id>       # 단일 시나리오
west twister -p native_sim -T <dir> --build-only
west twister -i                               # 실패 로그 출력
```

`tests.yaml`의 platform이 `unit_testing`인 스위트는 `-p unit_testing`으로 돕니다.

---

## 4. 리뷰 체크리스트

### 4.1 커밋 메시지·트레일러

- `area:` 프리픽스. **파일의 최근 이력으로 확인**:
  `git log --format=%s -20 -- <path>`. 예: `Bluetooth: Host:`, `drivers: i2c: nrfx:`,
  `boards: nordic:`, `kernel:`, `doc:`. `subsys:`, `treewide:`, `WIP` 금지.
- 제목 75컬럼 이하, 끝에 마침표 없음, 명령형.
- **본문 필수(비어 있으면 안 됨)**, 75컬럼 래핑. 무엇/왜/어떤 가정/어떻게 테스트.
- `Fixes #N`는 트레일러 블록 바로 앞의 **별도 문단**.
- 트레일러는 마지막 문단에 한 블록: `Assisted-by`, `Signed-off-by`, `Link:`,
  `(cherry picked from commit ...)`.
- `Signed-off-by`가 **Author 이름/이메일과 정확히 일치**하는지:
  `git log -1 --format='%an <%ae>%n%b'`.
- `Co-authored-by` 금지. `Assisted-by`는 정확히 하나.
- 머지 커밋·fixup/squash 커밋 금지. `git rebase upstream/main`으로 갱신.
- 커밋 하나가 논리적으로 한 건. 시리즈의 각 커밋이 각자 빌드·테스트돼야 합니다.

### 4.2 C 코드 스타일 (`doc/contribute/style/code.rst`)

- 8칸 탭 들여쓰기, 100컬럼 줄 길이.
- 모든 `if`/`else`/루프 본문에 중괄호.
- `/* */` 주석만, `//` 금지.
- `snake_case`. 이진 리터럴 금지. 이모지 금지.
- **명시적 비교**: `if (err != 0)`, `if (ptr == NULL)`, `if (len > 0)`.
  `if (!err)`(정수) 금지.
- 고정폭 타입: `uint8_t`/`int32_t`, 크기는 `size_t`. bare `int`/`short`/`long` 금지
  (Dir 4.6). `char`는 텍스트에만.
- unsigned 상수에 `U` 접미사. 8진 상수 금지.
- 주석 처리된 코드 금지.
- 에러를 담는 반환값은 전부 확인.
- 모든 헤더에 include guard.

### 4.3 MISRA 서브셋 (`doc/contribute/coding_guidelines/index.rst`)

- **객체 스코프 최소화** (Rule 8.9, CERT DCL19-C): 함수 하나만 참조하면 그 함수에,
  `static` 로컬로. 한 브랜치만 참조하면 그 브랜치에.
- **동적 할당**: Dir 4.12·Rule 21.3(`<stdlib.h>` 할당자)는 금지가 기본. Rule A.4가
  커널/서브시스템의 libc 할당자 허용 범위를 정합니다. 서브시스템 관례를 따르고,
  정적 풀로 충분한 곳에 힙을 추가하지 않습니다.
- **Rule A.1**: `#ifdef CONFIG_FOO`보다 `if (IS_ENABLED(CONFIG_FOO))`. 양쪽 분기가 모두
  컴파일돼야 하므로 헤더 선언은 unconditional.
- **Rule A.2**: inclusive language.
- **Rule 85**: 제어식은 본질적으로 Boolean.

### 4.4 Devicetree (`.dts`/`.overlay`)

- `compatible`은 `<vendor>,<device>`. vendor는 `dts/bindings/vendor-prefixes.txt`에 등재.
  generic 하드웨어(`gpio-leds` 등)는 접두사 없음.
- **새 compatible마다 `dts/bindings/`에 바인딩**이 있어야 합니다.
- `zephyr-keep-sorted` start/stop 블록은 정렬 유지.
- 슬롯/핀 매핑은 `gpio-map`/`gpio-nexus`(`wisblock_sensor_*` 같은 노드)와 대조해서 검증.
- **선례는 기억이 아니라 트리에서 세어서** 확인합니다. "그런 선례가 있었나"는
  `grep`/카운트 + `git log --diff-filter=A`(최근 이력이 전체 카운트와 같은 방향인지)로
  답합니다. 스타일 문서와 실무가 다르면 둘 다 숫자로 제시합니다.

### 4.5 DT 바인딩 (`.yaml`)

- 구조: `description`, `compatible`, `include`(`base.yaml`, `uart-device.yaml` 등),
  `properties`(각 속성에 `type`/`required`/`description`), 필요한 경우 `child-binding`,
  `examples`.
- 추가한 속성이 실제 드라이버에서 읽히는지 확인합니다.
- `power-domains`는 `base.yaml`에 정의된 표준 속성이라 별도 선언이 필요 없습니다.
  커스텀 속성(`touch-gpios` 등)은 반드시 해당 바인딩에 선언해야 합니다.
- 예시는 실제로 빌드되는 노드 형태로 작성합니다.

### 4.6 Kconfig (`doc/contribute/style/kconfig.rst`, `doc/build/kconfig/tips.rst`)

- 심볼 네이밍·`menuconfig`/`if` 구조.
- `select` vs `depends on` 구분 (tips.rst).
- **코드/문서에서 참조하는 모든 `CONFIG_` 심볼은 Kconfig에 정의**돼야 합니다(KconfigBasic).
- 실드 `Kconfig.defconfig`는 `if SHIELD_<NAME>` 가드 + `def_bool
  $(shields_list_contains,<name>)` 패턴(Kconfig.shield).
- **기존 심볼의 기본값만 바꿀 때는 `configdefault`** (`doc/build/kconfig/extensions.rst`).
  `configdefault FOO`는 `default` 필드만 가질 수 있고, `config FOO`로 재정의하면 그
  심볼의 `depends on`이 약화됩니다. 트리에서 240개 보드 파일이 `configdefault`를 씁니다.
- `default y`는 `depends on DT_HAS_..._ENABLED`와 짝지어야 드라이버가 불필요하게 켜지지
  않습니다.

### 4.7 CMake (`doc/contribute/style/cmake.rst`)

- **건드린 줄만 검사**됩니다. 옛 파일은 grandfather 처리되므로, 옛 `CMakeLists.txt`를
  템플릿으로 복사하면 CI가 실패할 수 있습니다.

### 4.8 문서 (`doc/contribute/documentation/guidelines.rst`)

- reStructuredText, 미국 영어, 100컬럼(URL 제외).
- Doxygen(`doc/contribute/style/doxygen.rst`): 공개 함수는 brief, `@param`,
  **각 반환값마다 `@retval <value> <text>`** (성공 먼저, 같은 값이 여러 조건에서 나오면 반복).
- API 변경(신규·폐기·제거)은 릴리스노트·마이그레이션 가이드에. `VERSION`이
  `X.(Y-1).99`면 다음 릴리스는 `X.Y` → `doc/releases/release-notes-X.Y.rst`,
  `doc/releases/migration-guide-X.Y.rst`.

### 4.9 테스트·샘플 메타데이터

- 메타데이터 파일명은 `tests.yaml`. `sample.yaml`/`testcase.yaml`은 레거시(신규 파일 CI
  거부).
- twister가 직접 실행하는 셸 스크립트(bsim `tests_scripts/*.sh` 등)는 모드 `100755`.
- `SHIELD=` 조합은 `tests.yaml`에 한 줄씩. `SHIELD="base;module"`처럼 여러 실드를
  세미콜론으로 **겹쳐 쌓는 것이 정상**입니다(트리 15건 — 베이스보드 + 모듈 실드 등).
  단일 시나리오 슬롯(display sample 등)은 실드 하나씩이므로, 그런 경우에만 조합을 코어
  모듈별로 분산합니다.

### 4.10 작성자 주장의 근거 확인

- **드라이버 리뷰는 데이터시트/매뉴얼을 사용자에게 요구합니다.** 레지스터 의미·타이밍·
  초기화 순서·극성·전원 시퀀스 같은 하드웨어 동작은 데이터시트 없이 검증할 수 없으므로,
  리뷰 시작 전에 작성자에게 데이터시트/매뉴얼 URL 또는 파일을 요청합니다.
- 데이터시트 값(극성·타이밍·주소·부트 시간), "~가 표준이다" 같은 관례 주장이
  커밋 메시지·코드 주석·문서에 **근거 없이 단정**돼 있으면 출처를 묻습니다.
- 리뷰어 자신의 지적도 같은 기준입니다 — 단정 전에 트리/문서로 확인합니다(§9.1).

---

## 5. 지적하지 말 것 (요약)

- CI 자동 검출(checkpatch, gitlint, compliance, CI 빌드/테스트 실패).
- PR 범위 밖 신기능·대규모 정리 요청.
- PR이 건드리지 않은 줄의 스타일.
- tree-wide 정리.
- 문서화되지 않은 스타일 선호(→ non-blocking으로만).
- **"CI가 잡을 것을 지적하지 않는다"와 "리뷰어가 직접 빌드한다"는 양립합니다.** 빌드는
  다른 지적의 전제이지, 빌드 실패 자체를 코멘트로 반복하지 않는다는 뜻입니다.

---

## 6. 보고 형식

구체적이고 검증 가능한 문제만, 규칙과 근거를 함께 보고합니다. 버그는 그것을 트리거하는
입력을 함께 적습니다.

```markdown
| # | 심각도 | 파일:라인 | 문제 | 규칙/근거 |
|---|--------|-----------|------|-----------|
| 1 | HIGH   | drivers/foo/bar.c:42 | 반환값 무시 | code.rst "에러 반환값 확인" |
| 2 | LOW    | dts/bindings/...      | vendor 미등재 | vendor-prefixes.txt |
```

심각도 예: CRITICAL(보안/데이터 손상), HIGH(기능 오동작), MEDIUM(경계 조건·호환성),
LOW(문서화된 스타일 위반).

---

## 7. 리뷰 등록 절차 (사람 검증 필수)

1. AI가 위 체크리스트로 초안을 생성합니다(심각도 + 파일:라인 + 규칙 인용).
2. 사람이 각 항목을 확인하고, 근거가 틀린 항목·톤이 어색한 항목을 고칩니다.
3. 검증된 내용을 `review.md`로 저장하고 `gh pr review <N> --comment --body-file review.md`
   또는 `--request-changes`로 등록합니다.
4. **raw AI 출력을 그대로 붙여넣지 않습니다.**
5. **작성자가 반박하면 빠르게 접습니다.** 반박이 맞으면 코멘트를 철회/수정하고 approve로
   전환합니다. 리뷰어가 틀렸을 때 빨리 접는 것도 관례이며, reviewer_expectations.rst의
   "advance the PR to a mergeable state" 원칙에 부합합니다. `gh api
   repos/zephyrproject-rtos/zephyr/pulls/<N>/comments`로 내 코멘트를 찾고, 정정 코멘트나
   `--comment` 리뷰로 수정 의사를 남깁니다.

---

## 8. 리뷰 루프 예시

```sh
# 1) PR 재현
git fetch upstream main
git fetch upstream pull/1234/head
git checkout --detach FETCH_HEAD
git rebase upstream/main

# 2) 대상 파악
gh pr view 1234
gh pr diff 1234 > /tmp/pr.diff
./scripts/get_maintainer.py commits upstream/main..HEAD

# 3) 빌드 (다른 지적의 전제 — CI보다 먼저 잡기)
west build -b <board> <app-or-test-dir>        # 필요한 경우 -DSHIELD="base;module"
west twister -p native_sim -T <dir> --build-only

# 4) 기계 검사(CI 중복 지적 회피 확인)
git format-patch -1 --stdout HEAD | ./scripts/checkpatch.pl -
./scripts/ci/check_compliance.py --parallel -c upstream/main..HEAD

# 5) 사람이 채울 리뷰 초안 생성 → 검증 → 등록
```

---

## 부록: 리뷰어가 자주 틀리는 것 (빠른 참조)

- `if (!err)` → `if (err != 0)`.
- bare `int` → `uint8_t`/`int32_t`/`size_t`.
- `//` → `/* */`.
- 파일 스코프 전역 → 함수 내 `static` 로컬.
- `testcase.yaml`/`sample.yaml` → `tests.yaml`.
- 인용 없는 스타일 지적 → 해당 스타일 문서 `file:line` 인용.
- CI가 잡을 것(체크패치/깃린트/포맷) → 지적 금지.

---

## 9. 실측 리뷰 관례 (gh 조사)

2026-10-21에 `gh pr list`/`gh api pulls/N/comments`로 최근 머지된 PR의 **인라인 리뷰
코멘트 1,558개**(BSP 추가 100여 개 포함)를 수집했습니다. 모든 PR 번호는
`zephyrproject-rtos/zephyr`의 PR입니다.

```
area: Boards              395
area: Bluetooth Host      383
area: Devicetree Binding  224
area: Power Management    217
area: Sensor Subsystem    186
area: Sensors              65
area: Boards/SoCs        (추가 수집)
area: Kernel               54
area: Networking           34
```

주요 리뷰어: `jhedberg`·`Thalley`·`anchao`(BT Host), `kartben`·`sylvioalves`·
`nordicjm`·`erwango`(Boards/SoC), `marekmatej`(Devicetree Binding), `rlubos`·`jukkar`
(Networking), `mathieuchopstm`(PM), `nashif`·`maass-hamburg`(Kernel), `jfischer-no`
(USB/CDC), AI 리뷰(`Copilot`) 다수.

### 9.1 보편적 관례 (전 영역 공통)

1. **근거 기반 리뷰 — 직접 재현하고 명령·결과를 코멘트에 붙입니다.**
   [#122616](https://github.com/zephyrproject-rtos/zephyr/pull/122616): jhedberg —
   "Measured with this test on qemu_cortex_a53 and qemu_cortex_m3: arm64 needs
   2.7-2.9x..."
   [#122449](https://github.com/zephyrproject-rtos/zephyr/pull/122449): "the suite
   passes without this sleep (native_sim with 24 shuffled orders...)".
2. **규칙 인용.** [#122537](https://github.com/zephyrproject-rtos/zephyr/pull/122537):
   Thalley — "Requirement by our coding guidelines; any function that returns a value
   shall either use that value or cast it to void."
3. **정확한 수정 제시** — `` ```suggestion `` 블록(표본 146건 중 22건).
4. **저자 응답 관례** — "Done", "done, thanks", "Fixed". 모든 코멘트에 응답.
5. **논블로킹 마킹** — "Nit:", "Minor:", "Non-blocking and fine as a follow-up PR"
   ([#122134](https://github.com/zephyrproject-rtos/zephyr/pull/122134)).
6. **마법 숫자 근거 요구** — [#122537](https://github.com/zephyrproject-rtos/zephyr/pull/122537):
   "Please elaborate on why the pool sizes are 1 and 4".
7. **죽은 코드 제거 요구** — [#122449](https://github.com/zephyrproject-rtos/zephyr/pull/122449):
   "No test checks what this option adds... drop it or test it."
8. **스코프 최소화**(MISRA 8.9) — [#122606](https://github.com/zephyrproject-rtos/zephyr/pull/122606):
   "this could be moved inside the `if`".
9. **에러 처리 철학** — unrecoverable → `assert`/`LOG_ERR`, cleanup 경로에서 원래
   `err` 보존, 버려지는 반환값은 `(void)` 캐스트.
10. **크로스레퍼런스** — 방금 머지된 PR·스펙 섹션을 링크로 가리킵니다.

### 9.2 BSP (Boards/SoCs/Devicetree) — 가장 많은 지적 영역

보드·SoC·dts 파일에서 반복되는 지적 유형입니다.

1. **타 보드와 일관성** [#106210](https://github.com/zephyrproject-rtos/zephyr/pull/106210):
   "Most boards have `CONFIG_CLOCK_CONTROL=y` here".
2. **복붙 전파 오류** [#106210](https://github.com/zephyrproject-rtos/zephyr/pull/106210):
   kartben — "this has spread through copy-paste in a bunch of boards".
3. **compatible 오타** [#106210](https://github.com/zephyrproject-rtos/zephyr/pull/106210):
   "It is `pcf8563` not `pfc8563`".
4. **DT `ranges` 누락** [#107255](https://github.com/zephyrproject-rtos/zephyr/pull/107255):
   nordicjm — "missing `ranges`" / "ranges should always have been there".
5. **I2C `sda-gpios`/`scl-gpios` 누락** [#106183](https://github.com/zephyrproject-rtos/zephyr/pull/106183).
6. **board yaml 태그** [#106183](https://github.com/zephyrproject-rtos/zephyr/pull/106183):
   "please add `netif:wifi`"(→ [#105537](https://github.com/zephyrproject-rtos/zephyr/pull/105537)),
   "Those ignore tags could go".
7. **doc 사양 정확성** [#106183](https://github.com/zephyrproject-rtos/zephyr/pull/106183):
   "PSRAM on this board is 8MB".
8. **`chosen` rtc 등 표준 노드 활용** [#106210](https://github.com/zephyrproject-rtos/zephyr/pull/106210).
9. **방금 머지된 변경 반영** [#106183](https://github.com/zephyrproject-rtos/zephyr/pull/106183):
   "just updated and merged in main in #105768".
10. **SoC 공용 dtsi 재사용** [#106183](https://github.com/zephyrproject-rtos/zephyr/pull/106183):
    "Would you add esp32c5_wroom_n8r8.dtsi as well?".
11. **defconfig 불필요 항목 제거** [#106210](https://github.com/zephyrproject-rtos/zephyr/pull/106210).
12. **보드 포팅 철학** [#113140](https://github.com/zephyrproject-rtos/zephyr/pull/113140):
    jfischer-no — "Must follow .../board_porting.html#general-recommendations";
    "Subsystems must remain **disabled by default**"(board_porting.rst).
13. **매직 넘버 → `#define`/주석** [#122615](https://github.com/zephyrproject-rtos/zephyr/pull/122615):
    "have this magic number as a #define or at least comment".
14. **`#if` 조건 정확성** [#122506](https://github.com/zephyrproject-rtos/zephyr/pull/122506):
    "`> 0` drops uart0... needs `>= 0`"; 전처리 `#else` 페어링 버그 지적.
15. **SoC 전원/타이밍 추론** [#122488](https://github.com/zephyrproject-rtos/zephyr/pull/122488):
    "the list walk and esp_timer_get_time() sit between the counter read...".
16. **파일명 snake_case** [#122401](https://github.com/zephyrproject-rtos/zephyr/pull/122401):
    nordicjm — "file names should be lower case... zephyr is snake case not camel case"
    (non-blocking "not listing it as a blocker").
17. **dts `clocks` 속성 명시** [#122189](https://github.com/zephyrproject-rtos/zephyr/pull/122189):
    "add clocks property here... it works now because HSI48 is enabled by default".
18. **doc 이미지 링크로 대체·용량 축소** [#113140](https://github.com/zephyrproject-rtos/zephyr/pull/113140):
    "link to image/doc page on your site", "reduce resolution to under 600px".
19. **보드 후크 사용** [#113140](https://github.com/zephyrproject-rtos/zephyr/pull/113140):
    "use board hook"(→ `board_early_init_hook()`).
20. **flash → mapped partitions** [#113140](https://github.com/zephyrproject-rtos/zephyr/pull/113140):
    "move to mapped partitions".
21. **보드가 만든 Kconfig 참조 금지** [#113140](https://github.com/zephyrproject-rtos/zephyr/pull/113140):
    "don't refer to Kconfigs created in boards".
22. **sysbuild.cmake 과잉 제거** [#113140](https://github.com/zephyrproject-rtos/zephyr/pull/113140):
    "no?" → 파일 삭제.
23. **hw 표는 자동 생성** [#113140](https://github.com/zephyrproject-rtos/zephyr/pull/113140):
    "this is what the hw table generates so delete".
24. **SoC Kconfig 네이밍** [#120245](https://github.com/zephyrproject-rtos/zephyr/pull/120245):
    nordicjm — "not named right .../style/kconfig.html".
25. **`depends on $(dt_nodelabel_has_prop,...)`** [#120245](https://github.com/zephyrproject-rtos/zephyr/pull/120245).
26. **SPDX 헤더** [#120245](https://github.com/zephyrproject-rtos/zephyr/pull/120245):
    "`SPDX-FileCopyrightText: Copyright 2026 NXP` for new added file".
27. **IS_ENABLED vs #ifdef** [#120245](https://github.com/zephyrproject-rtos/zephyr/pull/120245):
    design_guidelines.rst 인용 양측 논쟁.
28. **테스트에서 빠진 보드의 overlay 삭제** (Boards 샘플): "Shouldn't the overlay file
    be removed if the test no longer runs".
29. **리뷰어가 선례 링크 직접 제시** [#106183](https://github.com/zephyrproject-rtos/zephyr/pull/106183):
    "See esp32c5_devkitc_hpcore.dts#L57".

### 9.3 WisBlock 초기 지원 (#113140) — juanjin-dev 병합 PR 실리뷰

`juanjin-dev`의 WisBlock 초기 지원 PR [#113140](https://github.com/zephyrproject-rtos/zephyr/pull/113140)
(2026-09-28 병합). 리뷰어는 `nordicjm`(Boards/SoC), `jfischer-no`(USB/CDC).
nordicjm이 CHANGES_REQUESTED 3회 후 APPROVED, kartben도 APPROVED.

**리뷰 준비(하드웨어 자료):**

- WisBlock 모듈 데이터시트는
  `https://docs.rakwireless.com/product-categories/wisblock/<소문자 모델명>/datasheet/`
  형식입니다(예: `rak19010` → `.../rak19010/datasheet/`). 리뷰 전에 해당 URL을 직접
  열어 핀 배치·전원·전기적 특성을 확인합니다.
- **스키매틱은 리뷰 전에 사용자에게 요구합니다.** 폴라리티·핀 배선·부스트/레벨 시프터
  유무 같은 보드 레벨 판단은 데이터시트 텍스트만으로 확정할 수 없으므로, 작성자
  (사용자)에게 스키매틱 이미지를 받아 대조한 뒤에야 오버레이/pinctrl을 확정합니다.

실제로 지적된 것들(32개 인라인 코멘트):

1. **저작권** — "you cannot steal people's copyright"(Kconfig.rak4631),
   최종 리뷰 "most commits OK but adding a copyright to a file where you've done
   nothing is not acceptable". → 되돌림.
2. **doc 이미지** — "delete from here and link to image/doc page on your site"
   (mounting.webp), "revert back to original image, we don't need a 100KiB larger
   repo"(rak4631.webp), "vastly reduce resolution to under 600px... remove
   background"(rak19007.webp) → 400x400/19 KiB로 축소.
3. **자동 생성 하드웨어 표 삭제** — "this is what the hw table generates so delete"
   (doc/index.rst).
4. **flash → mapped partitions** — rak4631.dts "move to mapped partitions".
5. **MCUboot 예외 제거** — "just don't apply the shield to MCUboot, get rid of this"
   (rak19007 Kconfig.defconfig).
6. **보드가 만든 Kconfig 참조 금지** — "don't refer to Kconfigs created in boards".
7. **board hook** — board.c "use board hook" → `board_early_init_hook()`로 교체.
8. **sysbuild.cmake 제거** — "no?" → 파일 삭제.
9. **CDC ACM** — jfischer-no: "Shields should not enable CDC ACM serial backend (and
   USB)" → 콘솔 Kconfig/DT 노드를 실드→보드로 이동.
10. **board_porting 권고 준수** — "Must follow .../board_porting.html#general-recommendations".
11. **기존 사용자 참조** — "Use existing users in `boards/` as reference".
12. **커밋 메시지 이유** — "nothing in commit messages that describe why you have to
    add new UDC/USB configuration".
13. **공용 dtsi 사용** — "Now includes the common `cdc_acm_serial.dtsi` instead of the
    handwritten node".
14. **코멘트 미해결 시 리뷰 중단 경고** — nordicjm: "this is not fixed, abandoning
    review because comments have NOT been addressed".
15. **DT_HAS_ZEPHYR_CDC_ACM_UART_ENABLED 분기 오용** — jfischer-no: "That is not how
    it should be used".

**교훈**: 저작권 헤더는 실제로 작성한 파일에만, 이미지는 링크+최소 용량, 실드는
USB/CDC를 켜지 말 것, 보드 전용 Kconfig를 실드가 참조하지 말 것, 모든 리뷰 코멘트에
응답하고 해결 상태를 알릴 것.

### 9.4 Bluetooth Host — jhedberg·Thalley·anchao

1. 스펙 인용 [#122134](https://github.com/zephyrproject-rtos/zephyr/pull/122134):
   "BLUETOOTH CORE SPECIFICATION Version 6.3 | Vol 4, Part E, Section 7.8.56".
2. Kconfig 버퍼 산술 [#122616](https://github.com/zephyrproject-rtos/zephyr/pull/122616).
3. declarations before statements [#122449](https://github.com/zephyrproject-rtos/zephyr/pull/122449).
4. `(void)` 캐스트 [#122537](https://github.com/zephyrproject-rtos/zephyr/pull/122537).
5. 에러 반환값 확인 [#122348](https://github.com/zephyrproject-rtos/zephyr/pull/122348):
   "errors shall per our coding guidelines be tested".
6. 마법 숫자 근거 [#122537](https://github.com/zephyrproject-rtos/zephyr/pull/122537).
7. 죽은 코드 [#122449](https://github.com/zephyrproject-rtos/zephyr/pull/122449).
8. quirk/예외 검증 [#122134](https://github.com/zephyrproject-rtos/zephyr/pull/122134).
9. `static ZTEST` [#122537](https://github.com/zephyrproject-rtos/zephyr/pull/122537).
10. LOG_ERR 상향 [#122134](https://github.com/zephyrproject-rtos/zephyr/pull/122134).

### 9.5 Devicetree Binding — marekmatej·maass-hamburg·nordicjm

1. 속성명 = 실제 동작 [#121099](https://github.com/zephyrproject-rtos/zephyr/pull/121099):
   "rename it to reflect this. Eg. `dummy-comp-disable`".
2. deprecation 릴리스 타이밍 [#121099](https://github.com/zephyrproject-rtos/zephyr/pull/121099).
3. child-binding에 compatible 금지 [#120572](https://github.com/zephyrproject-rtos/zephyr/pull/120572).
4. 배열 const [#119977](https://github.com/zephyrproject-rtos/zephyr/pull/119977):
   "const: 4 is not building. keep const: [4]".
5. 설명 모순 [#119942](https://github.com/zephyrproject-rtos/zephyr/pull/119942)(Copilot).
6. DT 주석은 `/* */` [#119942](https://github.com/zephyrproject-rtos/zephyr/pull/119942).
7. 표준 속성 `clock-frequency` [#119917](https://github.com/zephyrproject-rtos/zephyr/pull/119917).
8. `spi-max-frequency` min/max [#119917](https://github.com/zephyrproject-rtos/zephyr/pull/119917).

### 9.6 Power Management — mathieuchopstm·ZhaoxiangJin·erwango

1. 문서 정확성 [#121069](https://github.com/zephyrproject-rtos/zephyr/pull/121069):
   "the 'default' case is in fact run post ops... then resume devices".
2. toolchain gate [#121069](https://github.com/zephyrproject-rtos/zephyr/pull/121069).
3. Kconfig 의존성 주석 [#121069](https://github.com/zephyrproject-rtos/zephyr/pull/121069).
4. **API 변경 = migration guide** [#118977](https://github.com/zephyrproject-rtos/zephyr/pull/118977):
   erwango — "This is an API change... a note in the migration guideline should have
   been issued".
5. IS_ENABLED 논쟁 [#120245](https://github.com/zephyrproject-rtos/zephyr/pull/120245).
6. Kconfig 네이밍 [#120245](https://github.com/zephyrproject-rtos/zephyr/pull/120245).
7. `cpu-power-states` 구성 [#118685](https://github.com/zephyrproject-rtos/zephyr/pull/118685).

### 9.7 Sensor·Drivers — sylvioalves·teburd·pdgendt·XenuIsWatching

1. race 조건 [#114551](https://github.com/zephyrproject-rtos/zephyr/pull/114551):
   "should pool_free also clear next?".
2. atomic status 단어 [#114551](https://github.com/zephyrproject-rtos/zephyr/pull/114551).
3. tracing prototype 누락 [#114551](https://github.com/zephyrproject-rtos/zephyr/pull/114551).
4. heap 기본 0 → `HEAP_MEM_POOL_ADD_SIZE_I3C` [#100369](https://github.com/zephyrproject-rtos/zephyr/pull/100369).
5. Kconfig `depends on I3C_CONTROLLER || I3C_TARGET` [#100369](https://github.com/zephyrproject-rtos/zephyr/pull/100369).
6. peripheral API 선호 [#120576](https://github.com/zephyrproject-rtos/zephyr/pull/120576):
   "use sl_hal_emu over emlib".
7. platform_allow [#120274](https://github.com/zephyrproject-rtos/zephyr/pull/120274).
8. DMA check 제거 [#119586](https://github.com/zephyrproject-rtos/zephyr/pull/119586).
9. global overlay vs board conf [#119586](https://github.com/zephyrproject-rtos/zephyr/pull/119586).
10. platform_allow는 별도 커밋 [#119586](https://github.com/zephyrproject-rtos/zephyr/pull/119586).
11. UART FIFO flush(Copilot) [#121242](https://github.com/zephyrproject-rtos/zephyr/pull/121242).
12. 반환값 확인 [#119534](https://github.com/zephyrproject-rtos/zephyr/pull/119534).
13. 빈 줄 정리 [#118891](https://github.com/zephyrproject-rtos/zephyr/pull/118891).
14. **드라이버 동기화는 `struct k_sem`** — 접근 제어/완료 대기는 뮤텍스보다 세마포어를
    씁니다. [#113278](https://github.com/zephyrproject-rtos/zephyr/pull/113278): JordanYates
    — "Most drivers use semaphores rather than mutexes for access control, since they are
    lower overhead and generally we don't actually expect any contention for the resource."
    → `struct k_sem`로 교체.

### 9.8 Kernel — nashif·maass-hamburg·sylvioalves

1. 스코프 최소화 [#122606](https://github.com/zephyrproject-rtos/zephyr/pull/122606).
2. 죽은 코드 제거 [#122606](https://github.com/zephyrproject-rtos/zephyr/pull/122606).
3. deadlock [#119031](https://github.com/zephyrproject-rtos/zephyr/pull/119031):
   "k_obj_type_walk_locked() holds obj_core_lock... deadlocks on smp".
4. NULL deref [#119031](https://github.com/zephyrproject-rtos/zephyr/pull/119031):
   "`_current` is still NULL when CONFIG_MULTITHREADING=n".
5. FIFO 이중 등록 [#119031](https://github.com/zephyrproject-rtos/zephyr/pull/119031).
6. 테스트가 아무것도 검증 안 함 [#118933](https://github.com/zephyrproject-rtos/zephyr/pull/118933):
   "rem <= 2 accepts 1 and 2... passes without testing".
7. K_PRIO_COOP로 sleep 제거 [#119305](https://github.com/zephyrproject-rtos/zephyr/pull/119305).
8. userspace 태그 제거 [#119305](https://github.com/zephyrproject-rtos/zephyr/pull/119305).

### 9.9 Networking — rlubos·jukkar

1. RFC 인용 [#121918](https://github.com/zephyrproject-rtos/zephyr/pull/121918):
   "RFC 9000 §10.1", "RFC 9002 §6.4".
2. `#if defined()` 관례 [#122141](https://github.com/zephyrproject-rtos/zephyr/pull/122141).
3. 패킷 인코딩 바이트 검증 [#122434](https://github.com/zephyrproject-rtos/zephyr/pull/122434):
   QNAME trailing dot → malformed query.
4. 대기 상한 [#122023](https://github.com/zephyrproject-rtos/zephyr/pull/122023):
   "we could wait forever... CONFIG_NET_SOCKET_MAX_SEND_WAIT".
5. lock window [#122006](https://github.com/zephyrproject-rtos/zephyr/pull/122006).
6. 32비트 정수 wrap [#121930](https://github.com/zephyrproject-rtos/zephyr/pull/121930).
7. psa crypto API [#121926](https://github.com/zephyrproject-rtos/zephyr/pull/121926).
8. 도달 불가 분기 [#121918](https://github.com/zephyrproject-rtos/zephyr/pull/121918).

---

### 9.10 최신 관례 (바인딩 title·examples 배치, device-not-ready 로깅 등)

**바인딩 YAML `title:`**
- 근거: `doc/build/dts/bindings-syntax.rst` §"Title" — "title: Concise title for the long
  description [optional]", "Titles should not exceed 100 characters. The description
  field should be used [for the long text]".
- 도입: [#89967](https://github.com/zephyrproject-rtos/zephyr/pull/89967) "dts: bindings:
  add title property", 선택 여부 재명시
  [#99301](https://github.com/zephyrproject-rtos/zephyr/pull/99301), 보드 하드웨어 표
  생성에 사용 [#88124](https://github.com/zephyrproject-rtos/zephyr/pull/88124), 오류
  수정 [#90337](https://github.com/zephyrproject-rtos/zephyr/pull/90337).
- 관례: 신규 바인딩에는 100자 이내 `title:`을 붙입니다(트리 4401개 중 416개).

**바인딩 YAML 키 배치 — `examples:`는 하단**
- 표준 키 순서(위 템플릿): `title` → `description` → `include` → `compatible` →
  `properties` → `child-binding` → `bus`/`on-bus` → **`examples`** → `*-cells`.
  `examples`는 주소/셀 정의(`*-cells`) 바로 앞, 즉 **하단**에 배치합니다.
- 리뷰 사례 [#120360](https://github.com/zephyrproject-rtos/zephyr/pull/120360):
  dhsu-nordic — "Could this be moved to the bottom, so it aligned with the address
  locations."

**바인딩에서 CI(DevicetreeBindings)가 잡는 것**
- `required: false`는 기본값이라 중복 → 제거(`required_false_check`).
- `compatible`과 파일명 일치(`compatible_and_file_name_match_check`).
- 속성명 검사(`check_yaml_property_name`).
- 이 셋은 `scripts/ci/check_compliance.py`의 `DevicetreeBindings`가 강제하므로 사람
  리뷰에서 다시 요구하지 않습니다(위 §5 원칙).

**`LOG_ERR_DEVICE_NOT_READY(dev)` / `LOG_WRN_DEVICE_NOT_READY(dev)`**
- 정의: `include/zephyr/device.h:927`(ERR), `:916`(WRN). `device_is_ready()` 체크
  실패 시 사용합니다("%s device not ready").
- 도입: [#104323](https://github.com/zephyrproject-rtos/zephyr/pull/104323) "Unify
  'device not ready' errors" — nashif: "should be declared in device.h, not a general
  logging macro"; butok의 요청으로 `LOG_WRN_DEVICE_NOT_READY` 추가.
- 리뷰 사례 [#119917](https://github.com/zephyrproject-rtos/zephyr/pull/119917):
  maass-hamburg — "use `LOG_ERR_DEVICE_NOT_READY`".

**`DEVICE_DT_GET_COMMA` / `DEVICE_DT_INST_GET_COMMA`**
- 정의: `include/zephyr/device.h:335`, `:358`. 콤마 리스트 생성 매크로.
- 리뷰 사례 [#118127](https://github.com/zephyrproject-rtos/zephyr/pull/118127):
  henrikbrixandersen — "Should this be `DEVICE_DT_INST_GET_AND_COMMA()` to match our
  current examples of `REG_AND_COMMA()`?"

**`WAIT_FOR(expr, timeout, delay_stmt)`**
- 정의: `include/zephyr/sys/util.h:1102`. 폴링 루프 단순화.
- 리뷰 사례 [#115474](https://github.com/zephyrproject-rtos/zephyr/pull/115474):
  bjarki-andreasen — "This could be simplified with the `WAIT_FOR` macro".

**`k_timepoint_t` / `sys_timepoint_calc()`**
- 수동 시간 계산 대신 timepoint API 사용.
- 리뷰 사례 [#118531](https://github.com/zephyrproject-rtos/zephyr/pull/118531):
  pdgendt — "Use `k_timepoint_t` instead."

### 9.11 칩 패밀리 드라이버 지향 (드라이버 설계 관례)

같은 레지스터 맵을 공유하고 해상도·핀 기능·채널 수만 다른 칩들은 **드라이버 하나로**
처리하는 것이 관례입니다. `juanjin-dev`의 병합된 ADC 드라이버가 표준 예시입니다.

**예시 1 — ADS7828/ADS7830** [#114359](https://github.com/zephyrproject-rtos/zephyr/pull/114359):
- 드라이버 하나: `drivers/adc/adc_ads7828.c`.
- 바인딩: `dts/bindings/adc/ti,ads7828-base.yaml`(공통 필드, `compatible` 없음) +
  `ti,ads7828.yaml`/`ti,ads7830.yaml`(각각 `compatible` + `include: ti,ads7828-base.yaml`).
- Kconfig 하나: `ADC_ADS7828`, `depends on DT_HAS_TI_ADS7828_ENABLED ||
  DT_HAS_TI_ADS7830_ENABLED`.

**예시 2 — ADCxx1C 패밀리(6종)** [#114289](https://github.com/zephyrproject-rtos/zephyr/pull/114289):
- 드라이버 하나: `drivers/adc/adc_adcxx1c.c`.
- 바인딩: `ti,adcxx1c-base.yaml` + 칩별 6개(`ti,adc081c021.yaml`, `ti,adc081c027.yaml`,
  `ti,adc101c021.yaml`, `ti,adc101c027.yaml`, `ti,adc121c021.yaml`, `ti,adc121c027.yaml`).
- Kconfig 하나: `ADC_ADCXX1C`, 6개 compatible의 `DT_HAS_..._ENABLED` OR로 의존.
- Kconfig help의 근거 문구: "The six devices share one register map and differ only in
  resolution (8, 10 or 12 bit) and in the function of one package pin, so they are all
  handled by a single driver."

**예시 3 — WIZnet W5100S (공용 MACRAW 코어)** [#113315](https://github.com/zephyrproject-rtos/zephyr/pull/113315):
**리뷰어가 명시적으로 패밀리 구조를 요구한 사례**입니다.

- parthitce: "This is almost a straight copy of eth_w5500.c. The only real differences are
  the SPI framing, the PHY status bit polarity, and the NETLCKR unlock. IMO it makes sense
  to wrap around with one shared MACRAW core and per-chip structs."
- 결과 구조: `drivers/ethernet/wiznet/eth_wiznet.c`(공용 MACRAW "socket 0" 코어) +
  칩별 `eth_w5100s.c` + 공용 `eth_wiznet.h`. 기존 W5500/W6100도 이 공용 코어로 이동.
- Kconfig 공용화(maass-hamburg): "I think these Kconfigs can be shared for all wiznet
  drivers" → `CONFIG_ETH_WIZNET_*` 5개로 통합(구 칩별 이름은 마이그레이션 가이드에).
- `struct ethernet_api`를 구조체 첫 멤버로 두어 api로 사용(maass-hamburg).
- `reset-gpios`/`int-gpios` 존재 시에만 `select GPIO`(maass-hamburg).

**패턴 요약:**
- 드라이버 `.c` 1개 = 칩 패밀리 1개(공용 코어 + 칩별 파일).
- 바인딩 = `<vendor>,<family>-base.yaml`(공통 필드, `compatible` 없음) + 칩별
  `<vendor>,<chip>.yaml`(`compatible` + `include: <vendor>,<family>-base.yaml`).
- Kconfig 1개 = 패밀리 1개, `depends on DT_HAS_<CHIP>_ENABLED`를 OR로 나열.
- 칩별 차이(해상도 등)는 DT 속성 또는 compatible 파생 값으로 처리합니다.
- **칩을 하나 더 추가할 때 기존 드라이버의 복사본이면, 리뷰어가 공용 코어로의
  리팩터링을 요구합니다**(parthitce, #113315).

---

## 10. 시사점 (로컬 리뷰에 반영)

- 리뷰 코멘트의 표준 구조는 **"근거(재현 명령·결과) + 규칙 인용 + 정확한 수정 제안"**.
- 저자는 짧게 응답("Done"). 리뷰어는 "Nit/Minor/Non-blocking/follow-up"으로 우선순위를 표시.
- AI 리뷰(Copilot)도 같은 형식: 재현 단계 + 트리거 조건 + 영향.
- 서브시스템마다 인용 대상이 다릅니다: BT=스펙, Net=RFC, DT=속성 의미, PM=문서·API,
  BSP=타 보드 선례·board_porting 권고.
- **BSP 리뷰에서 가장 자주 걸리는 것**: 저작권 헤더, doc 이미지 용량/링크, 타 보드
  일관성, 복붙 오류, `ranges`/`clocks`/`sda-gpios` 등 DT 누락, mapped partitions,
  board hook, 보드 전용 Kconfig 참조, CDC ACM을 실드가 켜는 것.
