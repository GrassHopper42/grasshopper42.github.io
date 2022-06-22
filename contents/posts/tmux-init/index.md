---
title: "Tmux Setting"
description: ".tmux.conf"
data: 2022-06-17
update: 2022-06-03
tags:
  - tmux
  - vim
  - WSL
---

개발을 처음 배울때부터 WSL을 세팅해서 사용했지만 VSC를 주로 사용하다보니 기본적인 세팅만 해두고 썼지만 이번에 백엔드를 Spring boot로 전환하면서 Vim을 적극적으로 이용해보기로 했다.
Vim을 이용하려고 세팅을 좀 만지다보니 또 세팅병이 도져서 터미널을 갈아엎었다고 볼 수 있을만큼 손을 많이 댔는데 이 과정에서 하루~이틀정도 삽질을 했던것같다.
여러가지 플러그인과 세팅들이 있지만 가장 먼저 Tmux세팅부터 남긴다.

## Tmux?

Tmux는 **Terminal multiplexer**의 일종인데 터미널 화면을 분할해서 사용할 수 있다.
세션에서 서버를 실행중이라면 종료하지 않고 최소화와 같은 기능으로도 사용할 수 있지만 나는 화면분할 위주로 사용하고 있다.

## 용어

본격적인 사용에 앞서 사용과 설정에서의 이해를 위해 Tmux의 실행단위들을 알아보자  

### Pane

**Pane**은 Tmux에서 가장 작은 실행단위로 Tmux에서 화면을 분할한다는 것은 이 Pane을 생성한다는 뜻이다.
가로나 세로로 분할할 수 있고 특정 커맨드를 입력하면 Pane의 갯수에 맞춰 추천하는 화면 구성을 돌아가며 전환해준다.

### Window

Pane이 한개 이상 모여 **Window**를 구성한다.
Window는 VSC나 브라우저의 탭으로 이해하면 되는데 동일한 화면에 나타나지는 않지만 실행중인 세션에서 옮겨가며 사용할 수 있다.

### Session

Window가 하나 이상 모이면 **Session**이 구성된다.
**Session**은 tmux에서 관리하는 가장 큰 실행단위로 tmux를 실행하면 가장 먼저 생성되는 요소이다.
VSC에서의 창 하나로 이해하면 될 것 같다.
`detach`로 작업을 유지한채 최소화 시키고 `attach`를 통해 다시 열 수 있다.

## 설치

Ubuntu를 기준으로 설치는 아주 쉽다.

```bash
sudo apt install tmux
```

설치가 다 됐으면 `tmux`를 입력해 실행할 수 있다.

## 설정

아래는 내가 사용하고 있는 .tmux.conf이다.
root디렉토리(Ubuntu라면 `~/`)에 `.tmux.conf`파일을 만들어주면 된다.

<details>
<summary>.tmux.conf</summary>

