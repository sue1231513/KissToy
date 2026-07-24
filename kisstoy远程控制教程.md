# kisstoy 远程控制速通教程

> 从扒源码到控制成功，踩过的每个坑都写在这里了。

---

## 一、原理

kisstoy 的远程控制不是 HTTP 轮询，是 **WebSocket 实时通信**。

```
被控端 APP ──蓝牙──> 玩具
被控端 APP ──WebSocket──> kisstoy 云端 <──WebSocket── 控制端
                                     （云端中转命令）
```

被控端（手机 APP）通过蓝牙连玩具，同时通过 WebSocket 连云端。控制端也连云端。云端负责中转控制命令。

---

## 二、前置条件

- kisstoy **新版 APP**（包名 `com.rek.kisstoy`，应用宝可下载）
  - 旧版 `com.vince.kissone` 服务器已停，蓝牙能连但云端不通
- 玩具已通过蓝牙连上 APP
- APP 中发起远程分享，获得分享链接
- 被控端 APP 需保持前台或小窗，后台可能断连

---

## 三、从分享链接提取参数

链接格式：
```
https://api.app.knightjenay.cn/kisstoy/remote/#/?device_id=13&group=xxxxxxxx&id=xxxxxx&lang=zh
```

三个关键参数：

| 参数 | 说明 | 示例 |
|------|------|------|
| `device_id` | 设备 ID | `13` |
| `group` | 控制群组码 | `6e9a15b369863b24496ce78271deb0b7` |
| `id` | 分享会话 ID（每次发起分享会变） | `725882` |

---

## 四、API 端点

Base URL: `https://api.app.knightjenay.cn`

### 1. 绑定远程控制

```
POST /kisstoy/remote-control/binding
Content-Type: application/json

Body: {"id": 分享会话ID}
```

返回：
```json
{"code": 1, "msg": "操作成功", "data": {}}
```

### 2. 获取设备详情（可选，查看支持的电机类型）

```
GET /kisstoy/device/detail?id={device_id}
```

返回示例：
```json
{
  "code": 1,
  "msg": "获取数据成功",
  "data": {
    "id": 13,
    "name": "迷路-吮吸版",
    "config": {
      "motors": [
        {"type": 1, "start_up": 0.2},
        {"type": 3, "start_up": 0.4}
      ]
    }
  }
}
```

`start_up` 是最低启动强度（0-1 的小数），实际发送时需乘以 100 转成整数。

---

## 五、WebSocket 连接

```
wss://api.app.knightjenay.cn/websocket-kisstoy?group={group}
```

连接成功后收到：
```json
{"event": "connect", "result": "success", "data": null}
{"event": "group", "result": "success", "data": null}
```

---

## 六、查询设备在线状态 ⚠️ 最大的坑

发送：
```json
{"event": "online_status", "data": {"group": "你的group码"}}
```

> **⚠️ 注意：`data` 里是 `group`，不是 `target`！写错就收不到回复！**
> 
> 我在这个坑里卡了一整轮，所有链接都显示"设备已关闭连接"，其实是查询格式写错了。

收到：
```json
{"event": "online_status", "result": "success", "data": {"online_status": 1}}
```

`online_status === 1` 表示设备在线，可以控制。

---

## 七、控制命令 ⚠️ 第二个坑

```json
{
  "event": "control",
  "data": {
    "target": "group码",
    "device_id": "设备ID",
    "motors": {
      "1": 50,
      "3": 45
    }
  }
}
```

### 电机类型

| type | 功能 | 最低启动值 | 值范围 |
|------|------|-----------|--------|
| 1 | 震动 | 20 | 0-100 整数 |
| 3 | 吮吸 | 40 | 0-100 整数 |
| 4 | 抽插/伸缩 | - | 0-100 整数 |
| 5 | 电击 | - | 0-100 整数 |
| 6 | 拍打 | - | 0-100 整数 |

