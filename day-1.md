# 把「总是掉线」的飞书机器人修好

## 起因

飞书群里的机器人隔一阵就不回话了。最初的报障只有一句话——「飞书总是掉线」。

拆开之后才发现，「掉线」根本不是一个问题,而是六个不同的故障叠在一起,每一个都有自己的修法。修完一个还掉,就是下一个在等着。

## 诊断:先分清是谁的锅

第一件事是把「网络问题」和「凭证问题」分开。最快的判据不是猜,而是看飞书到底返回了什么:

| 返回 | 结论 | 该做什么 |
| --- | --- | --- |
| HTTP 200 + `{"code":10014,"msg":"app secret invalid"}` | 凭证失效,与网络无关 | 去开放平台重取 Secret |
| HTTP 503 + 纯文本 `DNS cache overflow` | 出口网关抖动 | 重试即可,别误判成对方挂了 |
| 超时 / ConnectionReset | 出站链路问题 | 退避重试 |

这次是第一种 —— App Secret 已经和 App ID 对不上了。判断依据只花了一次请求:网络是通的,是飞书主动拒了。

同样的排查思路也适用于其他凭证:一个 key 能不能用,不能看它是否「存在」,要看它**实际调一次接口**。MiniMax 国内站的 key 能列出九个模型,却因为余额不足在调用时返回 `402 insufficient balance`——列得出来,不代表调得动。

## 修法:六层加固

既然是六个故障,就得六层一起上。清单长这样,做完一项划掉一项:

- [x] token 自动续期:磁盘缓存 + 过期前 5 分钟刷新
- [x] 凭证指纹绑定:token 缓存记录凭证指纹,Secret 一换旧 token 立刻作废
- [x] 网络抖动自愈:HTTP 503 / `DNS cache overflow` 走指数退避重试
- [x] 进程自愈:主循环兜住所有异常,凭证失效时进入等待恢复态而不是退出
- [x] 凭证热更新:指纹一变立即重新鉴权,不用重启进程
- [x] 心跳可观测:每轮写 `health.json`,随时能判断 bot 是死是活、卡在哪一步

## 关键实现:token 缓存必须绑定凭证指纹

这是最容易漏掉的一处。最初的实现只按过期时间判断缓存:

```python
def token(self, force=False):
    if not force and self._token and self._token_expire - time.time() > TOKEN_SKEW:
        return self._token          # ← Secret 已经换过了,这里还在用旧 token
    ...
```

问题在于:凭据被替换成新的那一瞬间,旧的 token **在服务器上其实还是有效的**,所以它不会报错,只会让你带着一个即将失效的身份继续跑,直到两小时后自然过期。修法是给缓存加一层凭证指纹:

```python
def _fingerprint(self):
    """凭证指纹(sha256 前 16 位),不存明文。"""
    raw = "%s|%s" % (self._app_id or "", self._app_secret or "")
    return hashlib.sha256(raw.encode()).hexdigest()[:16]

def _load_cached_token(self):
    d = json.load(open(self.token_file))
    if d.get("expire_at", 0) - time.time() <= TOKEN_SKEW:
        return False
    # 指纹不一致 = 凭证换过了,旧 token 不能复用
    if d.get("fingerprint") != self._fingerprint():
        return False
    self._token, self._token_expire = d["token"], d["expire_at"]
    return True
```

顺带一提,凭证指纹还能解决另一个坑:secret 存储更新后,**已经在跑的进程环境变量不会变**。所以光有环境变量是不够的,得支持从文件读凭证,让指纹一变就热切换:

```bash
# /workspace/.cache/feishu/cred.json
{"app_id": "cli_xxx", "app_secret": "xxx"}
```

## 可观测:心跳文件

最后一步是让「它到底还活着吗」变成一个可以一眼回答的问题。每轮轮询写一次心跳:

```json
{
  "status": "alive",
  "updated_at_str": "2026-11-13 15:00:34",
  "token_expires_at": 1784556834,
  "restarts": 0,
  "llm": "echo",
  "last_error": null
}
```

`status` 只有三种取值,足够覆盖所有场景:`alive`(正常)、`auth_error`(凭证失效,进程不退出、60 秒后重试,凭证一更新即恢复)、`degraded`(有异常但还在跑)。有了它,「掉线」就从一句抱怨变成一条日志。

## 一张图收尾

<img src="images/feishu-bot-hardening.png" alt="机器人被多层保护环包裹,周围环绕着时钟、重试箭头、云和齿轮图标,象征 token 续期、网络重试、进程自愈与心跳观测四类防线" width="720">

图里的几层环,对应的正是上面那张清单里的六条加固项。真正让机器人不再「总是掉线」的,不是某一处代码,而是**每一层都在自己失效的时候留下可读的痕迹**——心跳文件写着状态,日志写着原因,错误码指着该改哪里。
