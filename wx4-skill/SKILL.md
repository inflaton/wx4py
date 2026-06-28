---
name: wx4-skill
description: 微信自动化 skill，用于帮助用户快速完成微信消息群发、文件发送、群管理、聊天记录获取、多群监听、AI 自动回复等自动化任务。当用户需要批量发送微信消息、管理微信群、获取聊天记录、监听群消息、或接入 AI 群聊机器人时使用此 skill。
---

# 微信自动化操作 Skill

## ⚠️ 使用前必读：安装检查

**在执行任何操作前，必须先确认 wx4py 库已安装。**

### 步骤1：检查是否已安装

```python
try:
    from wx4py import WeChatClient
    print("✅ wx4py 已安装")
except ImportError:
    print("❌ 未安装 wx4py")
```

### 步骤2：如果未安装，执行安装

```bash
# 方法1：从 PyPI 安装（推荐）
pip install wx4py

# 方法2：从 GitHub 安装（最新版）
pip install git+https://github.com/claw-codes/wx4py.git

# 方法3：从本地安装
cd /path/to/wx4py
pip install .
```

### 步骤3：验证安装成功

```python
from wx4py import WeChatClient
print("✅ 安装成功！")
import wx4py
print(f"版本: {wx4py.__version__}")
```

---

## 概述

本 skill 基于 wx4py 库，帮助用户通过 Python 代码自动化控制 Windows 微信客户端。

**适用场景**：
- 批量群发通知到多个群或联系人
- 定时发送文件、图片、消息
- 获取和分析聊天记录
- 管理群公告、群昵称、免打扰等设置
- 监听多个群聊消息，并在被 @ 时自动回复
- 接入 OpenAI 兼容接口或自定义 AI 回调，构建群聊机器人
- 自动化日常微信操作

**前置要求**：
- Windows 系统
- 微信 PC 客户端已安装并登录（Qt 版本）
- Python >=3.9
- 终端、编辑器、脚本文件统一使用 UTF-8 编码环境

**编码环境建议**：
- Python 源码文件保存为 UTF-8
- PowerShell / 终端优先使用 UTF-8 编码
- 读写文件时显式指定 `encoding='utf-8'`
- 如果出现中文乱码，设置 `$env:PYTHONIOENCODING='utf-8'`

---

## 核心功能

### 1. 消息发送

#### 发送单条消息到联系人或群
```python
from wx4py import WeChatClient

wx = WeChatClient()
wx.connect()
wx.chat_window.send_to("文件传输助手", "测试消息")
wx.chat_window.send_to("工作群", "下午3点开会", target_type='group')
wx.disconnect()
```

#### 批量群发消息
```python
wx = WeChatClient()
wx.connect()
groups = ["群1", "群2", "群3"]
wx.chat_window.batch_send(groups, "重要通知：明天放假")
wx.disconnect()
```

### 2. 文件发送

#### 发送单个文件
```python
wx = WeChatClient()
wx.connect()
wx.chat_window.send_file_to("文件传输助手", r"C:\reports\weekly.pdf")
wx.chat_window.send_file_to("工作群", r"C:\data.xlsx", target_type='group', message='也可以携带消息')
wx.disconnect()
```

### 3. 聊天记录获取
```python
wx = WeChatClient()
wx.connect()
messages = wx.chat_window.get_chat_history(target="工作群", target_type='group', since='today')
for msg in messages:
    print(f"[{msg['time']}] {msg['content']}")
wx.disconnect()
```

### 4. 群管理

```python
wx = WeChatClient()
wx.connect()
members = wx.group_manager.get_group_members("工作群")
wx.group_manager.set_group_nickname("工作群", "张三的小号")
wx.group_manager.set_do_not_disturb("工作群", enable=True)
wx.group_manager.set_pin_chat("工作群", enable=True)
wx.group_manager.set_announcement_from_markdown("工作群", "markdown文件路径")
wx.group_manager.modify_announcement_simple("工作群", "新公告内容")
wx.disconnect()
```

### 5. 群聊监听与 AI 自动回复

```python
from wx4py import AsyncCallbackHandler, MessageEvent, WeChatClient

groups = ["工作群1", "工作群2"]
def reply(event: MessageEvent):
    if not event.is_at_me:
        return ""
    return f"收到：{event.content}"

with WeChatClient(auto_connect=True) as wx:
    wx.process_groups(groups, [AsyncCallbackHandler(reply, auto_reply=True, reply_on_at=True)], block=True)
```

