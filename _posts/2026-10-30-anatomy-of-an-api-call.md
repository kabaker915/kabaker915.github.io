---
layout: post
title: "一次大模型 API 调用，到底发生了什么"
date: 2026-10-30 22:00:00 +0800
tags: [Python, API, 大模型]
description: "我一直觉得调 API 是黑魔法，直到把它拆成五个步骤。这篇用自己项目里的代码做解剖，讲清字典、JSON、字节三者的区别，以及为什么不写 timeout 会让程序永久卡死。"
---

项目里有一段调用 DeepSeek 的代码，是 AI 帮我写的。它能跑，但我完全看不懂。

后来我强迫自己把它拆开，才发现这件事**没有魔法**——本质上就是一次 HTTP 请求。

## 一、心智模型：这就是打电话

把大模型 API 想象成一个只会听普通话、只回普通话的接线员。

| 打电话 | 代码里对应的东西 |
| --- | --- |
| 拨的号码 | `url = "https://api.deepseek.com/chat/completions"` |
| 你的证件（证明你是谁、扣谁的钱） | `Authorization: Bearer sk-xxx` |
| 你说的内容 | `messages` 里的 `content` |
| 必须说普通话 | 数据先 `json.dumps` 再 `encode('utf-8')` |
| 接通后要等对方说话 | `urlopen(req)` 会**卡住**，直到对方回话 |
| 对方回话 | `response.read()` |
| 听懂对方说什么 | `json.loads(...)` 转回字典 |

所谓"调用大模型 API"，就是：

> 你按它规定的格式，往一个网址 POST 一段 JSON；它按它规定的格式，POST 回来一段 JSON。

## 二、五个步骤

```
1. 组装你想说的话        data = {...}
2. 变成字节流            json.dumps(data).encode('utf-8')
3. 装成请求              Request(url, data=..., headers={...}, method='POST')
4. 发出去并阻塞等待      urlopen(req, timeout=20)
5. 拆信封取回答          json.loads(...)["choices"][0]["message"]["content"]
```

## 三、逐步拆开看

### 第一步：组装请求体

```python
data = {
    "model": "deepseek-chat",
    "messages": [
        {"role": "system", "content": "你是一个专业的网络安全运营专家。"},
        {"role": "user", "content": "请分析这条日志..."},
    ],
    "temperature": 0.3,
}
```

三个字段：

| 字段 | 作用 |
| --- | --- |
| `model` | 用哪个模型 |
| `messages` | 对话内容，是一个**列表** |
| `temperature` | 0~2，越低越稳定 |

`messages` 为什么是列表？因为大模型 API 是**无状态**的——它不记得上一次对话。
每次请求都要把完整历史一起发过去。

- `role: "system"` → 人设，相当于"你现在扮演安全专家"
- `role: "user"` → 用户说的话
- `role: "assistant"` → 模型之前说过的话（多轮对话用）

`temperature: 0.3` 调得比较低，因为安全研判要的是稳定结论，不要它自由发挥。

### 第二步：字典 → 字节

```python
payload = json.dumps(data).encode('utf-8')
```

这一行做了两件事，很多人只知道一半：

```python
{"model": "deepseek-chat"}       # Python 字典，内存里的对象
'{"model": "deepseek-chat"}'     # JSON 字符串，带引号的文本
b'{"model": "deepseek-chat"}'    # bytes，前面多个 b
```

`json.dumps` 把对象序列化成 JSON 文本，`.encode('utf-8')` 把文本编成字节。

**为什么必须要字节？** 因为网络只能传字节，不能直接传 Python 对象。

记忆口诀：**出内存 encode，进内存 decode。**

### 第三步：装成请求

```python
headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {api_key}",
}
req = urllib.request.Request(url, data=payload, headers=headers, method="POST")
```

这一步**还没有发出去**，只是把信封装好。

两个 header 必须理解：

- `Content-Type: application/json`
  告诉服务器"我发给你的是 JSON，请按 JSON 解析"。不写可能被拒。

- `Authorization: Bearer sk-xxxx`
  身份认证。`Bearer` 意思是**"持有者"**——谁拿着这个 token，谁就被当成你。

这就是为什么 key 泄露等于账号被白用。任何人拿到它，都能冒充你调用、扣你的钱。

### 第四步：发出去

```python
with urllib.request.urlopen(req, timeout=20) as response:
```

这一行是**真正的网络请求**，程序会停在这里等对方回话。

`with` 保证代码块结束时自动关闭连接，就算中间报错也会关。
不关会泄漏文件句柄，量大了把系统资源耗尽。

### 第五步：拆信封

