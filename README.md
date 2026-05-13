# 头歌作业自动化 Skill

这是一个用于 Codex 的公开 skill，面向 EduCoder/头歌实验、实训、作业任务的浏览器自动化和 VNC 环境排查。

它适合这些场景：

- 头歌任务页面包含 VNC/canvas 实验环境。
- 任务有隐藏评测脚本，页面期望输出和真实文件内容容易混淆。
- 需要读取任务图片、远程 checker、终端输出后再操作。
- Monaco 在线编辑器粘贴代码后出现缩进错乱、旧内容残留。
- Selenium/WebDriver/geckodriver 环境配置失败。
- 多关卡作业需要按 `继续挑战`、`上一关`、`下一关` 正确推进。

## 安装

把 `educoder-vnc-tasks` 目录复制到 Codex skills 目录：

```text
C:\Users\<你的用户名>\.codex\skills\educoder-vnc-tasks
```

最终结构应类似：

```text
.codex/
  skills/
    educoder-vnc-tasks/
      SKILL.md
```

## 使用

在 Codex 里提出类似请求时会触发：

```text
用 educoder-vnc-tasks skill 帮我做这个头歌作业
```

或：

```text
这个 EduCoder 任务有 VNC 环境和隐藏评测，帮我自动化完成
```

## 设计原则

- 不猜 URL，不跳关，按平台按钮推进。
- 不只看页面描述，必须核对任务图片和 checker。
- 不把页面最终成功文本误写进被检查的文件。
- 不用固定长等待，评测后轮询终态。
- 遇到权限、驱动、浏览器环境错误，优先排查实验环境，而不是盲目改代码。

## 许可证

MIT
