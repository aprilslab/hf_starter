# 11주차 칼로리카운터 — HuggingFace Spaces 폴백 배포

10주차 Oracle 배포에서 막힌 학생용 폴백 자산. 6주차 칼로리카운터 코드 + Dockerfile + GitHub Actions sync가 모두 들어있어, **본인 HF 계정 + HF_TOKEN만 준비하면** 본인 칼로리카운터가 `huggingface.co/spaces/<id>/<repo>`로 뜬다.

---

## 0. 진행 순서

```
┌────────────────────────────┐    ┌────────────────────────────┐
│  STAGE 1: 수동 첫 배포      │    │  STAGE 2: 자동 동기화       │
│  (HF Space 생성 + 첫 push) │    │  (GitHub Actions)          │
│                           │ →  │                           │
│  HF 가입 → Space 생성      │    │  GitHub main push          │
│  → 본인 GitHub에 코드 push  │    │  → Actions → HF push       │
│  → HF Space 자동 빌드      │    │  → HF Space 자동 빌드       │
│                           │    │                           │
│  목표: 한 번 떠 있는 것     │    │  목표: 앞으로 자동 동기화    │
│  확인 (§ 3)                │    │  되도록 (§ 4)               │
└────────────────────────────┘    └────────────────────────────┘
```

먼저 STAGE 1로 HF Space에 칼로리카운터가 뜨는 것 확인 → 그 다음 STAGE 2로 자동화. 건너뛰면 디버깅 불가능.

---

## 1. zip 안에 들어있는 것

```
week10_calorie_hf/
├── README.md                        ← 이 파일
├── .gitignore                       ← .env 제외 (필수)
│
├── app.py                           ← 6주차 칼로리카운터 코드 (Gradio)
├── model_config.py                  ← 모델 상수 + InferenceClient
├── requirements.txt                 ← Python 의존성
│
├── Dockerfile                       ← HF Space Docker 모드용 (포트 7860)
├── .env.example                     ← HF_TOKEN 자리 (로컬 테스트용)
│
└── .github/
    └── workflows/
        └── sync-to-hf.yml           ← GitHub push → HF Space sync
```

OCI 트랙에서 쓰던 `nginx-calorie.conf`, `docker-compose.yml`, `cert/`는 HF 트랙에서 사용 안 함 — HF가 HTTPS·도메인·reverse proxy 모두 자동 제공.

---

## 2. 학생이 직접 만들 것

| 항목 | 내용 |
|------|------|
| HF 계정 + Access Token | `huggingface.co/join` → Settings → Tokens → type: **Write** |
| HF Space 생성 | `huggingface.co/new-space` → SDK: **Docker** → Public |
| GitHub Secret `HF_TOKEN` | 본인 GitHub 리포 → Settings → Secrets → 위 토큰 등록 |
| `sync-to-hf.yml`의 `<id>`·`<repo>` | env 블록의 `HF_USER`·`HF_SPACE`를 본인 값으로 교체 |

---

## 3. STAGE 1 — 수동 첫 배포

### 3-1. HF 계정 + Access Token

