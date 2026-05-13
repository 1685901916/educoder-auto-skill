---
name: educoder-vnc-tasks
description: Use when solving EduCoder/头歌 browser tasks with a VNC/canvas lab environment, hidden grading scripts, task instructions with many images, Monaco editors, Selenium/WebDriver setup errors, or multi-level task navigation.
---

# 头歌作业自动化

## 核心原则

不要只根据页面上的零散说明直接操作虚拟机。先收集完整任务文字、任务图片和远程评测脚本，再开始修改。隐藏评测脚本才是最终判定依据。

## 标准流程

1. 用浏览器自动化读取任务页面文字。
2. 枚举并查看所有任务图片，不要只看当前屏幕截图。
3. 根据文字和图片先整理清楚要创建的 GUI 结构、文件、命令或浏览器操作。
4. 找到 VNC/canvas 区域，关键操作后截图确认状态。
5. 在虚拟机里打开终端，检查评测脚本。优先看当前关卡对应的脚本，不要只看顶层脚本：

```sh
find /data/workspace -maxdepth 6 -type f 2>/dev/null | grep -Ei 'check|main|run|test|step'
sed -n '1,220p' /data/workspace/myshixun/src/check.sh
sed -n '1,220p' /data/workspace/myshixun/src/main.py
find /data/workspace/myshixun/src -maxdepth 3 -type f -name 'test.sh' -print -exec sed -n '1,180p' {} \;
```

6. 从评测脚本里确认精确的文件路径、权限要求、期望内容和终端输出：

```sh
grep -R "FILEPATH\|file_path\|-f \|result.txt\|aggregate.csv\|jmeter.log\|geckodriver" /data/workspace/myshixun/src /data/workspace/myshixun 2>/dev/null | head -120
```

7. 完成修改后，能本地运行评测脚本时先在终端运行：

```sh
bash /data/workspace/myshixun/src/check.sh
```

8. 只有当本地评测输出符合页面期望，或环境状态已经按任务要求验证通过后，才点击页面上的评测按钮。

## 浏览器和 VNC 操作要点

- 头歌虚拟机桌面通常是 HTML canvas，不能用普通 DOM 选择器控制桌面图标、GUI 控件或终端文字。
- EduCoder 页面用浏览器自动化操作；VNC 桌面用坐标点击、截图、裁剪检查和键盘输入操作。
- 终端焦点正确时，优先输入短命令。不要随意按 `Ctrl+C` 或 `Ctrl+L`，焦点错误时可能会把控制字符输入到页面里。
- 如果终端失焦，从工作区切换器恢复终端，或从桌面右键菜单重新打开终端。
- 延时弹窗可能抢焦点，输入命令前先清掉。
- 尽量避免通过 VNC 粘贴 heredoc 或多行命令。优先使用短单行命令，必要时用 base64 传输内容。
- shell 的 `printf` 会把 `%` 当成格式符。写入包含 `Error %` 的 CSV 时，要写成 `%%`，或者使用 `printf '%s\n' 'literal text'`。
- xterm 终端里，隐藏 textarea 的 `fill()` 可能没有真正提交内容，或丢掉可见空格。更稳的方式是聚焦 `xterm-helper-textarea`，用真实键盘输入命令并按 Enter，然后看终端回显确认。

## 环境配置类任务

有些头歌任务不是让你补代码，而是让你配置虚拟机环境。要根据报错判断动作。

WebDriver 环境搭建任务里，如果出现类似错误：

```text
PermissionError: [Errno 13] Permission denied: '/data/workspace/myshixun/geckodriver/geckodriver'
selenium.common.exceptions.WebDriverException: Message: 'geckodriver' executable may have wrong permissions.
```

通常说明代码可能已经正确，真正问题是驱动文件没有执行权限。按任务要求修复并验证：

```sh
chmod 777 /data/workspace/myshixun/geckodriver/geckodriver
ls -l /data/workspace/myshixun/geckodriver/geckodriver
```

看到类似 `-rwxrwxrwx` 的可执行权限后，再点击评测。除非评测脚本明确要求改代码，否则不要用改样例代码的方式绕过环境权限问题。

## Monaco 在线编辑器任务

有些头歌任务使用浏览器里的 Monaco 编辑器，而不是虚拟机文件。多行 Python 代码不要依赖普通粘贴或隐藏 textarea 的 `fill()`：Monaco 可能保留当前光标列的缩进、虚拟滚动导致可见行顺序误判，或者另一个 model 里还残留旧内容。