> **⚠️ motors 的值是 0-100 的整数！不是 0-1 的小数！**
> 
> 发 0.2 在设备眼里约等于 0，等于没发。发 50 才是 50% 强度。
> 
> 我在这个坑里卡了第二轮，发了半天 0.2/0.4/0.6 对方毫无感觉，还以为是连接问题。
> 
> 源码里的量化函数：`E = D => D === 0 ? 0 : Math.floor(D / 5) * 5`
> Ka = 5，即值按 5 的步长量化（0, 5, 10, 15, 20, ... 100）。

### 按钮控制（部分设备）

```json
{
  "event": "control",
  "data": {
    "target": "group码",
    "device_id": "设备ID",
    "button": {
      "button_type": 1
    }
  }
}
```

`button` 值为 1（开）或 0（关），与 `motors` 二选一。

---

## 八、心跳

每 **10 秒** 发一次，否则连接会被掐：

```json
{"event": "ping"}
```

收到：
```json
{"event": "ping", "result": "success", "data": "pong"}
```

---

## 九、停止

motors 全部设为 0：

```json
{
  "event": "control",
  "data": {
    "target": "group码",
    "device_id": "设备ID",
    "motors": {"1": 0, "3": 0}
  }
}
```

---

## 十、踩坑总结

| # | 坑 | 原因 | 解决 |
|---|-----|------|------|
| 1 | 页面一直显示"设备已关闭连接" | online_status 查询用了 `target` | 改成 `group` |
| 2 | 控制命令发了对方没感觉 | motors 值用了 0-1 小数 | 改成 0-100 整数 |
| 3 | 设备时在线时不在线 | APP 切到后台断连 | 保持前台或小窗 |
| 4 | 分享链接失效 | 设备断线后会话超时 | 重新在 APP 发起分享 |
| 5 | 旧版 APP 一直超时 | 旧版服务器已停 | 装 `com.rek.kisstoy` 新版 |

---

## 十一、完整流程（Python 示例）