```conf
# rmap prefix from 'C-b' to 'C-a'
unbind C-b
unbind C-Space
set-option -g prefix C-a
bind-key C-a send-prefix

# split panes using | and -
bind | split-window -h
bind - split-window -v
unbind '"'
unbind %

# reload config file
bind r source-file ~/.tmux.conf\; display-message "Config reloaded."

# switch panes using Alt-arrow without prefix
bind -n M-Left select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up select-pane -U
bind -n M-Down select-pane -D

# vim setting
is_vim="ps-o state= -o comm= -t '#{pane_tty}' \
    | grep -iqE '^[^TXZ ]+ +(\\S+\\/)?g?(view|n?vim?x?)(diff)?$'"

bind-key -n C-h  if-shell  "$is_vim"  "send-keys C-h"  "select-pane -L"
bind-key -n C-j   if-shell  "$is_vim"  "send-keys C-j"   "select-pane -D"
bind-key -n C-k  if-shell  "$is_vim"  "send-keys C-k"  "select-pane -U"
bind-key -n C-l   if-shell  "$is_vim"  "send-keys C-l"   "select-pane -R"
bind-key -n C-\   if-shell  "$is_vim"  "send-keys C-\\"  "select-pane -l" 

# Enable mouse control (clickable windows, panes, resizable panes)
set -g mouse on

# don't rename windows automatically
set-option -g allow-rename off

# tmux
set-option -sg escape-time 10
set-option -g default-terminal 'tmux-256color'
set-option -ga terminal-overrides ',*256col*:Tc'
set-option -g focus-events on

######################################
###########Design change##############
######################################

## loud or quiet?
set-option -g visual-activity off
set-option -g visual-bell off
set-option -g visual-silence off
set-window-option -g monitor-activity on
set-option -g bell-action none

# modes
setw -g clock-mode-colour colour5
setw -g mode-style bold
setw -g mode-style fg=colour1
setw -g mode-style bg=colour18

# panes
set -g pane-border-style bg=colour0
set -g pane-border-style fg=colour255
set -g pane-active-border-style bg=colour242
set -g pane-active-border-style fg=colour83

# statusbar
set -g status-position bottom
set -g status-justify left
set -g status-bg colour18
set -g status-fg colour137
set -g window-status-style dim
set -g status-left "#{?client_prefix,Ω,ω}"
set -g status-right "#{cpu_bg_color} CPU: #{cpu_icon} #{cpu_percentage} | #[fg=colour233,bg=colour19,bold] %d/%m #[fg=colour233,bg=colour8,bold] %H:%M:%S "
set -g status-right-length 120
set -g status-left-length 20

setw -g window-status-current-style fg=colour1
setw -g window-status-current-style bg=colour18
setw -g window-status-current-style bold
setw -g window-status-current-format ' #I#[fg=colour249]:#[fg=colour255]#W#[fg=colour248]#F '
setw -g window-status-style fg=colour9
setw -g window-status-style bg=colour18
setw -g window-status-style none
setw -g window-status-format ' #I#[fgcolour237]:#[fg=colour250]#W#[fg=colour244]#F 'e

# List of plugins
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-cpu'
set -g @plugin 'egel/tmux-gruvbox'

set -g @tmux-gruvbox 'dark'

# Initialize TMUX plugin manager (keep this line at the very bottom of tmux.conf)
run '~/.tmux/plugins/tpm/tpm'
```

</details>

100줄 가까이 되는 길이라 줄였다. 순서대로 살펴보자면, 가장 먼저 `prefix`키를 기본 설정돼있는 `Ctrl`+`b`에서 `Ctrl`+`a`로 바꿔줬다.
기본 `Ctrl`키 위치에서도 `b`보다 `a`가 누르기 편하기도 하고 나는 `Capslock`버튼과 `Ctrl`버튼을 바꿔줬기 때문에 바로 옆에 붙어있어서 더 편했다.
그리고 수평분할을 `prefix`+`|`, 수직분할을 `prefix`+`-`로 바꿔줬다.

tmux는 `truecolor`를 지원하는데 이것 때문에 계속 오류가 떠서 애를 좀 먹었다.
중간에 _tmux_ 주석 처리 된 부분이 문제의 부분인데 여기저기 찾아보면서 만지작거리다가 이 설정으로 해결됐다.
.zshrc에서 아래의 설정도 해 줘야한다.

```zshrc
export TERM=tmux-256color
```

그 아래 디자인 부분은 tmux 상태바와 기타 다른부분들 디자인인데 잘 모르겠어서 다른 분의 configuration을 그대로 사용했다.
상태바는 맨 아래 부분에서 볼 수 있듯 플러그인을 설치해서 사용중인데 플러그인별로 스타일을 설정하거나 고정돼있어서 딱히 필요없는 듯 하다.
tmux 플러그인은 [깃허브](https://github.com/tmux-plugins/tpm)에서 플러그인 매니저를 설치한 후에 사용할 수 있다.

처음 설정을 했다면 저장 후 tmux를 실행하던가 이미 실행중이라면 터미널에 아래 명령어를 입력해서 reload시켜주자.

```bash
tmux source ~/.tmux.conf
```

처음 `.tmux.conf`가 적용되고 나서 위 conf처럼 단축키를 지정해줬다면 `<prefix>`+`r`키를 이용해 간편하게 reloading할 수 있다.

## 참고자료

- <https://hbase.tistory.com/200>
- <https://velog.io/@piopiop/Linux-tmux%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EC%9E%90>
- <https://velog.io/@ur-luella/tmux-%EC%82%AC%EC%9A%A9%EB%B2%95>
- <https://data-newbie.tistory.com/226>
- <https://www.hamvocke.com/blog/a-guide-to-customizing-your-tmux-conf/>
