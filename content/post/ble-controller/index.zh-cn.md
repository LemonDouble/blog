---
title: "为了在 Home Assistant 中使用 AliExpress 买的电视背光灯而逆向蓝牙协议的故事"
slug: ble-controller-dev-story
date: 2026-04-05T18:07:00+09:00
image: cover.png
categories:
    - HomeAssistant
    - IoT
    - BLE
tags:
    - HomeAssistant
    - BLE
    - GATT
    - bleak
    - HACSD
    - 智能家居
    - AliExpress
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

![AliExpress 商品图片](./backlight.webp)

我在 AliExpress 上买了一个电视背光灯。

其实一开始我并不在意联动不联动的问题..反正只要电视后面有光,才区区2万5千韩元[^1]?没理由不买吧?

[^1]: 约合人民币130元左右。

随便想想..要是能和电视内容实时联动那当然好..

如果是 Chromecast 的话有 HDMI 线,挂个 HDMI 钩子用就行,但我的电视没有那种东西,是 Android TV。那么大概?通过 ADB 连接或者装个单独的 App,提取屏幕边缘的信息然后实时联动..?

但是,2万5千韩元的东西要求那么多就过分了吧?所以~大概是把摄像头传感器贴在一起,摄像头识别到的颜色直接打到 LED 上的简单结构。

因为只拍电视的中央部分,所以和电视颜色完全 Sync 的感觉差得有点远,不过这无所谓,重要的是电视后面有光,嗯嗯。

但是?

![App / 遥控器图片](./image1.png)

其实是收到货之后才知道,居然有专用 App 和遥控器之类的东西。遥控器没什么用就扔了..专用 App 试了一下,发现是用蓝牙连接的。

不过?在 AliExpress?按销量排序排在最上面..想着说不定有哪位民间高手做了个可疑的10星专用集成什么的,内心抱着期待..翻来翻去也没找到那种东西..

倒是有个会蹦10次蓝牙权限确认的廉价专用 App..但努力翻了集成,似乎也没有。

但忽然想到..

- 蓝牙信号大致就是往周围撒的..可以从其他地方接收来分析
- 而且?既然有 App,而且蓝牙信号是从我手机里发出去的?估计扒一扒 Android 的调试日志就能知道信号
- 那么,虽然不太清楚?如果能想办法捕获 ON/OFF 信号,就可以接到 Home Assistant 上吧?

就这么突然冒出了这个想法,但是...

为了关掉一个2万5千韩元的 LED 设备折腾这么多?抱着这种心情眯着眼睛假装没看见....

<img src="./image2.png" style="width: 400px;" alt="电视后面的背光灯关不掉">

糟糕..背光灯关不掉了。

按理说

- 把 USB 电源线插到电视上
- 电视关了 USB 也就不供电了
- 电视一关背光灯也就`智能`地关掉

应该是这样的工作流..但我的情况是

- 我用 Android Debug Bridge 把 Home Assistant 和电视联动起来。用这个开电视什么的做各种事情
- 但是?电视完全关掉的话 ADB 就连不上
- 之前做的所有自动化都用不了
- 所以?我就让电视保持在省电模式
- 省电模式嘛?有电流通过,USB 也就有电
- 背光灯永远关不掉

我就这样开始走上了痛苦的循环..

当然有 App..LED 控制器上也有物理开关..但是麻烦啊...

所以咬着牙和永远开着的 LED 背光灯一起生活..感觉睡眠时间慢慢变少了,啊..趁周末来折腾一下吧..就这样开始了。

不过既然要做?蓝牙信号读写本来就是很常见的场景嘛?所以就找有没有现成的能写蓝牙信号的,意外地几乎没有。

那干脆..做个通用的能用很久,这样想着就动手了。

