# Claude Code Remote Control 설정 가이드

항상 켜둔 리눅스 장비(Jetson, 미니 PC, 서버 등)에서 Claude Code를 실행해 두고,
노트북·폰·다른 PC 어디서든 같은 대화로 이어서 작업하는 방법입니다.

이 문서는 Jetson Orin Nano(Ubuntu) 기준으로 작성했지만, 다른 리눅스 장비도 동일합니다.

---

## 1. 이게 왜 필요한가

Claude Code를 노트북에서 실행하면 대화 기록과 작업 파일이 그 노트북에 남습니다.
그래서 다른 기기에서는 같은 대화를 이어갈 수 없습니다.

Remote Control을 쓰면 구조가 이렇게 바뀝니다.

| 역할 | 담당 |
|---|---|
| 실제 실행, 코드와 파일 보관 | 항상 켜둔 장비 (이하 "서버 장비") |
| 화면과 입력 | 노트북, 폰, 아무 브라우저 |

- 서버 장비의 IP가 바뀌어도 상관없습니다. 서버 장비가 바깥(Anthropic)으로 연결을 거는 방식이라 고정 IP, 포트포워딩, DDNS가 필요 없습니다.
- 서버 장비가 켜져 있고 인터넷에 연결돼 있어야 작업을 시킬 수 있습니다.

## 2. 준비물

- Claude 유료 플랜 계정 (Pro, Max, Ultra 등). API 키 로그인으로는 Remote Control이 동작하지 않습니다.
- 리눅스 서버 장비 1대와 인터넷 연결
- 서버 장비의 터미널에 접근할 방법 (모니터+키보드 또는 SSH)

Anthropic 쪽 추가 요금은 없고, 사용량은 플랜의 기존 한도에서 차감됩니다.

## 3. 이 문서에서 바꿔 써야 하는 값

| 표기 | 의미 | 예시 |
|---|---|---|
| `<USER>` | 서버 장비의 리눅스 사용자 이름 | `orin-nano` |
| `<PROJECT_DIR>` | 작업할 프로젝트 폴더의 전체 경로 | `/home/orin-nano/orin_nano_super` |
| `<NAME>` | 세션 목록에 표시될 이름 | `jetson` |

사용자 이름은 `whoami`, 현재 폴더 경로는 `pwd`로 확인할 수 있습니다.

---

## 4. 설치와 로그인

서버 장비의 터미널에서 진행합니다.

### 4-1. Claude Code 설치

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

설치 확인:

```bash
claude --version
```

`command not found`가 나오면 터미널을 다시 열거나 `source ~/.bashrc`를 실행합니다.

### 4-2. 로그인

프로젝트 폴더에서 Claude Code를 실행합니다.

```bash
cd <PROJECT_DIR>
claude
```

처음 실행하면 아래 순서로 화면이 나옵니다.

1. **테마 선택**: 아무거나 고르고 Enter. 나중에 `/theme`로 바꿀 수 있습니다.
2. **로그인 방식 선택**: Claude 계정(구독)으로 로그인하는 항목을 고릅니다. Console/API 과금 쪽은 고르지 않습니다.
3. **URL 표시**: 서버 장비에 브라우저가 없으면 터미널에 주소가 뜹니다. 그 주소를 노트북 브라우저에서 열어 로그인하고, 표시되는 코드를 서버 장비 터미널에 붙여넣습니다.
4. **폴더 신뢰 여부**: 수락합니다. 한 번 수락해 둬야 Remote Control이 이 폴더에서 바로 실행됩니다.

입력창이 뜨면 `/exit`로 나옵니다.

---

## 5. Remote Control 실행 (수동 테스트)

자동 실행을 설정하기 전에, 먼저 손으로 실행해서 동작을 확인합니다.

### 5-1. tmux 설치

터미널 창을 닫거나 SSH가 끊겨도 프로그램이 죽지 않게 하려고 tmux 안에서 실행합니다.

```bash
sudo apt install -y tmux
```

### 5-2. 실행

```bash
tmux new -s claude
```

화면이 바뀌면 그 안에서 실행합니다.

```bash
cd <PROJECT_DIR>
claude remote-control --name "<NAME>"
```

처음 실행 시 Remote Control을 켤지 물으면 수락합니다. 세션 URL이 표시되고 대기 상태가 됩니다.

### 5-3. 다른 기기에서 접속

서버 장비와 같은 계정으로 아래 중 하나에 접속합니다.

- 브라우저: https://claude.ai/code
- Claude 데스크톱 앱
- Claude 모바일 앱의 **Code** 탭

세션 목록에서 `<NAME>`을 선택합니다. 컴퓨터 아이콘에 초록 점이 있으면 온라인입니다.

테스트로 아래 메시지를 보내봅니다.

```
현재 폴더에 어떤 파일이 있는지 알려줘
```

서버 장비의 프로젝트 폴더 내용이 답으로 오면 성공입니다.

---

## 6. 부팅 시 자동 실행 (systemd)

수동 실행 상태에서는 서버 장비를 재부팅하면 프로세스가 사라집니다.
systemd에 등록해서 전원만 켜면 자동으로 뜨게 만듭니다.

### 6-1. claude 실행 파일 위치 확인

```bash
which claude
```

보통 `/home/<USER>/.local/bin/claude`입니다. 다른 값이 나오면 아래 서비스 파일의 경로를 그 값으로 바꿉니다.

### 6-2. 수동으로 띄운 세션 종료

서비스가 같은 이름의 tmux 세션을 만들기 때문에 먼저 정리합니다.

