---
name: ntpd-fix
description: 서버의 ntpd 또는 chronyd 시간 동기화 문제를 처음부터 끝까지 해결한다 - Jira 티켓 생성, 진단, 표준 플레이북 실행, 검증, 티켓 종료. "fix ntpd on <host>", "ntp issue", "clock drift", HostClockNotSynchronising, ntpd drift.TEMP 권한 오류, ntpd_intres 호스트명 오류에 사용한다.
---

# ntpd-fix

서버 한 대의 시간 동기화 문제를 처리하는 절차입니다. 사용자가 호스트명을 주지 않았으면 물어봅니다.

아래 모든 명령에 쓸 값을 먼저 정합니다.

- `<host>`: 인벤토리 이름. 예: `prom-eqix-sv5`
- Jira 스크립트: `uv run --project .claude/skills/ntpd-fix/scripts .claude/skills/ntpd-fix/scripts/jira_ticket.py`. 아래에서는 `jira_ticket`으로 표기합니다. 프로젝트 루트에서 실행합니다.

Jira 자격 증명은 `.claude/skills/ntpd-fix/assets/.env`에서 읽습니다. 이 파일 내용은 절대 출력하지 않습니다.

## 누가 무엇을 실행하나

이 프로젝트의 `.claude/settings.json`은 Claude의 `ssh`와 `ansible` 실행을 막습니다. 아래의 `!` 명령을 사용자에게 주고 실행하게 한 뒤, 그 출력을 읽습니다.

차단 규칙을 우회하지 않습니다. 예를 들어 `.venv/bin/ansible-playbook`을 직접 부르거나 Ansible을 스크립트로 감싸지 않습니다. 이 규칙은 사용자만 바꿀 수 있습니다.

Ansible은 `~/Desktop/ooma-gitlab/_netops_infra`에서 실행해야 합니다. 플레이북, 인벤토리, `ansible.cfg`가 그 디렉토리에 있기 때문입니다. 이 절차에서는 그곳의 파일을 수정하지 않습니다. 그곳의 `_result/`도 읽지 않습니다. `/etc/shadow` 사본이 들어 있습니다.

## 1. 티켓 생성

사용자에게 이미 티켓이 있으면 이 단계를 건너뜁니다.

먼저 열린 중복 티켓이 있는지 확인합니다.

```sh
jira_ticket search --host <host>
```

텍스트 검색은 관련 없는 티켓도 돌려주므로, 요약을 보고 판단합니다. 이 호스트의 열린 시간 동기화 티켓이 있으면 그것을 쓰고 새로 만들지 않습니다.

```sh
jira_ticket create --project NETOP --summary "Fix ntpd issue on <host>" --description "<symptoms the user reported>"
```

기본값은 `NETOP` 프로젝트와 `Task` 유형입니다. 사용자가 다른 프로젝트를 지정할 때만 바꿉니다. 증상을 지어내지 않습니다. 증상을 듣지 못했으면 조사가 필요하다고 씁니다.

## 2. 진단

사용자에게 다음을 실행하게 합니다.

```sh
! cd ~/Desktop/ooma-gitlab/_netops_infra && uv run ansible <host> -b -m shell -a 'cat /etc/redhat-release; timedatectl; for s in ntpd chronyd; do echo "$s: $(systemctl is-active $s) / $(systemctl is-enabled $s)"; done; ntpq -pn; chronyc sources; ls -ld /var/lib/ntp; grep -E "^(server|peer|pool)" /etc/ntp.conf /etc/chrony.conf; journalctl -u ntpd -u chronyd -n 30 --no-pager; true'
```

출력의 OS를 보고 플레이북을 고릅니다.

| OS           | 플레이북       | 데몬    |
| ------------ | -------------- | ------- |
| CentOS 6/7/8 | `ntpd.yaml`    | ntpd    |
| OL9+         | `chronyd.yaml` | chronyd |

`common-issue-fixer/ntpdate.yaml`은 쓰지 않습니다. ntpdate 서비스를 끄기만 하며, `ntpdate.service`가 죽었을 때 쓰는 것입니다.

다음 경우에는 플레이북을 실행하지 말고 사용자에게 알립니다.

- 설정의 NTP 서버가 죽었거나 다른 곳에서도 닿지 않는 경우. 플레이북으로 고칠 수 없습니다.
- 운영 호스트에서 시간이 크게 어긋난 경우. 재시작하면 시계가 한 번에 바뀔 수 있으니 사용자가 먼저 결정하게 합니다.

## 3. 수정 적용

무엇이 바뀌는지 설명한 뒤 사용자에게 명령을 실행하게 합니다. `ntpd.yaml`은 CentOS 7에서 chrony를 제거하고, ntp를 설치하고, `/etc/ntp.conf`를 표준 템플릿으로 덮어쓰고, `/var/lib/ntp`를 `ntp:ntp 0755`로 맞춘 뒤 ntpd를 활성화하고 재시작합니다.

```sh
! cd ~/Desktop/ooma-gitlab/_netops_infra && uv run ansible-playbook <playbook> --limit <host>
```

결과를 주의해서 읽습니다.

- 재시작 태스크에 `ignore_errors: yes`가 있어서, `failed=0`이라도 데몬이 정상이라는 뜻은 아닙니다. recap의 `ignored=`를 확인하고, 항상 검증합니다.
- 재시작은 항상 `changed`로 나옵니다. `changed=1`이고 나머지가 모두 `ok`이면 설정은 이미 표준이었고 재시작만 된 것입니다. 이때는 근본 원인을 아직 모르며 문제가 다시 생길 수 있다고 사용자에게 알립니다.

## 4. 검증

재시작 후 1~2분 기다린 뒤 사용자에게 다음을 실행하게 합니다.

```sh
! cd ~/Desktop/ooma-gitlab/_netops_infra && uv run ansible <host> -b -m shell -a 'timedatectl; systemctl is-active ntpd chronyd; ntpq -pn; chronyc sources; chronyc tracking; ls -ld /var/lib/ntp; journalctl -u ntpd -u chronyd -n 30 --no-pager; true'
```

| 결과       | ntpd                                         | chronyd                             |
| ---------- | -------------------------------------------- | ----------------------------------- |
| 해결됨     | peer 앞에 `*`, `reach`가 0보다 큼            | `^*` 소스, `Leap status: Normal`    |
| 기다림     | `+` peer만 있거나 `reach`가 아직 올라가는 중 | `^?` 소스이고 `Reach`가 올라가는 중 |
| 해결 안 됨 | `reach`가 계속 0                             | 모두 `^?`, `Reach`가 계속 0         |

해결된 경우에도 `timedatectl`이 시계가 동기화되었다고 보여야 합니다. 해결되지 않았으면 플레이북을 다시 돌리지 않습니다. 다음은 NTP 서버를 살펴봅니다.

## 5. 티켓 종료

사용자가 요청하거나, 검증에서 수정이 확인된 뒤에만 종료합니다.

```sh
jira_ticket comment --key <KEY> --text "<verification result>"
jira_ticket close --key <KEY> --resolution Done --summary "<what was done and what was verified>"
```

요약은 영어로 쓰고, 실제로 일어난 일만 적습니다. 재시작만 했으면 그렇게 씁니다. 사용자가 검증 없이 닫으라고 하면 닫고, 동기화를 검증하지 않았다고 씁니다.

`create`, `comment`, `close`에 `--dry-run`을 붙이면 Jira를 바꾸지 않고 미리 볼 수 있습니다.