所以?这篇文章就是在做 [BLE Controller](https://github.com/Lemon-HACS/ble_controller) 时遇到的技术问题及解决过程的整理。


那么要折腾..首先得知道..怎么扒蓝牙包..吧?

我以前..在学校做项目时倒是做过 BLE Scan,但实际上从来没扒过包。(都在库的上层玩)

所以看到 Raw 就有点茫然..

先在 Android 里找到蓝牙调试,做个日志文件 (Android 里叫做 bug report)..

然后整个 zip 文件就直接扔给 Claude。现在情况这样..开关了4次左右,有什么吗?

它说能看到日志,我就开始瞎折腾。能跑通最好,跑不通也能在折腾中学到东西,没什么损失吧?

### 1. 大致结构

在故障排除故事之前..先大致过一下整体结构,要理解的话先整理一下 BLE 相关术语。

**BLE (Bluetooth Low Energy)** — 蓝牙低功耗版本。主要用在低功耗小型设备上的协议。在功耗很重要的 IoT 中也常用。

**GATT (Generic Attribute Profile)** — 与 BLE 设备交换数据的规约。简单来说就是定义了"对这个设备的某地址发什么内容能干什么"。结构是这样的:
- **Service** — 功能集合(例如:"LED 控制服务")。有一个叫 UUID 的唯一地址
- **Characteristic** — Service 中的具体功能(例如:"电源 ON/OFF")。也有 UUID
- 最终只要知道 **Service UUID + Characteristic UUID + 要发的 hex 数据**这三样,就能控制任何设备

**BlueZ** — Linux 的蓝牙栈。Home Assistant 是基于 Linux 的,所以 BLE 通信都要经过 BlueZ。偶尔会吐一些奇怪的错..这个后面再说。

所以做了一个让用户从 UI 直接输入这三个值的结构。

```
config_flow.py   ← UI 配置(设备扫描、UUID/数据输入)
__init__.py      ← 入口设置(创建 BLEDeviceManager)
ble_client.py    ← 管理所有 BLE 连接
switch.py        ← 开关实体
button.py        ← 按钮实体
select.py        ← 选择实体
```

#### BLEDeviceManager — 连接的心脏

`ble_client.py` 中的 `BLEDeviceManager` 管理所有 BLE 连接。

```python
class BLEDeviceManager:
    def __init__(self, hass, mac, *, keep_alive=False, keepalive_interval=10):
        self._client: BleakClientWithServiceCache | None = None
        self._connect_lock = asyncio.Lock()      # 连接串行化
        self._operation_lock = asyncio.Lock()     # GATT 操作串行化
        self._disconnect_timer: asyncio.TimerHandle | None = None
```

这里两个 Lock 是关键:
- **`_connect_lock`**:连接/断开一次只处理一个。两个地方同时尝试连接 BlueZ 会乱掉。
- **`_operation_lock`**:GATT Write 也是一次一个。BLE 同时发多条命令会被丢弃。

#### 两种连接模式

**On-demand 模式**(默认):只在发命令时连接,15秒安静后自动断开。BLE 是 1:1 连接,HA 占着连接的话厂家 App 就访问不了。需要并行使用时很有用。

```python
def _reset_disconnect_timer(self):
    if self._keep_alive:
        return  # keep_alive 模式下不需要计时器
    self._cancel_disconnect_timer()
    self._disconnect_timer = loop.call_later(
        DISCONNECT_DELAY,  # 15 秒
        lambda: asyncio.ensure_future(self._timed_disconnect()),
    )
```

**Keep Alive 模式**:HA 启动后立即连接,后台周期性检查连接状态。断了会自动重连。如果不打算用厂家 App,想让 HA 全部控制,这种方式响应更快。

```python
async def _keepalive_loop(self):
    if await self._ensure_connected():
        await self._fire_on_connect()  # 初始状态查询
    while True:
        await asyncio.sleep(self._keepalive_interval)
        if self._client is None or not self._client.is_connected:
            if await self._ensure_connected():
                await self._fire_on_connect()
        else:
            await self._fire_on_connect()  # 用 ping 维持连接
```


### 2. 故障排除之旅

从这里开始..折腾了相当多..

老实说我没想过为了一个 On/Off 开关要折腾这么多..

#### 2-1. Write With Response 无限 hang

**症状**:第一条命令好好的。但从第二条开始永远没响应。

**原因**:测试设备(uLamp 电视背光灯)的 Write Characteristic(功能)只支持 `Write Without Response`(0x04),但我把 `Write With Response` 设成了 ON。

简单说明一下..BLE Characteristic 有个**Properties**属性。定义"给我发数据时用这种方式"。主要有两种:
- **Write Without Response (0x04)** — 发了就完事。不管设备有没有收到
- **Write With Response (0x08)** — 发了之后等"已收到"的 ACK

可以用 [nRF Connect](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-mobile) 这个 App 确认..

<img src="./nrf.png" style="width: 400px;" alt="在 nrf 中看到的数据">

那里写的 NO RESPONSE 就是不给 ACK 的意思。

那 0xFFB1 是 On/Off 我是怎么知道的?-> 这个是从 HCI 日志里看出来的。

总之结论..调用 `write_gatt_char(response=True)` 会向设备发 Write Request 并等 ACK..但设备如果不支持,**ACK 永远不会来**。

而且为了等这个 ACK 内部一直占着锁,后面来的 Write 命令也全都进了等待状态。第一条命令永远完不了,第二条第三条也一个个堵住..

**解决**:给所有 GATT Write 加了 3 秒超时。

```python
async def write(self, char_uuid, data, response=False):
    async with self._operation_lock:
        if not await self._ensure_connected():
            return False
        try:
            await asyncio.wait_for(
                self._client.write_gatt_char(char_uuid, data, response=response),
                timeout=GATT_TIMEOUT,  # 3 秒
            )
            return True
        except TimeoutError:
            _LOGGER.error("GATT write 超时 — 强制 disconnect")
            await self._disconnect()
            return False
```

超时就强制断开连接释放 Lock,下一条命令重新连接。

**教训**:控制 BLE 设备前,务必用 nRF Connect 之类的 App 检查 Characteristic Properties。先看清楚设备给不给 ACK..


#### 2-2. 0x0e Unlikely Error — Notify 订阅冲突

**症状**:间歇性出现 `BleakDBusError: 0x0e Unlikely Error`。

间歇性的更让人烦躁(..)

`0x0e Unlikely Error` 是什么呢..是 BlueZ(Linux 蓝牙栈)在认为"这种情况不应该发生"时吐出的错误。名字叫 Unlikely 听着挺好笑,但实战中其实经常碰到(..)

**原因**:BLE 有个**Notify**功能。设备主动告诉你"状态变了!"——要接收这个消息,需要用 `start_notify()` 订阅。

问题是这个订阅会从多个地方同时进行。后台周期性检查状态的循环和用户按开关时执行的代码同时尝试订阅同一个 Characteristic(功能)..BlueZ 就抛出 `0x0e`。

最初的实现是每次发数据时:
1. `start_notify()` — 开始订阅
2. 发送数据
3. 等待 Notify 响应
4. `stop_notify()` — 取消订阅

这个流程在两个地方同时执行就冲突。

**解决**:不再每次都订阅/取消,改成连接时只订阅一次并一直保持。简单说就是一个地方持续接收 Notify 放到缓冲区,其他地方从缓冲区里读。

```python
async def _subscribe_persistent_notify(self):
    """连接后调用一次。连接保持期间订阅一直保留。"""
    if not self._persistent_notify_uuid or self._notify_subscribed:
        return
    await self._client.start_notify(self._persistent_notify_uuid, self._on_notify)
    self._notify_subscribed = True

def _on_notify(self, _handle, data):
    """所有 Notify 接收都存到缓冲区。"""
    self._notify_data_buffer.append(bytes(data))
    self._notify_event.set()
```

订阅/取消的反复消失后错误也消失了。


#### 2-3. HA 实体状态更新延迟

**症状**:设备马上就开了,但 HA UI 的开关状态过好一会儿才变。

体感上挺明显的问题。

**原因**:`write_and_notify()` 在 Write 后会等 1.5 秒的 Notify 响应。如果 Notify 不来或者不匹配每次都要浪费 1.5 秒..

**解决**:改成 Write 之后立即乐观更新状态。前面说过 Write Without Response 模式下设备不发"收到"的 ACK。所以没法知道是否真的发出去了,但既然连接正常那大概率是发出去了吧~就先把 UI 更新掉。如果联动正常,发出去的概率要高得多。(顺便说,Twitter 的点赞/转发也是这么工作的。)

```python
# Before: 用 write_and_notify() 等 Notify 响应
ok, state = await self._manager.write_and_notify(...)
if state is not None:
    self._attr_is_on = state

# After: 立即反映 expected state
ok = await self._manager.write(self._char_uuid, data, response=self._response)
if ok:
    self._attr_is_on = expected_on  # 乐观更新
self.async_write_ha_state()
```

要是设备实际状态确实不一样呢?Keepalive 循环会周期性获取设备状态,以那个为准矫正。结果就是 UI 立即响应,实际状态的偏差几秒内就会被纠正。


#### 2-4. BLE 连接中断 — Keepalive 没在 Keep Alive 的问题

**症状**:开了 Keep Alive,但"意外断开连接"日志每 2 分钟出现 8~12 次。

诶 Keep Alive 怎么连接还会断,那不就不是 KeepAlive 了吗?

**原因**:初始实现只检查连接状态没发数据。BLE 设备一段时间没数据交换的话会因为 **idle timeout** 把连接断掉。(为了省电嘛)

```python
# Before: 只检查连接状态
while True:
    await asyncio.sleep(60)
    if not self._client.is_connected:
        await self._ensure_connected()  # 断了之后才重连
```

**解决**:让 Keepalive 循环周期性地**发送实际数据**。

```python
# After: 周期性发送 status query (ping)
while True:
    await asyncio.sleep(self._keepalive_interval)  # 10 秒(可配置)
    if not self._client.is_connected:
        if await self._ensure_connected():
            await self._fire_on_connect()  # 重连 + 状态查询
    else:
        await self._fire_on_connect()  # ping! 维持连接 + 刷新状态
```

这样做结果一举两得:
1. 周期性数据发送让设备不会断开连接
2. 状态查询的响应让 HA 实体状态也始终是最新的


#### 2-5. Keepalive 周期 — 每个设备都不一样

**症状**:Keepalive 周期从 60 秒 → 30 秒 → 15 秒 → 10 秒一路减,还是会断。

**原因**:我的背光灯就算 10 秒也经常断..减到 5 秒才稳定..BLE 设备的 idle timeout 各不相同。

其实最初做 Android HCI 日志 dump 时如果把连接静置一会儿就能知道连接是怎么维持的..但当时嫌麻烦就只抓了连接 + 开关,这是失策..

**解决**:把 Keepalive 周期做成每个设备可配置。

```python
class BLEDeviceManager:
    def __init__(self, hass, mac, *, keep_alive=False, keepalive_interval=10):
        self._keepalive_interval = keepalive_interval
```

在 Config Flow 和 Options Flow 中都加了 `keepalive_interval` 字段,每个设备都能设置最优值。默认 10 秒,不稳定的话推荐 5~7 秒。

#### 2-6. 没有 Notify UUID 只输入 pattern 会被悄悄忽略的问题

**症状**:输入了 Notify ON/OFF pattern 但状态确认不工作。也不报错就这么悄悄被忽略,所以原因很难找。

**原因**:Notify 相关配置因为有些设备不支持所以做成 Optional 了..但要用 Notify,**Notify UUID、ON pattern、OFF pattern、状态查询命令**这 4 个必须凑齐一套。少一个都不工作,但最初没有这个 validation,只少了 Notify UUID 也能没报错就保存。

**解决**:在保存时加 validation。4 个字段中只要有一个填了,其余的也强制要求。

```python
has_patterns = notify_on or notify_off or status_query
if has_patterns and not notify_uuid:
    errors["base"] = "notify_uuid_required"
```


### 3. 实战示例: uLamp 电视背光灯

那么下面是实际用这个集成把 AliExpress 背光灯(UAC088-YH-C0)接入 HA 的过程。

#### 协议是怎么搞清楚的

其实我也是第一次分析 BLE 协议..边问 AI 边一步步推进。万一有处境类似的朋友,过程也写一下。

**第 1 步:开启 HCI snoop log**
Android 里 `设置 → 开发者选项 → 启用蓝牙 HCI snoop 日志`打开。开启后手机里来回的蓝牙包都会保存到文件里。

**第 2 步:用专用 App 操作并捕获日志**
用厂家专用 App 做 ON/OFF 之类的动作。这时候发出去的包就是我们要找的数据。

**第 3 步:提取日志文件**
用 `adb pull /data/misc/bluetooth/logs/btsnoop_hci.log` 把文件导出,或者从开发者选项生成 bug report 提取。

**第 4 步:分析**
可以用 Wireshark 打开自己抠..但宝贵的周末不想盯着 hex 看..

就丢给 Claude Code 让它"帮我找 BLE GATT Write 包"。AI 自己解析了包结构,很快就搞定了。

这样搞清楚的包格式是这样的:

```
0A A5 <SEQ> <CMD> [<PARAMS>]
```

- `0AA5` — 协议头(固定)
- `SEQ` — 序列 ID(每次加 1,但不验证所以用固定值也能工作)
- `CMD 01` — 电源 ON/OFF(`01`=ON,`00`=OFF)
- `CMD FF` — 状态查询

HCI 日志里捕获的电源切换:

| 发送数据 | 动作 | Notify 响应 |
|------------|------|-------------|
| `0AA5120100` | OFF | `0AA5000A0000000000000000` |
| `0AA5130101` | ON | `0AA554050000640001001F` |

#### BLE Controller 配置值

| 项目 | 值 |
|------|-----|
| Service UUID | `0000ffb0-0000-1000-8000-00805f9b34fb` |
| Write Char UUID | `0000ffb1-0000-1000-8000-00805f9b34fb` |
| ON 数据 | `0aa5010101` |
| OFF 数据 | `0aa5010100` |
| Write With Response | **OFF**(只支持 0x04) |
| Notify UUID | `0000ffb2-0000-1000-8000-00805f9b34fb` |
| ON pattern | `0aa554` |
| OFF pattern | `0aa5000a` |
| 状态查询命令 | `0aa501ff` |
| Keep Alive | **ON** |
| Keepalive 周期 | **5 秒** |

#### 注意事项

- **电视 USB 电源**:通常电视关了 USB 电源也会断,设备就 off 了..但我的情况像前面说的让电视保持省电模式,USB 一直供电。所以才需要 BLE 控制。一般来说只能在电视开着时进行 BLE 控制
- **Keepalive 5 秒**:这个设备 10 秒周期还是会 idle disconnect。5 秒比较稳
- **必须关闭 Write With Response**:ffb1 Characteristic 的 Properties 是 0x06(Read + Write Without Response)。开了 Write With Response 会因为等 ACK 而 hang


### 4. 收尾

本来只是想随便弄个 On/Off 开关..结果学到了不少东西。

特别是物理设备的反馈循环总是很难转。改代码 → 在 HACS 里更新 -> 重启 Hass → 等 BLE 连接 → 测试 → 看日志..一个循环 5 分钟就没了,调试真的...不容易。

连接断、ACK 不来、同一资源并发访问会报错、每个设备的 idle timeout 不一样..

这个项目里学到的东西归纳一下:

1. **BLE Write 必须加超时。** ACK 不来 Lock 永远不会释放。
2. **Notify 订阅只订一次,连接保持期间一直保留。** 每次 start/stop 是 race condition 的温床。
3. **Keepalive 光检查不够。** 必须真的发数据才能维持连接。

BLE Controller 可以通过 [HACS](https://hacs.xyz/) 安装。如果想在 HA 中控制没有专用集成的 BLE 设备,去看看 [GitHub repo](https://github.com/Lemon-HACS/ble_controller) 吧!