详见现有文档中的更多示例（定时群发、转发规则、AI 接入等）。

---

## ⚡ 关键实战经验：UIA 不稳定性及应对方案

### 问题：微信 UIA 辅助功能树不稳定

wx4py 底层依赖 Windows UIAutomation，但微信（Qt）的 UIA 树**时好时坏**：

- **健康状态**：100+ 节点，可正常搜索、发送
- **故障状态**：仅 2 个节点（`mmui::MainWindow` + `MMUIRenderSubWindowHW`），搜索框不可用，所有操作失败

**故障原因**：Qt 辅助功能桥未正确加载。微信启动时读取系统屏幕阅读器标志，若当时未开启，UIA 树就不完整。

### 应对策略

**方案一：确保窗口在前台再重试**
```python
import time, win32gui, win32con

# 恢复微信窗口
hwnd_list = []
def cb(hwnd, _):
    title = win32gui.GetWindowText(hwnd)
    cls = win32gui.GetClassName(hwnd)
    if cls.startswith('Qt515') and ('WeChat' in title or 'Weixin' in title):
        hwnd_list.append(hwnd)
    return True
win32gui.EnumWindows(cb, None)

for hwnd in hwnd_list:
    win32gui.ShowWindow(hwnd, win32con.SW_RESTORE)
try:
    win32gui.SetForegroundWindow(hwnd_list[0])
except:
    pass
time.sleep(1)
```

**方案二：绕过 WeChatClient，直接连 UIA**
```python
from wx4py.core.uia_wrapper import UIAWrapper

# 先找 HWND
uia = UIAWrapper(135590)  # 替换为实际 HWND
root = uia.root
```

**方案三：多次重试直到树健康**
```python
def get_healthy_uia(hwnd, max_attempts=5):
    for i in range(max_attempts):
        try:
            win32gui.ShowWindow(hwnd, win32con.SW_RESTORE)
            time.sleep(0.5)
            uia = UIAWrapper(hwnd)
            root = uia.root
            count = 0
            stack = [(root, 0)]
            while stack and count < 300:
                n, d = stack.pop()
                count += 1
                if d < 15:
                    try:
                        for ch in n.GetChildren():
                            stack.append((ch, d + 1))
                    except:
                        pass
            if count >= 50:
                return uia
        except:
            pass
        time.sleep(1)
    return None
```

**方案四：重启微信**（最彻底的修复，但需要重新扫码登录）
```python
from wx4py.core.win32 import restart_wechat_process, find_wechat_window
hwnd = find_wechat_window()
if hwnd:
    restart_wechat_process(hwnd)
```

---

## 📋 新功能：枚举全部联系人

### 方法一：从 UIA 通讯录面板读取（推荐）

可直接从微信通讯录面板的 UIA 树中读取 `mmui::ContactsCellItemView` 控件。

```python
import time, win32gui, win32con
from wx4py.core.uia_wrapper import UIAWrapper

def get_all_contacts(hwnd):
    """从微信通讯录面板读取所有联系人（不含内置账号）"""
    win32gui.ShowWindow(hwnd, win32con.SW_RESTORE)
    time.sleep(0.5)
    uia = UIAWrapper(hwnd)
    root = uia.root

    # 点击 Contacts 标签页
    def find_and_click(root, name_part):
        stack = [(root, 0)]
        while stack:
            n, d = stack.pop()
            if d > 12: continue
            try:
                if name_part in (n.Name or "") and n.ControlTypeName == 'ButtonControl':
                    n.Click()
                    return True
                for ch in n.GetChildren():
                    stack.append((ch, d + 1))
            except:
                pass
        return False

    find_and_click(root, 'Contacts')
    time.sleep(2)

    # 读取通讯录控件
    contacts = []
    stack = [(root, 0)]
    while stack:
        n, d = stack.pop()
        if d > 20: continue
        try:
            cls = n.ClassName or ''
            name = (n.Name or '').strip()
            if cls == 'mmui::ContactsCellItemView' and name and len(name) >= 2:
                contacts.append(name)
            for ch in n.GetChildren():
                stack.append((ch, d + 1))
        except:
            pass

    builtin = {'文件传输助手', '微信团队'}
    return [c for c in contacts if c not in builtin]
```

