# `sig?!`

*[English](./README.md)*  

`sig?!`(`sig-question-bang`, 即`sigqb`)是一个"代理"你的Linux信号的小工具. 它可以捕获发送给对应程序的"可以捕获"的信号(例如`SIGINT`)等, 并选择将这些信号"吞掉", 或转发为其它信号, 并在同时可调用一行命令, 执行你想要执行的功能.
`sig?!`使用`Nim`开发, 开源于[GitHub](https://github.com/Water-Run/sigqb).  

## 安装

从[Releases](https://github.com/Water-Run/sigqb/releases)下载对应平台的二进制文件, 并将其添加到`PATH`中:  

```bash
cp -f ./sigqb ~/.local/bin/sigqb
chmod +x ~/.local/bin/sigqb
```

执行:

```bash
sigqb
```

获取使用信息:  

```bash
sig?! by waterrun, version 0.1.0
https://github.com/Water-Run/sigqb

Usage:
    sigqb
    sigqb <config.ini> -- <command> [args...]
```

## 使用

`sig?!`的工作流是这样的:  

1. 首先, 需要一个配置文件, 记录对应被捕获的命令, 映射的命令和伴随执行的指令(称为`SIDEEFFECT`)
2. 以这个配置文件启动你想要启动的命令, `sig?!`会根据配置文件执行对应的操作

### 配置文件

配置文件使用`.ini`格式.  
`[SECTION]`是对应被捕获的命令名, 例如`[SIGINT]`, `[SIGHUP]`等等, 采用纯大写的形式.
需要注意的是, Section不能是`[SIGKILL]`这样无法捕获的命令. 在配置文件中, 相同的信号Section只能出现一次.
信号Section对应两个Key, 分别是`MAPPING`和`SIDEEFFECT`. `MAPPING`是映射的命令(和Section的命名一致, 为空时"吞掉", 可以为`SIGKILL`等Section无法捕获的指令), `SIDEEFFECT`是伴随执行的指令, 可以用`"""`表示多行的内容. 两项均是可选的.  

示例:  

```ini
; sigqb config example (INI) — “示例”完整覆盖各类情况
; 约定：
; - [SECTION]：可捕获信号名（纯大写），如 [SIGINT] [SIGHUP] [SIGTERM] ...
; - MAPPING 为空：吞掉信号（不转发给子进程）
; - MAPPING=SIGXXX：把 SECTION 对应信号映射为 SIGXXX 转发给子进程
; - SIDEEFFECT：伴随执行的命令；可用 """ 多行
; - 未出现的 key：表示“未配置该行为”（由 sigqb 默认策略决定）

; A) 有 MAPPING 无 SIDEEFFECT：仅做映射转发，不执行伴随命令
[SIGHUP]
MAPPING=SIGTERM

; B) 有 SIDEEFFECT 无 MAPPING：仅执行伴随命令，信号转发策略走默认（常见是原样转发）
;    该条用于“收到信号时打点/清理”，但不主动改变转发规则
[SIGUSR1]
SIDEEFFECT=sh -c 'echo "[sigqb] SIGUSR1 received: hook only" >&2'

; C) MAPPING 原样转发（显式写同名）：效果等同“明确声明不改信号”，可配合 sideeffect
[SIGALRM]
MAPPING=SIGALRM
SIDEEFFECT=sh -c 'printf "%s [sigqb] SIGALRM forwarded as-is\n" "$(date -Is)" >> "${HOME}/.cache/sigqb.log"'

; D) 吞掉信号（MAPPING 为空）+ 有 sideeffect：经典“吞 Ctrl+C 但做点事”
[SIGINT]
MAPPING=
SIDEEFFECT=echo "[sigqb] SIGINT swallowed (Ctrl+C ignored)" >&2

; E) 仅吞掉（MAPPING 为空）且无 sideeffect：纯吞
[SIGUSR2]
MAPPING=

; F) 映射为强制终止（示例）：SIGQUIT -> SIGKILL，并留痕（多行）
[SIGQUIT]
MAPPING=SIGKILL
SIDEEFFECT="""
set -eu
log="${HOME}/.cache/sigqb.log"
mkdir -p "$(dirname "$log")"
printf "%s [sigqb] SIGQUIT -> SIGKILL (force)\n" "$(date -Is)" >> "$log"
"""

; G) 有 MAPPING + 有 sideeffect（单行）：SIGTERM -> SIGINT（优雅退出路径）
[SIGTERM]
MAPPING=SIGINT
SIDEEFFECT=sh -c 'echo "[sigqb] SIGTERM mapped to SIGINT for graceful shutdown" >&2'

; H) 空 SECTION 的情况（不推荐/通常应视为无效）：
;    INI 标准上“[]”不应被当作合法 section 名；这里仅为文档展示。
;    若你的实现允许，把它解释为“默认/兜底规则”也可以；否则应报错/忽略。
[]
MAPPING=SIGTERM
SIDEEFFECT=echo "[sigqb] default section hit (if supported)" >&2
```

### 使用`sig?!`  

在拥有配置文件后, 就可以使用`sig?!`代理你要执行的命令了:

语法:  

```bash
sigqb <配置文件.ini> -- <需要执行的命令> [这个命令的参数...]
```

例如, 假设上一小节示例中的配置文件为`test.ini`, 我们要执行`python main.py`:  

```bash
sigqb test.ini -- python main.py
```
