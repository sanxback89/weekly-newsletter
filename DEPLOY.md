# 주간 뉴스레터 배포 가이드 (GitHub 통합 방식)

> 2026-10-02 핸드백 뉴스레터 배포 성공 기준. 이 방식이 **1순위**다.
> Chrome·토큰·JS 주입이 전혀 필요 없다.

## 배포 대상

| 뉴스레터 | 레포 경로 | 배포 URL |
|---|---|---|
| 의류 (APPAREL) | `index.html` | https://sanxback89.github.io/weekly-newsletter/ |
| 핸드백 (HANDBAG) | `handbag.html` | https://sanxback89.github.io/weekly-newsletter/handbag.html |

리포지토리: `sanxback89/weekly-newsletter` (branch `main`)

## 전제 조건

- claude.ai 커넥터에 **GitHub 통합**이 연결되어 있을 것 (설정 → 커넥터 → 내 항목 → GitHub 통합 ✓)
- 세션에 `mcp__claude-code-remote__add_repo` 도구가 있을 것

## 절차

### 1. 레포를 push 권한으로 세션에 추가
```
mcp__claude-code-remote__add_repo { owner: "sanxback89", repo: "weekly-newsletter", access: "push" }
```
- 사전 확인용 curl / `gh repo view` / `git ls-remote` 는 하지 않는다 (가짜 404가 나올 수 있음).

### 2. 클론 (한 번만, 타임아웃 넉넉히)
```bash
git clone --depth 1 https://github.com/sanxback89/weekly-newsletter /home/claude/weekly-newsletter
```
- 이미 폴더가 있으면 `git -C /home/claude/weekly-newsletter rev-parse HEAD` 로 살아있는 클론인지 먼저 확인. 무작정 `rm -rf` 금지.
- 429 "Too many concurrent git operations" 이면 10초 쉬고 1회만 재시도.
- 클론 후 `register_repo_root` 호출.

### 3. 파일 교체 → 커밋 → push
```bash
cd /home/claude/weekly-newsletter
cp <생성한 HTML> handbag.html        # 의류면 index.html
git config user.name  >/dev/null || git config user.name  "sanxback89"
git config user.email >/dev/null || git config user.email "baekdoo28@gmail.com"
git add handbag.html
git commit -m "Handbag newsletter YYYY-MM-DD"   # 의류: "index.html newsletter YYYY-MM-DD"
git fetch origin main && git rebase origin/main   # 얕은 클론 push 413 방지
git push origin HEAD:main
```
- 성공 시 `xxxxxxx..yyyyyyy  HEAD -> main` 출력.

### 4. 배포 검증 (생략 금지)
GitHub Pages 재빌드 1~2분. 캐시 우회 쿼리로 폴링:
```bash
for i in 1 2 3 4 5 6 7 8; do
  sleep 25
  curl -sL --max-time 25 "https://sanxback89.github.io/weekly-newsletter/handbag.html?cb=$RANDOM" -o /tmp/dep.html -w "%{http_code} %{size_download} "
  echo "cards=$(grep -c 'class=\"article-card' /tmp/dep.html)"
done
```
카드 수와 헤더 날짜 범위가 이번 주 생성분과 일치하면 완료.

### 5. Outlook 초안 (발송 X)
Microsoft 365 커넥터 `outlook_create_draft` (bodyType html)
- 받는 사람: `doosan.back@yakjin.com`
- 제목: 의류 `Apparel Industry Weekly News_YYYY.MM.DD` / 핸드백 `Handbag & Accessories Industry Weekly News_YYYY.MM.DD`
- 본문: 기간·수록 건수 + 배포 링크 + **AI 자동 생성 자료임을 명시**

## 폴백 순서 (GitHub 통합이 안 될 때만)

1. `add_repo` 가 권한 오류 → 오류 메시지 그대로 보고 + 설정 → 커넥터에서 GitHub 재연결 요청
2. Claude in Chrome 연결 시 → GitHub 웹 업로드(`/upload/main`)로 커밋
3. 둘 다 안 되면 → HTML 파일을 전달하고 상황 보고

## 하지 말 것

| 경로 | 결과 |
|---|---|
| 클라우드 `curl` → GitHub Contents API (PUT) | 프록시가 403 차단 |
| PAT를 URL에 넣은 `git push` (add_repo 없이) | 403 `not in this session's authorized repository set` |
| 토큰을 스크립트·문서에 평문 기록 | 보안상 금지 — 이 방식은 토큰이 필요 없다 |

## 기타 메모

- 템플릿: `template.html` (CSS/JS/레이아웃 수정 금지, placeholder 5개만 교체)
- 월이 바뀌는 주의 DATE_RANGE 는 `SEPTEMBER 25 – OCTOBER 2, 2026` 처럼 양쪽 월을 모두 표기 (기본 스크립트는 시작 월만 써서 오표기됨)
- 클라우드 셸 `curl` 로 뉴스 사이트 HTTP 검증(Stage B)은 정상 동작함 (2026-10-02 확인: 70건 중 200 64건 / 403 6건)
- 서브에이전트는 WebSearch 세션 한도(약 200회)에 걸릴 수 있음 → 브랜드 수가 많은 에이전트는 둘로 쪼개는 것이 안전
