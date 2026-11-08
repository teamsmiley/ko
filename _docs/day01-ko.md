# 1일차

## 설치

### uv 설치

- `uv`(Python 패키지 및 환경 관리 도구)를 설치합니다. 나중에 스킬에서 사용합니다.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### claude code

```bash
curl -fsSL https://claude.ai/install.sh | bash
claude --version
```

### codex

```bash
npm install -g @openai/codex
codex --version
```

## 작업 디렉토리 만들기

```sh
mkdir ~/Desktop/my-ai-agent
cd ~/Desktop/my-ai-agent
code .
```

## git init

```sh
git init
```

git은 로컬에서만 씁니다. 리모트 리포(GitLab, GitHub 등)에는 가능하면 절대 푸시하지 않는 것을 권합니다.

- 이 디렉토리에는 앞으로 티켓 내용, 서버 정보, 작업 결과, 자격 증명 같은 중요한 데이터가 많이 쌓입니다.
- 리모트에 한 번 올라간 데이터는 나중에 지워도 사본이나 캐시가 남을 수 있습니다.
- 로컬 git만으로도 변경 이력 보기, 되돌리기 같은 git의 장점은 그대로 쓸 수 있습니다.

## 위험한 명령 막기

보안상의 이유로 Claude가 위험한 명령을 실행하지 못하게 모두 막아야 합니다. 막을 명령을 프로젝트의 `.claude/settings.json`에 `deny`로 등록합니다. 여기 등록된 명령은 Claude가 실행하려고 해도 거부됩니다.

```sh
mkdir -p .claude
cat > .claude/settings.json <<'EOF'
{
  "permissions": {
    "deny": [
      "Bash(ssh:*)",
      "Bash(scp:*)",
      "Bash(sftp:*)",
      "Bash(rsync:*)",
      "Bash(bash -c:*)",
      "Bash(sh -c:*)",
      "Bash(zsh -c:*)",
      "Bash(eval:*)",
      "Bash(uv run ssh:*)",
      "Bash(uv run scp:*)",
      "Bash(uv run rsync:*)"
    ]
  }
}
EOF
```

- `ssh`, `scp`, `sftp`, `rsync`: Claude가 원격 서버에 직접 접속하거나 파일을 옮기지 못하게 합니다. 서버 작업이 필요하면 Claude가 명령을 알려 주고, 실행은 사람이 합니다.
- `bash -c`, `sh -c`, `zsh -c`, `eval`: 위 명령을 다른 셸로 감싸서 우회하지 못하게 합니다.
- `uv run ssh` 등: `uv run`을 앞에 붙여서 우회하지 못하게 합니다.

## ask infra MCP 설치

```sh
claude mcp add --transport http ask-infra https://mcp.ooma.dev/mcp
claude
```