1. [huggingface.co/join](https://huggingface.co/join) 가입 + 이메일 인증
2. [Settings → Tokens](https://huggingface.co/settings/tokens) → **New token**
   - Token type: **Write** (Read는 push 불가)
   - Name: `aiweb2026-deploy`
3. 발급된 `hf_...` 토큰을 안전한 곳에 임시 저장. **노출 금지.**

### 3-2. HF Space 생성

1. [huggingface.co/new-space](https://huggingface.co/new-space) 접속
2. 입력값:
   - Owner: 본인 사용자명
   - Space name: `calorie-counter` (영문 소문자 + 하이픈)
   - License: `mit` 또는 자유
   - SDK: **Docker** → **Blank**
   - Hardware: `CPU basic · FREE`
   - Visibility: **Public**
3. **Create Space** → URL 확인 (`huggingface.co/spaces/<id>/<repo>`)

### 3-3. 본인 GitHub 리포에 코드 push

```bash
# 학생 PC에서
git clone https://github.com/<본인>/calorie-counter.git
cd calorie-counter
cp -r ~/Downloads/week10_calorie_hf/* .
cp -r ~/Downloads/week10_calorie_hf/.github .
cp ~/Downloads/week10_calorie_hf/.gitignore .

git add .
git commit -m "init: 11주차 HF 폴백 - 칼로리카운터 + sync"
git push origin main
```

### 3-4. GitHub Secret 등록

본인 GitHub 리포 → Settings → Secrets and variables → Actions → **New repository secret**

- Name: `HF_TOKEN`
- Value: 3-1에서 받은 `hf_...`

### 3-5. `sync-to-hf.yml` 본인 값으로 교체

`.github/workflows/sync-to-hf.yml`의 env 블록:

```yaml
env:
  HF_TOKEN: ${{ secrets.HF_TOKEN }}
  HF_USER: <id>           # ← 본인 HF username
  HF_SPACE: <repo>        # ← 본인 HF Space 이름 (3-2의 Space name)
```

### 3-6. 첫 동기화

```bash
git commit -am "feat: HF_USER, HF_SPACE 본인 값으로 교체"
git push origin main
```

- GitHub 리포 → Actions 탭 → 워크플로우 녹색 확인
- 1~3분 후 `huggingface.co/spaces/<id>/<repo>` 접속 → Gradio UI + 칼로리카운터 작동

---

## 4. STAGE 2 — 자동 동기화 검증

push 한 번으로 HF Space가 자동 갱신되는지 확인.

```bash
nano app.py        # title 한 글자 변경
git commit -am "test: 자동 동기화 확인"
git push origin main
```

→ Actions 그린 → 1~3분 후 HF Space에 변경 반영.

---

## 5. 9주차 페이지 Live Demo 버튼 살리기

9주차에 만든 본인 페이지(`<id>.aiweb2026.site`)의 데모 카드 버튼을 HF Space URL로 교체.

```html
<!-- 변경 전 -->
<a href="#" class="btn-disabled">Live Demo (Coming Week 10)</a>

<!-- 변경 후 -->
<a href="https://huggingface.co/spaces/<id>/<repo>"
   class="btn-primary" target="_blank" rel="noopener">Live Demo</a>
```

```bash
# 9주차 페이지 리포에서
git commit -am "feat: Live Demo 링크를 HF Space로 교체"
git push origin main
```

→ 1~2분 후 본인 페이지 새로고침 → 버튼이 살아남.

---

## 6. 자주 막히는 곳

| # | 증상 | 처리 |
|---|------|------|
| 1 | HF Space "Build failed" | Space → Logs → Build. `requirements.txt` 버전 충돌, 7860 포트 미노출 확인 |
| 2 | 빌드는 됐는데 페이지 안 뜸 | Space → Logs → Container. `app.py`가 `0.0.0.0` 바인딩인지, 환경변수 누락 여부 |
| 3 | GitHub Actions `403` / Permission denied | `HF_TOKEN` Secret 값 일치, 토큰 type이 **Write**인지, `<id>`·`<repo>` 교체 여부 |
| 4 | 첫 접속 빈 화면 (콜드 스타트) | HF 무료는 48h inactive 시 sleep. 5~30초 기다리면 깨어남 |
| 5 | OOM (Out of Memory) | 무료 16GB RAM 한도. 큰 모델 사용 시 양자화 또는 외부 API로 변경 |
| 6 | HF Space의 환경변수 누락 | Space → Settings → Variables and secrets에 `HF_TOKEN` 추가 (모델 다운로드용) |

---

## 7. OCI vs HF 비교

| 항목 | OCI (10주차) | HF (이 자료) |
|------|-------------|-------------|
| Live URL | `<id>-demo.aiweb2026.site` | `huggingface.co/spaces/<id>/<repo>` |
| 본인 도메인 | 사용 | 사용 안 함 |
| HTTPS·인증서 | Let's Encrypt 또는 Cloudflare | HF 자동 제공 |
| nginx 설정 | 직접 | 사용 안 함 |
| OS 권한 | Full root | 없음 (PaaS) |
| 상시 가동 | 24/7 | 48h inactive 시 sleep |
| RAM | 1GB + swap | 16GB |
| 카드 | 가입 시 필요 | 불필요 |
| 학기 후 유지 | OCI Always Free | HF 무료 |

평가 기준은 트랙과 무관 — 본인 Live URL 작동 + 기말 핵심 기능 작동 두 가지.

---

**작성일**: 2026-05-13
