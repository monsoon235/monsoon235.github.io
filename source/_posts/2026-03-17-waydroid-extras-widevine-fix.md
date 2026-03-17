---
title: "修复 waydroid-extras install widevine 报错"
date: 2026-03-17
lang: zh-CN
tags:
  - linux
  - waydroid
  - android
---

## 问题

运行 `sudo waydroid-extras install widevine` 时出现如下报错：

```
ERROR: [01:49:04] Stopping container

Traceback (most recent call last):
  File "/usr/bin/waydroid-extras", line 357, in <module>
    main()
    ~~~~^^
  File "/usr/bin/waydroid-extras", line 350, in main
    args.func(args)
    ~~~~~~~~~^^^^^^
  File "/usr/bin/waydroid-extras", line 95, in install_app
    container.stop()
    ~~~~~~~~~~~~~~^^
  File "/opt/waydroid-script/tools/container.py", line 46, in stop
    run(["waydroid", "container", "stop"])
    ~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/opt/waydroid-script/tools/helper.py", line 48, in run
    raise subprocess.CalledProcessError(
    ...<3 lines>...
    )
subprocess.CalledProcessError: Command '['waydroid', 'container', 'stop']' returned non-zero exit status 0.
```

## 原因分析

报错信息中有一个反常之处：`returned non-zero exit status 0`。在 Unix 系统中，退出码 0 代表**成功**，属于正常情况，并非"非零（non-zero）"。

这是 [waydroid-script](https://github.com/casualsnek/waydroid-script) 中 `tools/helper.py` 的一个 bug。该文件的 `run()` 函数原本意图在命令返回**非零**退出码时抛出异常，但由于使用了 Python 的逻辑取反运算符 `not`，逻辑恰好相反：

```python
# 存在 bug 的代码（示意）
if check and not result.returncode:
    raise subprocess.CalledProcessError(result.returncode, cmd)
```

在 Python 中，`not 0` 求值为 `True`，因此 `not result.returncode` 在退出码为 0（即命令**成功**）时为真，导致脚本在 `waydroid container stop` 正常停止容器后反而抛出异常。

正确的写法应当是：

```python
# 正确的代码
if check and result.returncode:
    raise subprocess.CalledProcessError(result.returncode, cmd)
```

## 解决方案

直接修改本地的 `helper.py` 文件，将错误的判断逻辑改正即可。

打开 `/opt/waydroid-script/tools/helper.py`，找到类似如下的代码：

```python
if check and not result.returncode:
    raise subprocess.CalledProcessError(result.returncode, cmd)
```

将其修改为：

```python
if check and result.returncode:
    raise subprocess.CalledProcessError(result.returncode, cmd)
```

修改完成后，重新运行：

```bash
sudo waydroid-extras install widevine
```

即可正常完成安装。

## 参考

- [casualsnek/waydroid-script](https://github.com/casualsnek/waydroid-script)
