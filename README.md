# livingDesignPoC

株式会社リビングデザイン様提案用PoCプロジェクト

## リポジトリ構成

このリポジトリは、frontendとbackendをGit submoduleとして管理する親リポジトリです。

```text
livingDesignPoC/
├── frontend/              # Next.js（LDfront submodule）
├── backend/               # FastAPI（LDback submodule）
├── .github/workflows/     # frontend/backendの統合CI
├── .gitmodules
└── README.md
```

| ディレクトリ | 技術 | リポジトリ |
| --- | --- | --- |
| `frontend` | Next.js / React / TypeScript | `nao317/LDfront` |
| `backend` | Python / FastAPI / PostgreSQL / pgvector | `nao317/LDback` |

親リポジトリは、各submoduleのブランチではなく、動作確認に使用する特定のコミットを
記録します。そのため、GitHub上でsubmoduleが `main` ではなくコミットIDとして表示される
のは正常です。

## 必要な環境

- Git（submoduleを利用できること）
- GitHubへSSH接続できること
- Node.js 24（CIと同じバージョン）
- npm
- Python 3.13推奨（CIと同じバージョン。現在の実装は3.10以上を想定）
- Docker
- Docker Compose v2（`docker compose` コマンド）
- curl

## 初回セットアップ

### 新しくcloneする場合

submoduleを含めてcloneします。

```bash
git clone --recurse-submodules git@github.com:nao317/livingDesignPoC.git
cd livingDesignPoC
```

通常の `git clone` を行った後でsubmoduleを取得する場合は、次を実行します。

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

取得状態を確認します。

```bash
git submodule status
```

各行の先頭が空白なら、親リポジトリが記録したコミットと一致しています。

### frontendのセットアップ

```bash
cd frontend
npm ci
cd ..
```

### backendのセットアップ

```bash
cd backend
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -r requirements.txt
cd ..
```

`python3` が別ツールのPythonを指している環境でも、作成後は
`backend/.venv/bin/python` を明示して実行できます。

## 全体のビルド

現在、親リポジトリに一括ビルドスクリプトはありません。frontendはNext.jsのビルド、
backendは依存関係・構文・importの確認を行います。

### frontend

```bash
cd frontend
npm ci
npm run lint
npm run build
cd ..
```

### backend

```bash
cd backend
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m pip check
.venv/bin/python -m compileall -q app
.venv/bin/python -c "from app.main import app; print(app.title)"
cd ..
```

backendの期待出力:

```text
Living Design PoC Backend
```

## 全体の実行

開発時は、DB、backend、frontendをそれぞれ別のターミナルで起動します。

### 1. DBコンテナ

親リポジトリのルートから起動できます。

```bash
docker compose -f backend/docker-compose.yml up -d db
docker compose -f backend/docker-compose.yml ps
```

現在のbackend APIはまだDBへ接続していないため、APIだけ確認する場合はDBを省略できます。

### 2. backend

```bash
cd backend
.venv/bin/python -m uvicorn app.main:app --reload --port 8000
```

- API: `http://127.0.0.1:8000`
- Swagger UI: `http://127.0.0.1:8000/docs`

### 3. frontend

```bash
cd frontend
npm run dev
```

- frontend: `http://localhost:3000`

終了する場合は、frontendとbackendを起動した各ターミナルで `Ctrl+C` を押します。
DBコンテナは次のコマンドで停止・削除します。

```bash
docker compose -f backend/docker-compose.yml down
```

DBデータを含むボリュームも削除する場合だけ、`--volumes` を付けます。

```bash
docker compose -f backend/docker-compose.yml down --volumes
```

`--volumes` を付けた場合、対象ボリューム内のDBデータは復元できません。

## 全体の動作確認

### frontend

```bash
cd frontend
npm run lint
npm run build
```

ブラウザで `http://localhost:3000` を開き、画面が表示されることを確認します。

### backend

backendを起動した状態で、別のターミナルからヘルスチェックします。

```bash
curl --fail --silent --show-error http://127.0.0.1:8000/health
```

期待結果:

```json
{"status":"ok"}
```

文書登録、検索、DBコンテナの詳しい確認方法は
[`backend/README.md`](backend/README.md) を参照してください。

### 自動テスト

frontendには現在lintとbuildによる確認があります。backendはpytestを実行するCI構成ですが、
現時点では `tests/` が未作成です。テスト追加後は次のコマンドで実行します。

```bash
cd backend
.venv/bin/python -m pytest
```

## CI

親リポジトリの `.github/workflows/ci.yml` は、pushまたはPull Requestでsubmoduleを
再帰的に取得し、次を実行します。

- frontend: `npm ci`、lint、Next.js build
- backend: PostgreSQL + pgvector、Ruff、pytest

