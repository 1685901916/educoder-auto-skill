---
name: educoder-vnc-tasks
description: Use when solving EduCoder/头歌 browser tasks with a VNC/canvas lab environment, hidden grading scripts, task instructions with many images, Monaco editors, Selenium/WebDriver setup errors, or multi-level task navigation.
---

# 头歌作业自动化

## Core Rule

Do not operate the VM from partial instructions. First collect the full task text, all task images, and the remote checker scripts. Treat the checker script as the grading source of truth.

## Required Workflow

1. Capture the task page text with browser automation.
2. Enumerate and inspect all task content images, not just screenshots of the current viewport.
3. Summarize the required GUI tree, files, commands, or browser actions from both text and images before acting.
4. Locate the VNC canvas and use screenshots after risky GUI actions.
5. Open a terminal in the VM and inspect grading scripts. Prefer step-specific scripts over broad names:

```sh
find /data/workspace -maxdepth 6 -type f 2>/dev/null | grep -Ei 'check|main|run|test|step'
sed -n '1,220p' /data/workspace/myshixun/src/check.sh
sed -n '1,220p' /data/workspace/myshixun/src/main.py
find /data/workspace/myshixun/src -maxdepth 3 -type f -name 'test.sh' -print -exec sed -n '1,180p' {} \;
```

6. Identify exact files, permissions, expected content, and expected terminal output from the checker:

```sh
grep -R "FILEPATH\|file_path\|-f \|result.txt\|aggregate.csv\|jmeter.log\|geckodriver" /data/workspace/myshixun/src /data/workspace/myshixun 2>/dev/null | head -120
```

7. Apply the task changes, then run the checker locally when possible:

```sh
bash /data/workspace/myshixun/src/check.sh
```

8. Only click the web page Evaluate button after the local checker output matches the page's expected output or the required environment state is verified.

## Browser And VNC Control Notes

- The VM desktop is usually an HTML canvas, so DOM selectors do not control desktop icons, GUI widgets, or terminal text.
- Use browser automation for the EduCoder page, and coordinate-based clicks, screenshots, crop checks, and keyboard input for the VNC canvas.
- If a terminal prompt is visible, send short commands at the prompt. Avoid unnecessary `Ctrl+C` or `Ctrl+L` because bad focus can type literal characters.
- If focus is lost, restore the terminal from the workspace switcher or open a fresh terminal from the desktop context menu.
- Time-extension dialogs can steal focus; clear them before terminal input.
- Avoid heredocs and multi-line paste through VNC when possible. Use short single-line commands or base64 payloads.
- Shell `printf` treats `%` as format markers. When writing CSV text that contains `Error %`, escape it as `%%`, use `printf '%s\n' 'literal text'`, or use `cat` only after confirming multi-line paste works.
- For xterm terminals, hidden textarea `fill()` may submit nothing or strip visible spaces. Prefer focusing `xterm-helper-textarea`, then using real keyboard input and Enter. Verify terminal output before evaluating.

## Environment Setup Tasks

Some EduCoder tasks are not code-fill tasks; they require changing the VM environment. Let the error decide the action.

For WebDriver setup tasks, a failure like:

```text
PermissionError: [Errno 13] Permission denied: '/data/workspace/myshixun/geckodriver/geckodriver'
selenium.common.exceptions.WebDriverException: Message: 'geckodriver' executable may have wrong permissions.
```

usually means the code may already be correct and the driver file lacks execute permission. Follow the task text, then verify:

```sh
chmod 777 /data/workspace/myshixun/geckodriver/geckodriver
ls -l /data/workspace/myshixun/geckodriver/geckodriver
```

Look for executable bits such as `-rwxrwxrwx` before clicking Evaluate. Do not edit sample code to work around an environment permission failure unless the checker requires code changes.

## Monaco Editor Tasks

Some EduCoder tasks use the browser-side Monaco editor instead of files in the VM. Do not rely on ordinary paste or hidden textarea `fill()` for multi-line Python: Monaco can preserve indentation from the current cursor column, reorder visible virtualized lines, or leave stale content in another model.

Prefer editing through the page's Monaco API when available:

```js
const code = `...full source...`;
const models = window.Monaco.editor.getModels();
for (const model of models) model.setValue(code);
if (typeof window.updateMonacoValue === "function") window.updateMonacoValue(code);
```

Verify the real model content, not the visible wrapped editor text:

```js
window.Monaco.editor.getModels().map((m, i) => ({
  i,
  lines: m.getValue().split("\n").slice(0, 20),
  tail: m.getValue().split("\n").slice(-10)
}));
```

EduCoder may keep old self-test output visible after code has been fixed. Re-run self-test and judge the refreshed output before clicking Evaluate.

## Multi-Level Navigation

When a shixun has multiple levels, do not guess task URLs or assume the next URL in history is the missing level. Use platform navigation:

- From the classroom homework list or detail page, click the assignment card's `开始学习` or `继续挑战`.
- Inside the task page, use the bottom-right `上一关` / `下一关` buttons to move between adjacent levels.
- After a level passes, close or click through the success overlay, then click `下一关`.
- If the detail table shows unpassed levels, enter via `继续挑战`, then use `上一关` / `下一关` until the page title matches the missing level.
- To confirm overall completion, return to the homework detail/list page and check the per-level table or card count, such as `5/5` and rows marked `已通过`.

Avoid hard-coding inferred level URLs unless the user explicitly provides one. EduCoder can open a new tab from `继续挑战`; select that tab before acting.

## Evaluation Timing

Do not sleep a fixed 15-20 seconds after every Evaluate. Click Evaluate once, then poll the result panel every 1 second until it contains a terminal state such as:

- `全部通过`
- `恭喜你`
- `不匹配`
- `Traceback`
- `Error`
- `Permission denied`

Then act on the result. This is faster and avoids repeated clicks while the grader is still running.

## Checker Script Principle

The page "expected output" is often the output of `check.sh` or `main.py`, not the content that should be written into files.

Before creating or editing result files, confirm the checker expectation. Useful probes:

```sh
grep -R "result.txt\|aggregate.csv\|jmeter.log\|expected\|assert\|grep" /data/workspace/myshixun/src /data/workspace/myshixun 2>/dev/null | head -160
```

If the checker only tests file existence, create the exact path first:

```sh
mkdir -p /home/headless/Desktop/workspace/myshixun/Jmeter
touch /home/headless/Desktop/workspace/myshixun/Jmeter/jmeter.log
```

If the checker validates file content, write the checker-expected line, not the page's final success text:

```sh
printf '%s\n' '<checker-expected-line>' > /path/from/checker/result.txt
bash /data/workspace/myshixun/src/check.sh
```

## Common Failure Modes

- Acting before task images are inspected, causing GUI hierarchy or file names to be wrong.
- Replacing required GUI elements with "equivalent" ones when hidden graders check component names.
- Confusing expected output with checked file content.
- Trusting tool success alone when platform grading is a separate shell or Python check.
- Clicking Evaluate repeatedly before local checker or environment state passes.
- Reusing a previous level's expected value without reading the new level's scripts.
- Reading only `/data/workspace/myshixun/test.sh` when the real step checker is under `src/stepN/test.sh`.
- Losing time to shell quoting, especially `%` inside CSV and unreliable VNC heredocs.
- Editing Monaco code by normal paste/fill and trusting the visible editor while the real model is stale or malformed.
- Guessing or hard-coding task URLs in a multi-level shixun, which can skip levels.
- Treating every failure as a code bug. Permission, driver, browser, or file-path failures may require environment changes instead.
