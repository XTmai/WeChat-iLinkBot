# 微信 iLink Bot API 开发文档

微信 iLink（智联）协议是微信官方开放的 Bot API，允许开发者通过 HTTP/JSON 接口接入个人微信，实现消息收发功能。

## 目录

- [概述](#概述)
- [快速开始](#快速开始)
- [认证与登录](#认证与登录)
- [消息接收](#消息接收)
- [消息发送](#消息发送)
- [媒体文件处理](#媒体文件处理)
- [高级功能](#高级功能)
- [错误处理](#错误处理)
- [完整示例](#完整示例)

---

## 概述

### 功能特性

- **私聊对话**：用户与 Bot 一对一交流
- **多媒体消息**：文本、图片、语音、文件、视频
- **输入状态**：显示"正在输入..."提示
- **长轮询机制**：无需 WebSocket，简单的 HTTP 轮询

### 架构概览

```
┌─────────────┐      ┌─────────────────┐      ┌─────────────┐
│  你的 Bot   │◄────►│  iLink Bot API  │◄────►│  微信 CDN   │
│  应用服务   │      │ ilinkai.weixin.qq.com   │  novac2c... │
└─────────────┘      └─────────────────┘      └─────────────┘
                              │
                              ▼
                       ┌─────────────┐
                       │   微信客户端  │
                       │  (扫码授权)   │
                       └─────────────┘
```

### 核心端点

| Endpoint | Method | 功能 |
|----------|--------|------|
| `/ilink/bot/get_bot_qrcode` | GET | 获取登录二维码 |
| `/ilink/bot/get_qrcode_status` | GET | 轮询扫码状态 |
| `/ilink/bot/getupdates` | POST | 长轮询收消息 |
| `/ilink/bot/sendmessage` | POST | 发送消息 |
| `/ilink/bot/getuploadurl` | POST | 获取 CDN 上传地址 |
| `/ilink/bot/getconfig` | POST | 获取 Bot 配置 |
| `/ilink/bot/sendtyping` | POST | 发送输入状态 |

---

## 快速开始

### 安装依赖

```bash
pip install cryptography Pillow qrcode requests
```

### 最小示例

```python
from weixin_push.client import WeixinClient
from weixin_push.store import save_session

client = WeixinClient()

# 1. 扫码登录
session = client.wait_for_login()
save_session(session)
print(f"登录成功! Bot: {session.account_id}")

# 2. 监听消息
from weixin_push.store import load_contacts, upsert_contact
from weixin_push.client import extract_text
import time

get_updates_buf = ""
while True:
    resp = client.get_updates(session.token, get_updates_buf=get_updates_buf)
    if resp.get("get_updates_buf"):
        get_updates_buf = resp["get_updates_buf"]

    for msg in resp.get("msgs", []):
        if msg.get("message_type") != 1:
            continue
        from_user_id = msg.get("from_user_id", "")
        context_token = msg.get("context_token", "")
        text = extract_text(msg)
        print(f"收到: {from_user_id}: {text}")

        # 回复文本
        client.send_text(session.token, from_user_id, f"收到: {text}", context_token)

        # 缓存联系人
        if from_user_id and context_token:
            upsert_contact(from_user_id, context_token=context_token,
                         last_text=text, last_message_id=msg.get("message_id"),
                         last_seen_at=time.strftime("%Y-%m-%dT%H:%M:%S"))
```

---

## 认证与登录

### 扫码登录流程

**Step 1: 获取二维码**

```bash
GET https://ilinkai.weixin.qq.com/ilink/bot/get_bot_qrcode?bot_type=3
```

返回：
```json
{
  "ret": 0,
  "qrcode": "https://ilinkai.weixin.qq.com/ilink/bot/qrcode?...",
  "expire_time": 120
}
```

**Step 2: 轮询扫码状态**

```bash
GET https://ilinkai.weixin.qq.com/ilink/bot/get_qrcode_status?qrcode=<qrcode>
```

状态流转：`wait` → `scanned` → `confirmed` 或 `expired`

**Step 3: 确认后获取 Token**

确认授权后返回：
```json
{
  "ret": 0,
  "status": "confirmed",
  "bot_token": "xxx",
  "ilink_bot_id": "xxx@im.bot",
  "ilink_user_id": "xxx@im.wechat",
  "baseurl": "https://ilinkai.weixin.qq.com"
}
```

### 保存会话

```python
from dataclasses import dataclass

@dataclass
class Session:
    token: str
    base_url: str
    account_id: str
    user_id: str
    saved_at: str

# 保存到文件
import json
def save_session(session: Session):
    with open("session.json", "w") as f:
        json.dump({
            "token": session.token,
            "base_url": session.base_url,
            "account_id": session.account_id,
            "user_id": session.user_id,
            "saved_at": session.saved_at,
        }, f, indent=2)
```

### 复用 Token

```python
# 加载保存的 token
import json
with open("session.json") as f:
    data = json.load(f)

session = Session(**data)
client = WeixinClient(base_url=session.base_url)
# 直接使用 session.token 进行 API 调用
```

---

## 消息接收

### 长轮询机制

iLink 使用长轮询接收消息，请求会在服务器端等待直到有新消息或超时。

```bash
POST https://ilinkai.weixin.qq.com/ilink/bot/getupdates
Headers:
  Content-Type: application/json
  AuthorizationType: ilink_bot_token
  Authorization: Bearer <bot_token>
  X-WECHAT-UIN: <base64_random_uint32>

Body:
{
  "get_updates_buf": "",
  "base_info": { "channel_version": "1.0.2" }
}
```

**重要**：`get_updates_buf` 是同步游标，收到后原样保存，下次请求原样回传，**不要解析或修改它**。

### 响应消息结构

```json
{
  "ret": 0,
  "get_updates_buf": "xxx",
  "msgs": [
    {
      "msg_id": "xxx",
      "from_user_id": "user@im.wechat",
      "to_user_id": "bot@im.bot",
      "message_type": 1,
      "message_state": 2,
      "context_token": "xxx",
      "create_time_ms": 1234567890000,
      "item_list": [
        { "type": 1, "text_item": { "text": "消息内容" } }
      ]
    }
  ]
}
```

### MessageItem 类型

| type | 内容类型 | 说明 |
|------|----------|------|
| 1 | 文本 | `text_item.text` |
| 2 | 图片 | `image_item.media`, `image_item.thumb_media` |
| 3 | 语音 | `voice_item.media`, `voice_item.text` |
| 4 | 文件 | `file_item.file_name`, `file_item.media` |
| 5 | 视频 | `video_item.media`, `video_item.thumb_media` |

### 提取文本内容

```python
def extract_text(message: dict) -> str:
    """从消息 item_list 中提取可读文本"""
    for item in message.get("item_list", []):
        item_type = item.get("type")
        if item_type == 1 and item.get("text_item", {}).get("text"):
            return item["text_item"]["text"]
        if item_type == 3 and item.get("voice_item", {}).get("text"):
            return f"[语音] {item['voice_item']['text']}"
        if item_type == 2:
            return "[图片]"
        if item_type == 4:
            file_name = item.get("file_item", {}).get("file_name", "")
            return f"[文件] {file_name}".strip()
        if item_type == 5:
            return "[视频]"
    return "[空消息]"
```

---

## 消息发送

### 发送文本消息

```bash
POST https://ilinkai.weixin.qq.com/ilink/bot/sendmessage
Headers:
  Content-Type: application/json
  AuthorizationType: ilink_bot_token
  Authorization: Bearer <bot_token>
  X-WECHAT-UIN: <base64_random_uint32>
  Content-Length: <body_length>

Body:
{
  "msg": {
    "to_user_id": "user@im.wechat",
    "client_id": "py-xxx",
    "message_type": 2,
    "message_state": 2,
    "context_token": "<从入站消息获取>",
    "item_list": [
      { "type": 1, "text_item": { "text": "hello" } }
    ]
  },
  "base_info": { "channel_version": "1.0.2" }
}
```

**关键字段说明**：

| 字段 | 值 | 说明 |
|------|-----|------|
| `message_type` | 2 | 2=BOT 发送的消息 |
| `message_state` | 2 | 2=FINISH，消息发送完成 |
| `context_token` | string | **必须**，从入站消息中获取 |
| `client_id` | string | 客户端生成的唯一 ID，用于去重 |

**重要**：`context_token` 必须原样带回，否则消息不会关联到正确的对话窗口。

### Python 实现

```python
import uuid
import json
import urllib.request

class WeixinClient:
    def __init__(self, base_url: str = "https://ilinkai.weixin.qq.com", timeout: float = 15.0):
        self.base_url = base_url
        self.timeout = timeout

    def _headers(self, token: str = None, body: bytes = None) -> dict:
        headers = {
            "Content-Type": "application/json",
            "AuthorizationType": "ilink_bot_token",
        }
        if token:
            headers["Authorization"] = f"Bearer {token}"
        if body is not None:
            headers["Content-Length"] = str(len(body))
        return headers

    def _post_json(self, endpoint: str, payload: dict, token: str = None) -> dict:
        body = json.dumps(payload, ensure_ascii=False).encode("utf-8")
        url = f"{self.base_url}/{endpoint}"
        request = urllib.request.Request(
            url, data=body,
            headers=self._headers(token=token, body=body),
            method="POST"
        )
        with urllib.request.urlopen(request, timeout=self.timeout) as response:
            return json.loads(response.read().decode("utf-8"))

    def send_text(self, token: str, to_user_id: str, text: str, context_token: str) -> str:
        client_id = f"py-{uuid.uuid4()}"
        self._post_json(
            "ilink/bot/sendmessage",
            {
                "msg": {
                    "from_user_id": "",
                    "to_user_id": to_user_id,
                    "client_id": client_id,
                    "message_type": 2,
                    "message_state": 2,
                    "context_token": context_token,
                    "item_list": [{"type": 1, "text_item": {"text": text}}],
                }
            },
            token=token,
        )
        return client_id
```

---

## 媒体文件处理

### 发送图片流程

```
┌─────────┐  getuploadurl   ┌─────────┐
│  Client │ ───────────────► │  iLink  │
│         │ ◄─────────────── │   API   │
└─────────┘   upload_param   └─────────┘
                                    │
              ┌─────────────────────┘
              │ 计算 MD5/密文大小
              │ 生成 16 字节 AES key
              ▼
┌─────────┐  AES-128-ECB 加密
│  本地   │
└─────────┘
              │ POST /upload
              ▼
┌─────────┐  CDN 上传       ┌─────────┐
│  CDN   │ ───────────────► │  返回    │
└─────────┘ ◄─────────────── │ x-encrypted-param   │
                              └─────────┘
                                    │
              ┌─────────────────────┘
              │ sendmessage 引用媒体
              ▼
        ┌─────────┐
        │  iLink  │
        │   API   │
        └─────────┘
```

### Step 1: 获取上传地址

```python
import hashlib
import secrets

filekey = secrets.token_hex(16)  # 32 字符随机字符串
aeskey = secrets.token_bytes(16)  # 16 字节随机密钥

# 计算文件信息
plaintext = Path(file_path).read_bytes()
rawsize = len(plaintext)
rawfilemd5 = hashlib.md5(plaintext).hexdigest()
filesize = aes_ecb_padded_size(rawsize)

upload_resp = client.get_upload_url(
    token,
    filekey=filekey,
    media_type=1,  # 1=图片 2=视频 3=文件 4=语音
    to_user_id=to_user_id,
    rawsize=rawsize,
    rawfilemd5=rawfilemd5,
    filesize=filesize,
    aeskey_hex=aeskey.hex(),
    no_need_thumb=True,  # 不需要缩略图
)
```

### Step 2: AES-128-ECB 加密

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

def aes_ecb_padded_size(plaintext_size: int) -> int:
    """计算 PKCS7 填充后的密文长度"""
    return ((plaintext_size // 16) + 1) * 16

def encrypt_aes_ecb(plaintext: bytes, key: bytes) -> bytes:
    """AES-128-ECB + PKCS7 填充加密"""
    pad_len = 16 - (len(plaintext) % 16)
    padded = plaintext + bytes([pad_len]) * pad_len
    cipher = Cipher(algorithms.AES(key), modes.ECB(), backend=default_backend())
    encryptor = cipher.encryptor()
    return encryptor.update(padded) + encryptor.finalize()

ciphertext = encrypt_aes_ecb(plaintext, aeskey)
```

### Step 3: 上传到 CDN

```python
import urllib.parse
import urllib.request

def upload_to_cdn(ciphertext: bytes, upload_param: str, filekey: str, aeskey: bytes) -> str:
    query = urllib.parse.urlencode({
        "encrypted_query_param": upload_param,
        "filekey": filekey,
    })
    url = f"https://novac2c.cdn.weixin.qq.com/c2c/upload?{query}"

    request = urllib.request.Request(
        url,
        data=ciphertext,
        headers={"Content-Type": "application/octet-stream"},
        method="POST"
    )
    with urllib.request.urlopen(request, timeout=30) as response:
        encrypted_param = response.headers.get("x-encrypted-param")
        if not encrypted_param:
            raise Exception("CDN 上传成功但未返回 x-encrypted-param")
        return encrypted_param

download_param = upload_to_cdn(ciphertext, upload_param, filekey, aeskey)
```

### Step 4: 发送图片引用消息

```python
import base64

# 关键：aes_key 必须先转 hex 字符串，再 base64 编码
aes_key_base64 = base64.b64encode(aeskey.hex().encode("utf-8")).decode("utf-8")

payload = {
    "msg": {
        "to_user_id": to_user_id,
        "client_id": f"py-{uuid.uuid4()}",
        "message_type": 2,
        "message_state": 2,
        "context_token": context_token,
        "item_list": [{
            "type": 2,
            "image_item": {
                "media": {
                    "encrypt_query_param": download_param,
                    "aes_key": aes_key_base64,
                    "encrypt_type": 1,
                }
            }
        }]
    },
    "base_info": { "channel_version": "1.0.2" }
}

client._post_json("ilink/bot/sendmessage", payload, token=token)
```

### 发送文件

发送文件与图片类似，只是 `media_type` 不同：

```python
# media_type: 1=图片 2=视频 3=文件 4=语音
uploaded = client._upload_file(token, to_user_id, file_path, media_type=3)

media_item = {
    "type": 4,
    "file_item": {
        "media": {
            "encrypt_query_param": uploaded["download_param"],
            "aes_key": aes_key_base64,
            "encrypt_type": 1,
        },
        "file_name": os.path.basename(file_path),
        "len": str(uploaded["file_size"]),
    },
}
```

---

## 高级功能

### 输入状态指示

在 AI 生成回复期间，可以显示"正在输入..."状态。

**Step 1: 获取 typing_ticket**

```python
config = client.get_config(token, to_user_id, context_token)
typing_ticket = config.get("typing_ticket")
```

**Step 2: 发送输入状态**

```python
client.send_typing(token, to_user_id, typing_ticket, status=1)  # 显示"正在输入"
# ... AI 处理中 ...
client.send_typing(token, to_user_id, typing_ticket, status=2)  # 取消输入状态
```

### 断点续传

将 `get_updates_buf` 持久化保存，程序重启后可继续从上次位置接收消息，避免重复处理。

```python
from pathlib import Path

BUF_FILE = Path("sync_buf.dat")

# 启动时读取
get_updates_buf = BUF_FILE.read_text() if BUF_FILE.exists() else ""

# 运行中保存
resp = client.get_updates(token, get_updates_buf=get_updates_buf)
if resp.get("get_updates_buf"):
    get_updates_buf = resp["get_updates_buf"]
    BUF_FILE.write_text(get_updates_buf)
```

---

## 错误处理

### 常见错误码

| ret | 说明 | 处理方式 |
|-----|------|----------|
| 0 | 成功 | - |
| -1 | 参数错误 | 检查请求参数 |
| -2 | Token 无效 | 重新登录 |
| -14 | 会话过期 | 重新扫码登录 |
| -22 | 权限不足 | 检查 Bot 权限 |

### 异常处理示例

```python
class WeixinApiError(RuntimeError):
    """微信 iLink Bot API 错误"""
    def __init__(self, message: str):
        super().__init__(message)

def handle_api_error(func):
    """API 错误处理装饰器"""
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except WeixinApiError as e:
            error_msg = str(e)
            if "超时" in error_msg:
                print("请求超时，稍后重试")
            elif "-14" in error_msg or "会话过期" in error_msg:
                print("会话过期，需要重新登录")
            else:
                print(f"API 错误: {e}")
    return wrapper
```

### 监听异常处理

```python
while listening:
    try:
        resp = client.get_updates(token, get_updates_buf=get_updates_buf)
        # 处理消息...
    except WeixinApiError as e:
        if "会话过期" in str(e):
            print("会话过期，请重新登录")
            listening = False
        else:
            print(f"监听异常: {e}")
            time.sleep(3)  # 退避后重试
    except Exception as e:
        print(f"未知异常: {e}")
        time.sleep(3)
```

---

## 完整示例

### 命令行工具完整实现

```python
#!/usr/bin/env python3
"""
微信 iLink Bot 命令行工具
支持：扫码登录、消息监听、发送文本/文件/图片
"""

from __future__ import annotations
import argparse
import datetime as dt
import os
import sys
import time
import base64
import hashlib
import json
import secrets
import socket
import tempfile
import urllib.error
import urllib.parse
import urllib.request
import uuid
from dataclasses import dataclass
from io import BytesIO
from pathlib import Path
from typing import Any

try:
    from cryptography.hazmat.backends import default_backend
    from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
    from PIL import Image
    import qrcode
except ImportError:
    print("请先安装依赖: pip install cryptography Pillow qrcode")
    sys.exit(1)


DEFAULT_BASE_URL = "https://ilinkai.weixin.qq.com"
CDN_BASE_URL = "https://novac2c.cdn.weixin.qq.com/c2c"
CHANNEL_VERSION = "1.0.2"


def random_wechat_uin() -> str:
    """生成请求头所需的 X-WECHAT-UIN"""
    value = secrets.randbelow(2**32)
    return base64.b64encode(str(value).encode("utf-8")).decode("utf-8")


def extract_text(message: dict[str, Any]) -> str:
    """从消息 item_list 中提取可读文本"""
    for item in message.get("item_list") or []:
        item_type = item.get("type")
        if item_type == 1 and item.get("text_item", {}).get("text"):
            return item["text_item"]["text"]
        if item_type == 3 and item.get("voice_item", {}).get("text"):
            return f"[语音] {item['voice_item']['text']}"
        if item_type == 2:
            return "[图片]"
        if item_type == 4:
            file_name = item.get("file_item", {}).get("file_name", "")
            return f"[文件] {file_name}".strip()
        if item_type == 5:
            return "[视频]"
    return "[空消息]"


def aes_ecb_padded_size(plaintext_size: int) -> int:
    return ((plaintext_size // 16) + 1) * 16


def encrypt_aes_ecb(plaintext: bytes, key: bytes) -> bytes:
    pad_len = 16 - (len(plaintext) % 16)
    padded = plaintext + bytes([pad_len]) * pad_len
    cipher = Cipher(algorithms.AES(key), modes.ECB(), backend=default_backend())
    encryptor = cipher.encryptor()
    return encryptor.update(padded) + encryptor.finalize()


def guess_mime_type(file_path: str) -> str:
    import mimetypes
    mime, _ = mimetypes.guess_type(file_path)
    return mime or "application/octet-stream"


def print_terminal_qr(content: str) -> None:
    qr = qrcode.QRCode(border=2)
    qr.add_data(content)
    qr.make(fit=True)
    matrix = qr.get_matrix()
    for row in matrix:
        print("".join("██" if cell else "  " for cell in row))


@dataclass
class Session:
    token: str
    base_url: str
    account_id: str
    user_id: str
    saved_at: str


class WeixinApiError(RuntimeError):
    pass


class WeixinClient:
    def __init__(self, base_url: str = DEFAULT_BASE_URL, timeout: float = 15.0):
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout
        self.cdn_base_url = CDN_BASE_URL.rstrip("/")

    def _headers(self, token: str = None, body: bytes = None) -> dict[str, str]:
        headers = {
            "Content-Type": "application/json",
            "AuthorizationType": "ilink_bot_token",
            "X-WECHAT-UIN": random_wechat_uin(),
        }
        if token:
            headers["Authorization"] = f"Bearer {token}"
        if body is not None:
            headers["Content-Length"] = str(len(body))
        return headers

    def _get_json(self, path: str, timeout: float = None) -> dict[str, Any]:
        url = f"{self.base_url}/{path.lstrip('/')}"
        request = urllib.request.Request(url, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=timeout or self.timeout) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as exc:
            raise WeixinApiError(f"HTTP {exc.code}: {exc.read().decode()}") from exc
        except (TimeoutError, socket.timeout) as exc:
            raise WeixinApiError("请求超时") from exc
        except urllib.error.URLError as exc:
            raise WeixinApiError(f"请求失败: {exc}") from exc

    def _post_json(self, endpoint: str, payload: dict[str, Any], token: str = None, timeout: float = None) -> dict[str, Any]:
        body = json.dumps({**payload, "base_info": {"channel_version": CHANNEL_VERSION}}, ensure_ascii=False).encode("utf-8")
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        request = urllib.request.Request(url, data=body, headers=self._headers(token=token, body=body), method="POST")
        try:
            with urllib.request.urlopen(request, timeout=timeout or self.timeout) as response:
                raw = response.read().decode("utf-8")
                return json.loads(raw) if raw else {}
        except urllib.error.HTTPError as exc:
            raise WeixinApiError(f"HTTP {exc.code}: {exc.read().decode()}") from exc
        except (TimeoutError, socket.timeout) as exc:
            raise WeixinApiError("请求超时") from exc
        except urllib.error.URLError as exc:
            raise WeixinApiError(f"请求失败: {exc}") from exc

    def start_login(self) -> dict[str, Any]:
        return self._get_json(f"ilink/bot/get_bot_qrcode?bot_type=3")

    def check_qr_status(self, qrcode: str) -> dict[str, Any]:
        return self._get_json(f"ilink/bot/get_qrcode_status?qrcode={qrcode}")

    def wait_for_login(self, poll_interval: float = 1.0, deadline_seconds: int = 300) -> Session:
        qr_resp = self.start_login()
        qrcode = qr_resp["qrcode"]
        qrcode_url = qr_resp.get("qrcode_img_content", "")

        print("请用微信扫描下方二维码完成登录：")
        if qrcode_url:
            try:
                print_terminal_qr(qrcode_url)
            except Exception:
                print(qrcode_url)
        else:
            print(qrcode)

        deadline = time.time() + deadline_seconds
        refresh_count = 0

        while time.time() < deadline:
            try:
                status = self.check_qr_status(qrcode)
            except WeixinApiError as exc:
                if "超时" in str(exc):
                    print("\n状态轮询超时，继续重试...")
                    time.sleep(poll_interval)
                    continue
                raise

            state = status.get("status")
            if state == "wait":
                print(".", end="", flush=True)
            elif state == "scaned":
                print("\n已扫码，请在手机上确认。")
            elif state == "expired":
                refresh_count += 1
                if refresh_count > 3:
                    raise WeixinApiError("二维码多次过期，请重新执行 login。")
                qr_resp = self.start_login()
                qrcode = qr_resp["qrcode"]
                print(f"\n二维码已过期，已刷新 ({refresh_count}/3)：")
                try:
                    print_terminal_qr(qr_resp.get("qrcode_img_content", ""))
                except Exception:
                    print(qrcode)
            elif state == "confirmed":
                print("\n登录成功。")
                return Session(
                    token=status["bot_token"],
                    base_url=status.get("baseurl") or self.base_url,
                    account_id=status["ilink_bot_id"],
                    user_id=status["ilink_user_id"],
                    saved_at=time.strftime("%Y-%m-%dT%H:%M:%S%z"),
                )
            time.sleep(poll_interval)
        raise WeixinApiError("登录超时，请重新执行 login。")

    def get_updates(self, token: str, get_updates_buf: str = "", timeout_seconds: float = 38.0) -> dict[str, Any]:
        return self._post_json(
            "ilink/bot/getupdates",
            {"get_updates_buf": get_updates_buf},
            token=token,
            timeout=timeout_seconds,
        )

    def get_upload_url(self, token: str, **kwargs) -> dict[str, Any]:
        return self._post_json("ilink/bot/getuploadurl", kwargs, token=token)

    def _cdn_upload(self, plaintext: bytes, upload_param: str, filekey: str, aeskey: bytes) -> str:
        ciphertext = encrypt_aes_ecb(plaintext, aeskey)
        query = urllib.parse.urlencode({"encrypted_query_param": upload_param, "filekey": filekey})
        url = f"{self.cdn_base_url}/upload?{query}"
        for _ in range(3):
            request = urllib.request.Request(url, data=ciphertext, headers={"Content-Type": "application/octet-stream"}, method="POST")
            try:
                with urllib.request.urlopen(request, timeout=self.timeout) as response:
                    encrypted_param = response.headers.get("x-encrypted-param")
                    if not encrypted_param:
                        raise WeixinApiError("CDN 上传成功但未返回 x-encrypted-param")
                    return encrypted_param
            except urllib.error.HTTPError as exc:
                if 400 <= exc.code < 500:
                    raise WeixinApiError(f"CDN 上传失败 HTTP {exc.code}") from exc
            time.sleep(1)
        raise WeixinApiError("CDN 上传失败")

    def _upload_file(self, token: str, to_user_id: str, file_path: str, media_type: int) -> dict[str, Any]:
        plaintext = Path(file_path).read_bytes()
        rawsize = len(plaintext)
        rawfilemd5 = hashlib.md5(plaintext).hexdigest()
        filesize = aes_ecb_padded_size(rawsize)
        filekey = secrets.token_hex(16)
        aeskey = secrets.token_bytes(16)

        upload_resp = self.get_upload_url(
            token,
            filekey=filekey,
            media_type=media_type,
            to_user_id=to_user_id,
            rawsize=rawsize,
            rawfilemd5=rawfilemd5,
            filesize=filesize,
            aeskey_hex=aeskey.hex(),
        )
        upload_param = upload_resp.get("upload_param")
        if not upload_param:
            raise WeixinApiError(f"getuploadurl 未返回 upload_param")

        download_param = self._cdn_upload(plaintext, upload_param, filekey, aeskey)
        return {
            "download_param": download_param,
            "aeskey_raw": aeskey,
            "file_size": rawsize,
            "file_size_ciphertext": filesize,
        }

    def _upload_image_file(self, token: str, to_user_id: str, file_path: str) -> dict[str, Any]:
        plaintext = Path(file_path).read_bytes()
        rawsize = len(plaintext)
        rawfilemd5 = hashlib.md5(plaintext).hexdigest()
        filesize = aes_ecb_padded_size(rawsize)
        filekey = secrets.token_hex(16)
        aeskey = secrets.token_bytes(16)

        upload_resp = self.get_upload_url(
            token,
            filekey=filekey,
            media_type=1,
            to_user_id=to_user_id,
            rawsize=rawsize,
            rawfilemd5=rawfilemd5,
            filesize=filesize,
            aeskey_hex=aeskey.hex(),
            no_need_thumb=True,
        )
        upload_param = upload_resp.get("upload_param")
        if not upload_param:
            raise WeixinApiError(f"getuploadurl 未返回 upload_param")

        download_param = self._cdn_upload(plaintext, upload_param, filekey, aeskey)
        return {"download_param": download_param, "aeskey_hex": aeskey.hex()}

    def send_text(self, token: str, to_user_id: str, text: str, context_token: str) -> str:
        client_id = f"py-{uuid.uuid4()}"
        self._post_json(
            "ilink/bot/sendmessage",
            {
                "msg": {
                    "from_user_id": "",
                    "to_user_id": to_user_id,
                    "client_id": client_id,
                    "message_type": 2,
                    "message_state": 2,
                    "context_token": context_token,
                    "item_list": [{"type": 1, "text_item": {"text": text}}],
                }
            },
            token=token,
        )
        return client_id

    def send_file(self, token: str, to_user_id: str, file_path: str, context_token: str, text: str = "", force_file: bool = False) -> str:
        mime = guess_mime_type(file_path)
        if mime.startswith("image/") and not force_file:
            uploaded = self._upload_image_file(token, to_user_id, file_path)
            aes_key_base64 = base64.b64encode(uploaded["aeskey_hex"].encode("utf-8")).decode("utf-8")
            media_item = {
                "type": 2,
                "image_item": {
                    "media": {
                        "encrypt_query_param": uploaded["download_param"],
                        "aes_key": aes_key_base64,
                        "encrypt_type": 1,
                    }
                },
            }
        else:
            uploaded = self._upload_file(token, to_user_id, file_path, media_type=3)
            aes_key_base64 = base64.b64encode(uploaded["aeskey_raw"].hex().encode("utf-8")).decode("utf-8")
            media_item = {
                "type": 4,
                "file_item": {
                    "media": {
                        "encrypt_query_param": uploaded["download_param"],
                        "aes_key": aes_key_base64,
                        "encrypt_type": 1,
                    },
                    "file_name": os.path.basename(file_path),
                    "len": str(uploaded["file_size"]),
                },
            }

        items = []
        if text:
            items.append({"type": 1, "text_item": {"text": text}})
        items.append(media_item)

        client_id = ""
        for item in items:
            client_id = f"py-{uuid.uuid4()}"
            self._post_json(
                "ilink/bot/sendmessage",
                {
                    "msg": {
                        "from_user_id": "",
                        "to_user_id": to_user_id,
                        "client_id": client_id,
                        "message_type": 2,
                        "message_state": 2,
                        "context_token": context_token,
                        "item_list": [item],
                    }
                },
                token=token,
            )
        return client_id

    def download_image_to_temp(self, image_url: str) -> str:
        import mimetypes
        request = urllib.request.Request(image_url, method="GET")
        with urllib.request.urlopen(request, timeout=self.timeout) as response:
            content_type = response.headers.get("Content-Type")
            data = response.read()
        suffix = mimetypes.guess_extension((content_type or "").split(";")[0].strip()) or ".img"
        temp_dir = Path(tempfile.gettempdir()) / "weixin_push"
        temp_dir.mkdir(parents=True, exist_ok=True)
        temp_path = temp_dir / f"remote-{uuid.uuid4().hex}{suffix}"
        temp_path.write_bytes(data)
        return str(temp_path)


# ============ 存储模块 ============

BASE_DIR = Path(__file__).resolve().parent
DATA_DIR = BASE_DIR / "data"
SESSION_FILE = DATA_DIR / "session.json"
CONTACTS_FILE = DATA_DIR / "contacts.json"


def ensure_data_dir():
    DATA_DIR.mkdir(parents=True, exist_ok=True)


def load_json(path: Path, default=None):
    if not path.exists():
        return default
    return json.loads(path.read_text(encoding="utf-8"))


def save_json(path: Path, payload):
    ensure_data_dir()
    path.write_text(json.dumps(payload, ensure_ascii=False, indent=2), encoding="utf-8")


def save_session(session: Session):
    save_json(SESSION_FILE, {"token": session.token, "base_url": session.base_url, "account_id": session.account_id, "user_id": session.user_id, "saved_at": session.saved_at})


def load_session() -> Session | None:
    payload = load_json(SESSION_FILE)
    if not payload:
        return None
    return Session(**payload)


def load_contacts() -> dict[str, dict[str, Any]]:
    return load_json(CONTACTS_FILE, {})


def save_contacts(contacts: dict[str, dict[str, Any]]):
    save_json(CONTACTS_FILE, contacts)


def upsert_contact(user_id: str, *, context_token: str, last_text: str, last_message_id: int | str | None, last_seen_at: str):
    contacts = load_contacts()
    contacts[user_id] = {"context_token": context_token, "last_text": last_text, "last_message_id": last_message_id, "last_seen_at": last_seen_at}
    save_contacts(contacts)


# ============ CLI 模块 ============

def require_session() -> Session:
    session = load_session()
    if session is None:
        raise SystemExit("未找到登录会话，请先执行: python -m weixin_push.cli login")
    return session


def cmd_login(_):
    client = WeixinClient()
    session = client.wait_for_login()
    save_session(session)
    print(f"已保存登录会话: {session.account_id}")
    return 0


def cmd_listen(args):
    session = require_session()
    client = WeixinClient(base_url=session.base_url)
    get_updates_buf = args.sync_buf or ""
    print(f"开始监听，Bot: {session.account_id}")
    while True:
        try:
            resp = client.get_updates(session.token, get_updates_buf=get_updates_buf)
            if resp.get("get_updates_buf"):
                get_updates_buf = resp["get_updates_buf"]
            if resp.get("ret", 0) != 0:
                raise WeixinApiError(f"getupdates 失败: ret={resp.get('ret')}")
            for msg in resp.get("msgs") or []:
                if msg.get("message_type") != 1:
                    continue
                from_user_id = msg.get("from_user_id") or ""
                context_token = msg.get("context_token") or ""
                text = extract_text(msg)
                ts_ms = msg.get("create_time_ms")
                when = dt.datetime.fromtimestamp(ts_ms / 1000).isoformat(timespec="seconds") if isinstance(ts_ms, (int, float)) else dt.datetime.now().isoformat(timespec="seconds")
                print(f"[{when}] {from_user_id}: {text}")
                if from_user_id and context_token:
                    upsert_contact(from_user_id, context_token=context_token, last_text=text, last_message_id=msg.get("message_id"), last_seen_at=when)
        except KeyboardInterrupt:
            print("\n监听已停止。")
            return 0
        except Exception as exc:
            print(f"监听异常: {exc}")
            time.sleep(3)


def cmd_contacts(_):
    contacts = load_contacts()
    if not contacts:
        print("暂无联系人缓存。先执行 listen 收取一条用户消息。")
        return 0
    for user_id, info in sorted(contacts.items(), key=lambda x: x[1].get("last_seen_at", ""), reverse=True):
        print(user_id)
        print(f"  最近时间: {info.get('last_seen_at', '-')}")
        print(f"  最近消息: {info.get('last_text', '-')}")
    return 0


def cmd_send(args):
    session = require_session()
    contacts = load_contacts()
    info = contacts.get(args.to)
    if info is None:
        raise SystemExit("未找到该用户的 context_token，请先让对方发一条消息并执行 listen。")
    context_token = info.get("context_token")
    if not context_token:
        raise SystemExit("该用户缺少 context_token，无法发送。")

    client = WeixinClient(base_url=session.base_url)
    if args.file:
        file_path = os.path.abspath(args.file)
        if not os.path.exists(file_path):
            raise SystemExit(f"文件不存在: {file_path}")
        message_id = client.send_file(session.token, args.to, file_path, context_token, args.text or "", force_file=True)
    elif args.image_url:
        temp_path = client.download_image_to_temp(args.image_url)
        try:
            message_id = client.send_file(session.token, args.to, temp_path, context_token, args.text or "", force_file=False)
        finally:
            try:
                os.remove(temp_path)
            except OSError:
                pass
    else:
        message_id = client.send_text(session.token, args.to, args.text, context_token)
    print(f"发送成功，client_id={message_id}")
    return 0


def build_parser():
    parser = argparse.ArgumentParser(description="微信 iLink Bot 推送工具")
    subparsers = parser.add_subparsers(dest="command", required=True)
    subparsers.add_parser("login", help="扫码登录并保存 token").set_defaults(func=cmd_login)
    listen_parser = subparsers.add_parser("listen", help="长轮询监听消息")
    listen_parser.add_argument("--sync-buf", default="", help="调试用途：指定初始 get_updates_buf")
    listen_parser.set_defaults(func=cmd_listen)
    subparsers.add_parser("contacts", help="查看已缓存联系人").set_defaults(func=cmd_contacts)
    send_parser = subparsers.add_parser("send", help="发送消息")
    send_parser.add_argument("--to", required=True, help="目标微信用户 ID")
    send_parser.add_argument("--text", help="发送文本")
    send_parser.add_argument("--file", help="发送本地文件")
    send_parser.add_argument("--image-url", help="发送网络图片 URL")
    send_parser.set_defaults(func=cmd_send)
    return parser


def main(argv=None):
    parser = build_parser()
    args = parser.parse_args(argv)
    return args.func(args)


if __name__ == "__main__":
    sys.exit(main())
```

### curl 最小实现

如果你只想用 bash 脚本快速测试：

```bash
#!/bin/bash

BASE_URL="https://ilinkai.weixin.qq.com"

echo "=== Step 1: 获取二维码 ==="
QR_RESP=$(curl -s "$BASE_URL/ilink/bot/get_bot_qrcode?bot_type=3")
QRCODE=$(echo $QR_RESP | python3 -c "import sys,json; print(json.load(sys.stdin).get('qrcode',''))")
echo "请扫描二维码: $QRCODE"

echo "=== Step 2: 等待扫码确认 ==="
while true; do
    STATUS_RESP=$(curl -s "$BASE_URL/ilink/bot/get_qrcode_status?qrcode=$QRCODE")
    STATUS=$(echo $STATUS_RESP | python3 -c "import sys,json; print(json.load(sys.stdin).get('status',''))")
    echo "状态: $STATUS"
    [ "$STATUS" = "confirmed" ] && break
    sleep 2
done

BOT_TOKEN=$(echo $STATUS_RESP | python3 -c "import sys,json; print(json.load(sys.stdin).get('bot_token',''))")
echo "登录成功，Token: $BOT_TOKEN"

echo "=== Step 3: 收消息 ==="
UIN=$(python3 -c "import base64,random; print(base64.b64encode(str(random.randint(0,2**32-1)).encode()).decode())")
curl -s "$BASE_URL/ilink/bot/getupdates" \
    -H "Content-Type: application/json" \
    -H "AuthorizationType: ilink_bot_token" \
    -H "Authorization: Bearer $BOT_TOKEN" \
    -H "X-WECHAT-UIN: $UIN" \
    -d '{"get_updates_buf": "", "base_info": {"channel_version": "1.0.2"}}'
```

---

## 参考资料

| 资源 | 链接 |
|------|------|
| Python SDK | https://pypi.org/project/openilink-sdk-python/ |
| 官方插件仓库 | https://github.com/hao-ji-xing/openclaw-weixin |

---

## 注意事项

1. **Token 安全**：保存好 `bot_token`，泄露可能导致账号被盗用
2. **context_token**：回复消息时必须原样带回，这是最常见的错误原因
3. **媒体加密**：所有媒体文件使用 AES-128-ECB 加密，需要本地处理
4. **get_updates_buf**：当成 opaque blob 处理，不要解析其内部结构
5. **合法使用**：请遵守微信使用协议，不要用于发送垃圾信息

---

祝你开发顺利！