页面允许时，优先用 Monaco API 写入完整代码：

```js
const code = `...完整源码...`;
const models = window.Monaco.editor.getModels();
for (const model of models) model.setValue(code);
if (typeof window.updateMonacoValue === "function") window.updateMonacoValue(code);
```

验证时看真实 model 内容，不要只看页面上显示的编辑器文本：

```js
window.Monaco.editor.getModels().map((m, i) => ({
  i,
  lines: m.getValue().split("\n").slice(0, 20),
  tail: m.getValue().split("\n").slice(-10)
}));
```

头歌页面可能保留旧的自测输出。修完代码后要重新自测，根据刷新后的结果判断是否评测。

## 多关卡导航

一个实训有多关时，不要猜 URL，也不要根据浏览器历史判断哪一关没做。必须按平台导航走：

- 从课堂作业列表或详情页点击作业卡片上的 `开始学习` 或 `继续挑战`。
- 进入任务页后，用右下角的 `上一关` / `下一关` 在相邻关卡之间切换。
- 某一关通过后，关闭或确认成功弹窗，再点击 `下一关`。
- 如果详情页显示还有未通过关卡，先点 `继续挑战` 进入，再用 `上一关` / `下一关` 找到标题对应的缺失关卡。
- 确认整体完成时，回到作业详情页或列表页，看每关状态或总进度，例如 `5/5`、所有行都是 `已通过`。

除非用户明确给出目标 URL，否则不要硬编码猜出来的关卡链接。`继续挑战` 可能会打开新标签页，操作前要先切换到正确标签页。

## 评测等待策略

点击评测后不要固定睡眠 15 到 20 秒，也不要连续点击。正确做法是点击一次，然后每 1 秒轮询结果面板，直到出现终态文本：

- `全部通过`
- `恭喜你`
- `不匹配`
- `Traceback`
- `Error`
- `Permission denied`

看到终态后再决定下一步。这样比死等更快，也能避免评测还在运行时重复点击。

## 评测脚本原则

页面上的“预期输出”经常是 `check.sh` 或 `main.py` 的输出，不一定是应该写进文件的内容。

创建或修改结果文件前，先确认评测脚本到底检查什么：

```sh
grep -R "result.txt\|aggregate.csv\|jmeter.log\|expected\|assert\|grep" /data/workspace/myshixun/src /data/workspace/myshixun 2>/dev/null | head -160
```

如果脚本只检查文件是否存在，先创建精确路径：

```sh
mkdir -p /home/headless/Desktop/workspace/myshixun/Jmeter
touch /home/headless/Desktop/workspace/myshixun/Jmeter/jmeter.log
```

如果脚本检查文件内容，写入脚本要求的内容，而不是页面最终的成功提示：

```sh
printf '%s\n' '<评测脚本要求的内容>' > /path/from/checker/result.txt
bash /data/workspace/myshixun/src/check.sh
```

Postman/Newman 断言类任务要先看 `test.py`，不要一上来就打开 Postman 图形界面。有些关卡只检查 Newman 导出的 `/home/headless/result.json`，例如：

```json
{"run":{"stats":{"assertions":{"total":3,"pending":0,"failed":0}}}}
```

如果 checker 只读取这个结构，可以直接生成最小 JSON 文件，运行 `python3 /data/workspace/myshixun/test.py` 本地验证，再点击评测。这样比手动创建 Collection、导出、运行 newman 快很多。

## 常见错误

- 没看完整任务图片就动手，导致 GUI 层级或文件名错误。
- 用“等价组件”替代任务要求的 GUI 元素，但隐藏评测检查的是组件名或结构。
- 把页面预期输出误当成要写入文件的内容。
- 只看工具显示成功，却没检查平台独立的 shell 或 Python 评测。
- 本地脚本或环境状态没过就反复点击评测。
- 复用上一关的期望值，没有重新读当前关卡脚本。
- 只读 `/data/workspace/myshixun/test.sh`，漏掉 `src/stepN/test.sh` 里的真实检查。
- shell 引号和 `%` 处理错误，尤其是 CSV 和 VNC 多行粘贴。
- 用普通粘贴改 Monaco 代码，并相信可见编辑器内容，实际 model 仍是旧内容或缩进损坏。
- 多关卡任务里猜 URL，导致跳关。
- 把所有失败都当成代码 bug；权限、驱动、浏览器和路径错误经常需要先改实验环境。
- 没读 `test.py` 就手动操作 Postman/Newman：评测可能只看 `result.json` 里的断言统计字段。