CIがsubmoduleを取得するには、親リポジトリが参照するfrontend/backendのコミットが
それぞれのGitHubリポジトリへpush済みである必要があります。

## Submoduleの管理

### 基本的な考え方

`.gitmodules` の `branch = main` は、`git submodule update --remote` が参照する
リモートブランチを指定する設定です。通常のcheckoutやcloneでは、親リポジトリが記録した
コミットが優先され、自動的に各submoduleの最新 `main` へ移動するわけではありません。

### 状態確認

```bash
git submodule status
git -C frontend status --short --branch
git -C backend status --short --branch
```

`git submodule status` の先頭記号は、主に次の意味です。

| 記号 | 状態 |
| --- | --- |
| 空白 | 親リポジトリが記録したコミットと一致 |
| `-` | submoduleが未初期化 |
| `+` | 現在のsubmoduleコミットが親の記録と異なる |
| `U` | submoduleの参照が競合している |

### 親リポジトリが記録した状態へ合わせる

作業中の変更や未pushのコミットがないことを確認してから実行します。

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

この操作では、各submoduleがdetached HEADになる場合があります。これは親が固定した
コミットを再現するsubmoduleの通常動作です。submoduleで新しい作業を始める場合は、
先に作業ブランチへ切り替えてください。

### submoduleで変更を作る

backendを例にすると、先に子リポジトリでブランチ作成、commit、pushを行います。

```bash
git -C backend switch main
git -C backend pull --ff-only origin main
git -C backend switch -c feature/<issue-number>-<summary>

# backend内のファイルを変更した後
git -C backend add <files>
git -C backend commit -m "feat: <summary>"
git -C backend push -u origin feature/<issue-number>-<summary>
```

子リポジトリのPull Requestをmergeした後、親リポジトリが参照するコミットを更新します。

```bash
git -C backend switch main
git -C backend pull --ff-only origin main
git add backend
git commit -m "chore: update backend submodule"
git push
```

frontendの場合も、`backend` を `frontend` に読み替えて同じ順序で操作します。

親リポジトリではsubmodule内の個別ファイルではなく、`backend` または `frontend` という
submoduleの参照をcommitします。

### 両submoduleをリモートのmainへ更新する

作業中の変更がないことを確認してから実行します。

```bash
git submodule update --remote --recursive
git diff --submodule=log
git add frontend backend
git commit -m "chore: update submodules"
```

`git submodule update --remote` は `.gitmodules` の `branch = main` を参照します。
更新後は必ずfrontend/backendの組み合わせをビルド・確認してから親の参照をcommitします。

### 親リポジトリをpullした後

親リポジトリの更新にsubmodule参照の変更が含まれる場合は、次を実行します。

```bash
git pull --ff-only
git submodule sync --recursive
git submodule update --init --recursive
```

### 注意点

- submoduleに未コミットの変更がある状態で更新コマンドを実行しない
- 子リポジトリのコミットをGitHubへpushしてから、親のsubmodule参照をpushする
- 親のPRでは `git diff --submodule=log` で参照先コミットを確認する
- `.gitmodules` を変更した後は `git submodule sync --recursive` を実行する
- 親リポジトリだけをcloneした場合は `git submodule update --init --recursive` を実行する

## ブランチ戦略

このプロジェクトでは、PoCとしての快適な開発体験と変更履歴の整理を両立するため、以下のブランチ運用を基本とします

- `main` : 常に動作確認済みの状態を維持するブランチ（デフォルトブランチとして設定）
- `feature/<issue number>-<summary>` : 新規機能やページを作成するためのブランチ
- `fix/<issue number>-<summary>` : 不具合修正用のブランチ
- `docs/<issue number>-<summary>` : READMEや設計・メモなどドキュメント更新のためのブランチ


作業は原則としてIssue駆動で行い、作業完了後にはPullRequestを作成する。PRでは対応Issue、変更内容、確認方法を明記する

## Issue戦略

Issueは、作業内容、背景・完了条件を明確に定義し、PoCの進行状況を追跡するために利用する。

### Issueの種類

- `feature` : 新機能や画面・プロトタイプの追加
- `bug` : 不具合や期待動作との差分の修正
- `docs` : ドキュメントの追加・修正
- `task` : 調査、環境整備、軽微な作業
- `discussion` : 仕様検討や意思決定が必要な内容

### Issueに記載する内容

- 背景・目的
- 対応内容
- 完了条件
- 確認方法
- 関連する資料・Pull Request

### 運用ルール

- 1つのIssueには、1つの目的または成果物を定義します。
- 着手前にIssueを作成し、担当者と優先度を設定します。
- 作業ブランチ名にはIssue番号を含めます。
- Pull Requestを作成する際は、関連Issueを紐付けます。
- 完了条件を満たし、レビューまたは確認が完了したIssueをクローズします。
