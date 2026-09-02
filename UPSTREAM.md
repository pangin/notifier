# UPSTREAM — notifier

이 저장소는 `event-catalog/notifier`의 **GitHub fork**(fork network 내부, public)입니다. 원본 기준선과 내부 수정선을 분리해 추적하며, 기계가 읽는 값은 [upstream.json](./upstream.json)에 있습니다.

| 항목                          | 값                                                                                                                       |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| upstream                      | https://github.com/event-catalog/notifier (default branch `main`)                                                        |
| fork                          | https://github.com/pangin/notifier                                                                                       |
| 관계                          | GitHub fork (public). 원본 fork network에 연결되어 있어 GitHub compare/PR UI로 upstream과 비교 가능                      |
| 기준(pinned) SHA              | `a5872d5da7ab67abaa3cdf953769104d484edfbd` (2025-07-24T17:01:33+01:00, tag `v0.0.8`)                                     |
| 현재 fetched / integrated SHA | `upstream.json` → `.sha.fetched`, `.sha.integrated` (sync workflow가 갱신)                                               |
| owner                         | 성욱 (pangin)                                                                                                            |
| 역할 분류                     | **optional** — 기존 EventCatalog 디렉터리의 git diff를 읽어 Slack/webhook 알림을 보내는 독립 CLI. core 빌드·기동과 무관. |
| 라이선스(root)                | BUSL-1.1 (Change Date 2029-07-22 → MIT; package.json은 MIT로 표기되어 불일치)                                            |
| 동기화 주기                   | 매주 월요일 00:00 UTC (09:00 KST) 예약 실행 + workflow_dispatch 수동 실행                                                |
| 동기화 identity               | deploy key `upstream-sync`(secret `UPSTREAM_SYNC_SSH_KEY`). 미등록 시 GITHUB_TOKEN으로 push하며 경고 출력                |
| Linear                        | GONG-871, GONG-872, GONG-876, GONG-875                                                                                   |

## 브랜치 모델

| 브랜치                             | 용도                                                  | 규칙                                                                                                         |
| ---------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `vendor/upstream-main`             | upstream `main`의 **exact commit**만 담는 원본 기준선 | 내부 commit·metadata 금지. 사람의 direct push 금지, sync identity만 fast-forward 갱신. force-push·삭제 금지  |
| `main`                             | 내부 수정·배포 기준선(default branch)                 | PR로만 변경, merge commit만 허용(squash/rebase 비활성), force-push·삭제 금지, required check = `Fork Verify` |
| `sync/upstream-<YYYYMMDD>-<sha12>` | upstream 변경을 `main`에 반영하는 PR 브랜치           | bot이 생성. `main`에는 bot이 직접 push하지 않음                                                              |

Git remote는 GitHub 저장소의 속성이 아니라 **checkout/CI 설정**입니다. 모든 checkout과 CI에서 다음 규칙을 적용합니다.

```bash
git clone https://github.com/pangin/notifier.git && cd notifier
git remote add upstream https://github.com/event-catalog/notifier.git
git remote set-url --push upstream DISABLED   # upstream은 fetch-only
git fetch upstream --tags
```

## 동기화 절차 (`.github/workflows/upstream-sync.yml`)

1. `upstream/main`와 tag를 fetch (fetch-only remote).
2. `vendor/upstream-main`이 fetched commit의 조상인지 확인 → 아니면(history rewrite / non-fast-forward) **중단**, owner 검토.
3. `LICENSE*/NOTICE*/COPYING*` 변경이 있으면 **중단**. owner가 검토 후 `acknowledge_license_change=true`로 수동 재실행해야 진행.
4. `vendor/upstream-main`을 fetched commit으로 fast-forward push(sync identity). 새 tag도 push(기존 tag는 덮어쓰지 않음).
5. `main`에서 `sync/upstream-*` 브랜치를 만들고 `git merge --no-ff`(ancestry 보존). 충돌 시 **중단**, owner가 수동 통합.
6. `upstream.json`의 `sha.fetched/integrated`를 갱신해 커밋하고 `main`으로 PR 생성. PR 본문에 fetched SHA, 이전 integrated SHA, 포함된 upstream commit, 내부 patch diff stat을 기록.
7. PR은 required check 통과 후 **Create a merge commit**으로만 merge (squash/rebase 금지).

수동 실행: Actions → `Upstream Sync` → Run workflow. 실행 요약(Job Summary)에 fetched/vendor/integrated SHA, ahead/behind, 내부 patch diff가 표시됩니다.

## 상태 확인 명령

```bash
git fetch upstream --tags && git fetch origin
jq '.sha' upstream.json                                              # pinned / fetched / integrated
git rev-list --left-right --count origin/vendor/upstream-main...origin/main   # upstream-only  내부-only
git diff --stat origin/vendor/upstream-main origin/main               # 내부 patch diff
git merge-base --is-ancestor origin/vendor/upstream-main upstream/main && echo "vendor ⊂ upstream: OK"
```

## Clean build / run (runbook)

| 항목            | 값                                                                                                                           |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Node            | 20.x                                                                                                                         |
| package manager | pnpm (lockfile: pnpm-lock.yaml)                                                                                              |
| install         | `pnpm install --frozen-lockfile`                                                                                             |
| build           | `pnpm run build`                                                                                                             |
| test            | `pnpm vitest run`                                                                                                            |
| run             | `node dist/cli.js detect --catalog <catalog dir> --config ./notifier.yml --dry-run`                                          |
| smoke           | node dist/cli.js --help 가 0으로 종료하고 'detect' 명령을 출력                                                               |
| CI              | `.github/workflows/fork-verify.yml` — PR/push(main, vendor/upstream-main)/수동 실행. jobs: metadata, vendor-integrity, build |

license key와 `.env` 없이(OSS-only) 실행합니다. 빌드·테스트·--dry-run만.

## 라이선스 경계

- root: BUSL-1.1 (Change Date 2029-07-22 → MIT; package.json은 MIT로 표기되어 불일치)
- 상용/제한 경로: `저장소 전체`
- production 제한: production/사내 업무 사용은 유효한 license key 필요(코드상 검사는 없음)
- license key 환경변수: 없음
- 라이선스 파일: `LICENSE.md`
- upstream에서 위 파일이 바뀌면 sync workflow가 자동 반영을 중단하고 owner 검토를 요구합니다.

## upstream workflow가 fork에서 갖는 위험

- release.yml은 repository 가드가 있어 fork에서 skip
- workflow들이 pnpm 8을 지정하지만 lockfile은 v9 — fork 검사는 pnpm 9 사용

## 보호 설정 (GitHub rulesets)

- `main`: PR 필수(승인 1, repository admin은 PR merge 시 bypass 가능), required check `Fork Verify`, merge commit만 허용, force-push·삭제 금지.
- `vendor/upstream-main`: 갱신은 deploy key(sync identity)만 허용, force-push·삭제 금지(bypass 없음).
- 적용 상태는 `upstream.json` → `.protection.appliedAt` 및 GitHub Settings → Rules에서 확인합니다.