```python
import json
import time
import threading
import websocket  # pip install websocket-client

class KisstoyRemote:
    """kisstoy 远程控制器"""

    API_BASE = "https://api.app.knightjenay.cn"
    WS_BASE = "wss://api.app.knightjenay.cn/websocket-kisstoy"

    def __init__(self, device_id, group, share_id):
        self.device_id = str(device_id)
        self.group = group
        self.share_id = share_id
        self.ws = None
        self.online = False
        self._heartbeat = None

    def bind(self):
        """绑定远程控制"""
        import requests
        r = requests.post(
            f"{self.API_BASE}/kisstoy/remote-control/binding",
            json={"id": self.share_id}
        )
        return r.json()

    def connect(self):
        """建立 WebSocket 连接"""
        url = f"{self.WS_BASE}?group={self.group}"
        self.ws = websocket.WebSocketApp(
            url,
            on_open=self._on_open,
            on_message=self._on_message,
            on_close=self._on_close,
            on_error=self._on_error
        )
        # 后台运行
        t = threading.Thread(target=self.ws.run_forever, daemon=True)
        t.start()

    def _on_open(self, ws):
        print("[kisstoy] WebSocket 已连接")
        # 查询在线状态（注意：是 group 不是 target！）
        self._send({"event": "online_status", "data": {"group": self.group}})
        # 启动心跳
        self._heartbeat = threading.Timer(10, self._heartbeat_loop)
        self._heartbeat.daemon = True
        self._heartbeat.start()

    def _heartbeat_loop(self):
        if self.ws:
            self._send({"event": "ping"})
            self._heartbeat = threading.Timer(10, self._heartbeat_loop)
            self._heartbeat.daemon = True
            self._heartbeat.start()

    def _on_message(self, ws, message):
        data = json.loads(message)
        if data.get("event") == "online_status":
            self.online = data.get("data", {}).get("online_status") == 1
            print(f"[kisstoy] 设备在线状态: {self.online}")
        elif data.get("event") == "ping" and data.get("result") == "success":
            pass  # 心跳正常

    def _on_close(self, ws):
        print("[kisstoy] WebSocket 已断开")
        self.online = False

    def _on_error(self, ws, error):
        print(f"[kisstoy] WebSocket 错误: {error}")

    def _send(self, data):
        if self.ws:
            self.ws.send(json.dumps(data))

    def control(self, motors):
        """
        控制电机
        motors: dict, key=电机类型(str), value=强度(int 0-100)
        例: {"1": 50, "3": 45}  震动50, 吮吸45
        """
        if not self.online:
            print("[kisstoy] 设备离线，命令被忽略")
            return
        self._send({
            "event": "control",
            "data": {
                "target": self.group,
                "device_id": self.device_id,
                "motors": {str(k): v for k, v in motors.items()}
            }
        })
        print(f"[kisstoy] 已发送: {motors}")

    def stop(self):
        """停止所有电机"""
        self.control({"1": 0, "3": 0})

    def disconnect(self):
        """断开连接"""
        if self._heartbeat:
            self._heartbeat.cancel()
        if self.ws:
            self.ws.close()


# ── 使用示例 ──────────────────────────────────────
if __name__ == "__main__":
    # 从分享链接提取的参数
    toy = KisstoyRemote(
        device_id="13",           # 链接里的 device_id
        group="你的group码",       # 链接里的 group
        share_id=725882           # 链接里的 id
    )

    # 1. 绑定
    print(toy.bind())

    # 2. 连接 WebSocket
    toy.connect()

    # 3. 等待设备上线
    time.sleep(3)
    if not toy.online:
        # 再查一次
        toy._send({"event": "online_status", "data": {"group": toy.group}})
        time.sleep(2)

    if toy.online:
        # 4. 控制！注意值是 0-100 的整数！
        toy.control({"1": 20})        # 震动 20（最轻）
        time.sleep(5)
        toy.control({"1": 40})        # 震动 40
        time.sleep(5)
        toy.control({"1": 40, "3": 40})  # 震动40 + 吮吸40
        time.sleep(5)
        toy.stop()                    # 停止
    else:
        print("设备不在线，检查 APP 是否保持前台")

    # 5. 断开
    toy.disconnect()
```

---

## 附：从源码扒出的关键信息

源码位置：`https://api.app.knightjenay.cn/kisstoy/remote/assets/index-290ec656.js`

### 量化函数
```js
Ka = 5
E = D => D === 0 ? 0 : Math.floor(D / Ka) * Ka
```
值按 5 的步长量化：0, 5, 10, 15, 20, ... 100

### 在线状态查询
```js
L = () => {
    ws.send(JSON.stringify({event: "online_status", data: {group: r}}))
}
```
**`group` 不是 `target`！**

### 控制命令
```js
// 滑块控制（motors）
p = (D, W) => {
    const z = E(W);  // 量化后的值
    ws.send(JSON.stringify({
        event: "control",
        data: {target: r, device_id: s, motors: {[D]: z}}
    }))
}

// 按钮控制（button）
T = D => {
    ws.send(JSON.stringify({
        event: "control",
        data: {target: r, device_id: s, button: {[D]: u.value[D]}}
    }))
}
```

### 电机类型映射
```js
type 1: 震动 (vibrate)
type 2: 恒温/加热
type 3: 气囊/吮吸
type 4: 抽插/伸缩 (thrust)
type 5: 电击 (electric)
type 6: 拍打 (flap)
```
（不同设备支持的类型不同，查 device/detail 接口看 config.motors）

### 心跳间隔
```js
vE = 1e4  // 10000ms = 10秒
```

---

> 教程完。有手就能扒，有网就能控。
> 
> 两个坑记住就行：**online_status 用 group**，**motors 值用 0-100 整数**。
> 其余都是顺路的事。