### 方法二：通过搜索枚举（备选）

```python
from wx4py import WeChatClient
SURNAMES = "王李张刘陈杨黄赵周吴徐孙马朱胡林郭何高罗"
all_contacts = set()
with WeChatClient() as wx:
    for ch in SURNAMES:
        try:
            results = wx.chat_window.search(ch)
            for gname, items in results.items():
                for item in items:
                    if item.name.strip():
                        all_contacts.add(item.name.strip())
        except:
            pass
```

---

## 📤 新功能：给所有联系人发消息（群发广播）

### 完整工作流

由于 wx4py 的 `send_to()` 在搜索联系人时如果联系人被分到"未知"组（而不是"联系人"组）会报 `not found`，
需要通过点击搜索结果直接打开聊天再发送。

```python
import time
from wx4py import WeChatClient
from wx4py.features.chat import ChatWindow

def send_to_all_contacts(contacts, message, prefix_name=True):
    """
    向所有联系人发送消息。

    Args:
        contacts: list[str] — 联系人名称列表
        message: str — 要发送的消息
        prefix_name: bool — 是否在消息前加上对方名字（默认 True）
                       发送效果："{联系人名称}\n{消息内容}"

    原理：对每个联系人，搜索 → 点击搜索结果 → 粘贴消息 → 发送。
    """
    with WeChatClient() as wx:
        for contact in contacts:
            try:
                # 搜索联系人
                results = wx.chat_window.search(contact)

                # 在任意分组中查找（不限于"联系人"分组）
                target_item = None
                for gname, items in results.items():
                    for item in items:
                        if contact in item.name:
                            target_item = item
                            break
                    if target_item:
                        break

                if not target_item:
                    print(f"  ⚠️ 未找到: {contact}")
                    continue

                # 直接点击搜索结果
                try:
                    target_item.ctrl.Click()
                except Exception:
                    target_item.ctrl.Click(simulateMove=False)
                time.sleep(1.5)

                # 组装个性化消息（默认在消息前加上对方名字）
                personalized = f"{contact}，您好！\n{message}" if prefix_name else message

                # 发送消息
                chat_input = wx.chat_window._get_chat_input()
                if chat_input:
                    if ChatWindow.send_text_via_input(chat_input, personalized):
                        print(f"  ✅ {contact}")
                    else:
                        print(f"  ❌ 发送失败: {contact}")
                else:
                    print(f"  ❌ 未找到输入框: {contact}")

            except Exception as e:
                print(f"  ❌ 错误: {contact} - {e}")


# 使用示例（默认带对方名字前缀）
contacts = ["东浩（法越）", "Jeff", "Flora"]
send_to_all_contacts(contacts, "群发消息测试")
# 每人收到："张三\n群发消息测试"

# 如需群发相同内容不加名字：
# send_to_all_contacts(contacts, "群发消息测试", prefix_name=False)
```

### 完整脚本模板

```python
# -*- coding: utf-8 -*-
"""给所有联系人发消息"""
from wx4py import WeChatClient
from wx4py.features.chat import ChatWindow

# Step 1: 读取联系人列表
contacts = ["联系人1", "联系人2"]  # 或用 get_all_contacts() 自动获取

message = "你好，这是一条群发消息。"

# Step 2: 发送（默认带对方名称）
send_to_all_contacts(contacts, message)
# 每人收到："联系人名称\n你好，这是一条群发消息。"

# 如需禁用名称前缀：
# send_to_all_contacts(contacts, message, prefix_name=False)
```

### 识别群聊 vs 个人联系人的启发式方法

从聊天列表获取的条目可用以下关键词识别群聊：

```python
GROUP_KEYWORDS = ['群', '小聚', '讨论', '校友', '实验室', '家']
is_group = any(kw in name for kw in GROUP_KEYWORDS)
```

---

## 🔧 Win32 实用辅助函数

在 UIA 连接不稳定时，以下 win32 操作可以帮助恢复：