```python
raw = response.read().decode("utf-8")   # 字节 → 字符串
result = json.loads(raw)                # 字符串 → 字典
answer = result["choices"][0]["message"]["content"]
```

为什么路径这么长？因为服务器返回的东西长这样：

```json
{
  "id": "chatcmpl-xxx",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "这条日志是 SQL 注入攻击……" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 120, "completion_tokens": 80 }
}
```

- `choices` 是**列表**，因为一次可以生成多个候选（默认给 1 个，所以取 `[0]`）
- `message.content` 才是模型真正说的话
- `usage` 里是 token 数，就是计费依据

## 四、我踩过的两个坑

### 坑一：把 API Key 写死在代码里

```python
api_key = "sk-fd05..."     # 千万别这么干
```

我一开始就这么写的，而且还在另一个文件里又抄了一份。

源码会被 git 记录、被打包、被截图发到群里。**一旦进去就再也收不回来。**

正确做法是环境变量：

```python
import os
api_key = os.getenv("DEEPSEEK_API_KEY", "")
```

Windows PowerShell 里临时设置：

```powershell
$env:DEEPSEEK_API_KEY = "sk-你的key"
```

如果 key 已经泄露过，**第一件事是去后台作废重建**，而不是删掉那行代码。
删代码只是让你看不见，key 本身还是有效的。

### 坑二：没写 timeout

```python
urlopen(req)             # 默认无限等待
urlopen(req, timeout=20) # 正确
```

不写 timeout，服务器不响应的时候程序就**永久卡在这一行**。
界面上看起来就是"转圈不动"，连报错都没有，极难排查。

**所有外部调用都必须设超时。** 这是工程常识，不只是这一处。

## 五、最小可运行版本

理解之后我重写了一遍，20 行：

```python
import json
import os
import urllib.request

API_KEY = os.getenv("DEEPSEEK_API_KEY", "")

def ask(prompt: str) -> str:
    url = "https://api.deepseek.com/chat/completions"
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {API_KEY}",
    }
    body = {
        "model": "deepseek-chat",
        "messages": [{"role": "user", "content": prompt}],
        "temperature": 0.2,
    }

    payload = json.dumps(body).encode("utf-8")
    req = urllib.request.Request(url, data=payload, headers=headers, method="POST")

    try:
        with urllib.request.urlopen(req, timeout=20) as resp:
            return json.loads(resp.read().decode("utf-8"))["choices"][0]["message"]["content"]
    except Exception as exc:
        return f"调用失败: {exc}"

if __name__ == "__main__":
    print(ask("用一句话解释什么是 SQL 注入"))
```

跑通之后我删掉重写了两遍，才算是真记住了。

## 六、能跑之后，才会发现问题

调通只是开始。真拿它分析日志，立刻遇到三个新问题：

**1. 太慢。** 每条命中都同步调一次，50 条告警就是 50 次串行 HTTPS 请求。

**2. 太贵。** sqlmap 扫 200 次会产生 200 条 SQL 注入告警，
但它们在语义上是**同一件事**，不需要问 200 次。

**3. 太脆。** 对方限流、网络抖动、余额不足，任何一次失败都会让整个分析中断。

所以我给这个模块加了四层：

| 加了什么 | 解决什么 |
| --- | --- |
| 按（攻击类型 + 规则组合）去重 | 成本 |
| 线程池并发 | 速度 |
| 重试 + 指数退避 | 限流 |
| 失败降级 | 可靠性 |

**其中最重要的是最后一条。** 它背后的判断是：

> **AI 不在主链路上。规则检测是主链路，大模型只是增强项。**

没有它，系统必须依然能出报告。这句话我在面试里会主动讲——
因为它体现的是架构判断，而不是"我调了个大模型"。

## 七、如果被问到，我会这么答

**Q：为什么要用大模型？正则不够吗？**

> 不够。正则只能抓已知特征，改个编码就绕过。大模型能做语义判断，
> 看到 `SLEEP(5)` 能理解这是时间盲注。
> 但大模型也会幻觉、有延迟、要花钱，所以我让它只做"粗筛之后的精判"，
> 不参与最终判定，只输出给人看的解释和处置建议。

**Q：调用失败怎么办？**

> 三层：超时避免挂死、指数退避重试、最终降级只输出规则结论。
> AI 挂了，报告照样出。

**Q：成本怎么控？**

> 按攻击类型去重，同一类只问一次；设单次分析调用上限；payload 截断到 600 字符。
> 200 条 sqlmap 告警实际只会产生一次付费调用。

---

_下一篇打算写：我怎么把这一堆脚本重构成分层的模块，以及为什么分层的第一个好处其实不是"好看"。_