```bash
tmux kill-session -t claude
```

### 6-3. 서비스 파일 작성

```bash
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/claude-rc.service
```

아래 내용을 붙여넣고 `<USER>`, `<PROJECT_DIR>`, `<NAME>`을 자신의 값으로 바꾼 뒤 저장합니다 (`Ctrl+O`, Enter, `Ctrl+X`).

```ini
[Unit]
Description=Claude Code Remote Control

[Service]
# tmux는 백그라운드로 분리되어 실행되므로 forking 타입
Type=forking
WorkingDirectory=<PROJECT_DIR>
ExecStart=/usr/bin/tmux new-session -d -s claude '/home/<USER>/.local/bin/claude remote-control --name <NAME>'
ExecStop=/usr/bin/tmux kill-session -t claude
# 부팅 직후 인터넷이 아직 안 붙어서 실패해도 10초마다 재시도
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

tmux를 거쳐 실행하는 이유:

- Remote Control 화면이 터미널을 필요로 합니다.
- 나중에 `tmux attach -t claude`로 들어가 상태를 직접 볼 수 있습니다.

### 6-4. 등록하고 시작

```bash
systemctl --user daemon-reload
systemctl --user enable --now claude-rc.service
sudo loginctl enable-linger <USER>
```

`enable-linger`는 꼭 실행합니다. 사용자 서비스는 원래 로그인해야 시작되는데,
이 설정을 켜면 로그인 없이 부팅 시점에 바로 실행됩니다.

### 6-5. 확인

```bash
systemctl --user status claude-rc.service --no-pager
```

`active (running)`이 보이면 정상입니다.
(`--no-pager` 없이 실행해서 화면이 멈춰 있으면 `q`로 빠져나옵니다.)

재부팅 테스트:

```bash
sudo reboot
```

1~2분 뒤 다른 기기의 세션 목록에서 `<NAME>`에 초록 점이 다시 붙으면 완료입니다.

---

## 7. 네트워크

랜선이든 와이파이든 인터넷만 되면 동작합니다.

| 상황 | 동작 |
|---|---|
| 랜선 | 꽂으면 자동 연결 |
| 저장된 와이파이 | 자동 연결 |
| 처음 가는 곳의 와이파이 | 비밀번호를 모르므로 자동 연결 안 됨 |
| 로그인 페이지가 뜨는 와이파이 (카페, 일부 학교망) | 모니터 없이는 연결하기 어려움 |

모니터 없이 들고 다닐 계획이면 폰 핫스팟을 미리 한 번 연결해 두는 것을 권장합니다.
어디서든 핫스팟만 켜면 서버 장비가 자동으로 붙습니다.

터미널에서 와이파이 추가:

```bash
sudo nmcli device wifi connect "와이파이이름" password "비밀번호"
```

---

## 8. 자주 쓰는 명령

### tmux

| 하고 싶은 것 | 명령 |
|---|---|
| 실행 중인 화면 보기 | `tmux attach -t claude` |
| 화면에서 빠져나오기 (프로그램은 유지) | `Ctrl+b` 누르고 손 뗀 뒤 `d` |
| tmux 세션 목록 | `tmux ls` |

`Ctrl+c`나 `exit`로 나오면 Claude가 종료됩니다. 서비스 설정 덕분에 10초 뒤 자동으로 다시 뜹니다.

### 서비스

| 하고 싶은 것 | 명령 |
|---|---|
| 상태 확인 | `systemctl --user status claude-rc.service --no-pager` |
| 재시작 | `systemctl --user restart claude-rc.service` |
| 중지 | `systemctl --user stop claude-rc.service` |
| 자동 실행 해제 | `systemctl --user disable claude-rc.service` |
| 로그 보기 | `journalctl --user -u claude-rc.service -n 50 --no-pager` |

---

## 9. 문제 해결

**세션 목록에 안 보이거나 초록 점이 없다**

- 서버 장비가 켜져 있고 인터넷이 되는지 확인합니다: `ping -c 3 api.anthropic.com`
- 서비스 상태를 확인합니다 (8번의 상태 확인 명령).
- 접속한 기기의 계정이 서버 장비에 로그인한 계정과 같은지 확인합니다.

**`can't find session: ...`**

- 세션 이름 오타인지 확인합니다. 이름은 `claude`입니다.
- `tmux ls`로 실제 세션이 있는지 확인하고, 없으면 서비스 상태와 로그를 봅니다.

**서비스가 계속 재시작된다**

- `which claude` 결과와 서비스 파일의 경로가 같은지 확인합니다.
- `WorkingDirectory` 경로가 실제로 존재하는지 확인합니다.
- 4-2의 로그인과 폴더 신뢰 수락을 해당 폴더에서 했는지 확인합니다.
- 서비스를 멈추고 5-2처럼 수동으로 실행하면 에러 메시지를 직접 볼 수 있습니다.

**Remote Control이 시작되지 않는다**

- API 키가 아니라 Claude 구독 계정으로 로그인했는지 확인합니다. `claude` 실행 후 `/login`으로 다시 로그인할 수 있습니다.

---

## 10. 참고: 대화는 어디에 저장되나

| | 저장 위치 |
|---|---|
| 대화 기록 | 서버 장비(`~/.claude/projects/`)와 Anthropic 서버 (기기 간 동기화용) |
| 코드와 작업 파일 | 서버 장비 |
| 노트북, 폰 | 저장되지 않음. 화면 역할만 함 |

공식 문서: https://code.claude.com/docs/en/remote-control