```python
import win32gui, win32con, win32api

def restore_wechat_window():
    """恢复微信窗口到前台"""
    hwnds = []
    def cb(hwnd, _):
        title = win32gui.GetWindowText(hwnd)
        cls = win32gui.GetClassName(hwnd)
        if cls.startswith('Qt515') and ('WeChat' in title or 'Weixin' in title):
            hwnds.append(hwnd)
        return True
    win32gui.EnumWindows(cb, None)
    for hwnd in hwnds:
        win32gui.ShowWindow(hwnd, win32con.SW_RESTORE)
    if hwnds:
        try:
            win32gui.SetForegroundWindow(hwnds[0])
        except:
            pass

def press_esc(times=3):
    """按 Esc 键关闭可能的弹窗"""
    for _ in range(times):
        win32api.keybd_event(win32con.VK_ESCAPE, 0, 0, 0)
        time.sleep(0.08)
        win32api.keybd_event(win32con.VK_ESCAPE, 0, win32con.KEYEVENTF_KEYUP, 0)
        time.sleep(0.2)

def press_ctrl_key(key_code):
    """模拟 Ctrl+<key>"""
    win32api.keybd_event(win32con.VK_CONTROL, 0, 0, 0)
    win32api.keybd_event(key_code, 0, 0, 0)
    time.sleep(0.05)
    win32api.keybd_event(key_code, 0, win32con.KEYEVENTF_KEYUP, 0)
    win32api.keybd_event(win32con.VK_CONTROL, 0, win32con.KEYEVENTF_KEYUP, 0)
```

---

## ⚠️ 已知限制

### WeChat 4.x (Weixin.exe) 特殊问题

- 微信进程名为 **Weixin.exe** 而非 `WeChat.exe`
- **PyWxDump 不可用**：该库最高仅支持到 3.9.x，4.1.x 无偏移数据
- 本地 `contact.db` 加密，无法直接读取
- 联系人搜索被归类到"未知"组而非"联系人"组，导致 `send_to()` 失败

### 通用限制

- 仅支持 Windows 系统
- 需要微信客户端已登录
- 操作期间不要手动操作微信窗口
- 受微信 UIA 限制，聊天记录无法获取发送者姓名

---

## 使用模式

### 推荐：使用上下文管理器
```python
from wx4py import WeChatClient
with WeChatClient() as wx:
    wx.chat_window.send_to("文件传输助手", "Hello!")
```

### 手动连接管理
```python
wx = WeChatClient()
wx.connect()
try:
    wx.chat_window.send_to("文件传输助手", "Hello!")
finally:
    wx.disconnect()
```

## 快速参考

| 操作 | 方法 | 示例 |
|------|------|------|
| 发消息给联系人 | `chat_window.send_to(target, message)` | `wx.chat_window.send_to("张三", "Hi")` |
| 发消息到群 | `chat_window.send_to(target, message, target_type='group')` | `wx.chat_window.send_to("工作群", "Hi", target_type='group')` |
| 批量群发 | `chat_window.batch_send(targets, message, target_type)` | `wx.chat_window.batch_send(["群1", "群2"], "Hi", target_type='group')` |
| 发送文件 | `chat_window.send_file_to(target, file_path)` | `wx.chat_window.send_file_to("张三", r"C:\file.pdf")` |
| 获取聊天记录 | `chat_window.get_chat_history(target, target_type, since)` | `wx.chat_window.get_chat_history("工作群", 'group', 'today')` |
| 获取群成员 | `group_manager.get_group_members(group_name)` | `wx.group_manager.get_group_members("工作群")` |
| **枚举全部联系人** | `get_all_contacts(hwnd)` | 见上"新功能"章节 |
| **群发广播（带名字）** | `send_to_all_contacts(contacts, msg, prefix_name=True)` | 默认在消息前加对方名字 |

## 常见问题

**Q: 脚本运行时提示找不到微信窗口？**
A: 确保微信已打开并登录，窗口不要最小化。可尝试 `restore_wechat_window()`。

**Q: UIA 树只有 2 个节点？**
A: 尝试 `restore_wechat_window()` → `press_esc(5)` → 重试。最彻底的方法是重启微信。

**Q: `send_to()` 报 "not found" 但联系人明明存在？**
A: 微信 4.x 将联系人归类到"未知"组。改用搜索→直接点击的方案（即 `send_to_all_contacts`）。

**Q: 中文字符出现乱码？**
A: 设置 `$env:PYTHONIOENCODING='utf-8'`，文件保存为 UTF-8。
