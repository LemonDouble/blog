---
title: "為了在 Home Assistant 中使用 AliExpress 買的電視背光燈而逆向藍牙協定的故事"
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
    - 智慧家庭
    - AliExpress
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

![AliExpress 商品圖片](./backlight.webp)

我在 AliExpress 上買了一個電視背光燈。

其實一開始我並不在意聯動不聯動的問題..反正只要電視後面有光，才區區2萬5千韓元[^1]？沒理由不買吧？

[^1]: 約合新臺幣570元左右。

隨便想想..要是能和電視內容即時聯動那當然好..

如果是 Chromecast 的話有 HDMI 線，掛個 HDMI 鉤子用就行，但我的電視沒有那種東西，是 Android TV。那麼大概？透過 ADB 連線或者裝個單獨的 App，擷取螢幕邊緣的資訊然後即時聯動..？

但是，2萬5千韓元的東西要求那麼多就過分了吧？所以~大概是把攝影機感測器貼在一起，攝影機辨識到的顏色直接打到 LED 上的簡單結構。

因為只拍電視的中央部分，所以和電視顏色完全 Sync 的感覺差得有點遠，不過這無所謂，重要的是電視後面有光，嗯嗯。

但是？

![App / 遙控器圖片](./image1.png)

其實是收到貨之後才知道，居然有專用 App 和遙控器之類的東西。遙控器沒什麼用就丟了..專用 App 試了一下，發現是用藍牙連線的。

不過？在 AliExpress？按銷量排序排在最上面..想著說不定有哪位民間高手做了個可疑的10星專用整合什麼的，內心抱著期待..翻來翻去也沒找到那種東西..

倒是有個會跳10次藍牙權限確認的廉價專用 App..但努力翻了整合，似乎也沒有。

但忽然想到..

- 藍牙訊號大致就是往周圍撒的..可以從其他地方接收來分析
- 而且？既然有 App，而且藍牙訊號是從我手機裡發出去的？估計扒一扒 Android 的除錯日誌就能知道訊號
- 那麼，雖然不太清楚？如果能想辦法捕獲 ON/OFF 訊號，就可以接到 Home Assistant 上吧？

就這麼突然冒出了這個想法，但是...

為了關掉一個2萬5千韓元的 LED 裝置折騰這麼多？抱著這種心情瞇著眼睛假裝沒看見....

<img src="./image2.png" style="width: 400px;" alt="電視後面的背光燈關不掉">

糟糕..背光燈關不掉了。

按理說

- 把 USB 電源線插到電視上
- 電視關了 USB 也就不供電了
- 電視一關背光燈也就`聰明`地關掉

應該是這樣的工作流..但我的情況是

- 我用 Android Debug Bridge 把 Home Assistant 和電視聯動起來。用這個開電視什麼的做各種事情
- 但是？電視完全關掉的話 ADB 就連不上
- 之前做的所有自動化都用不了
- 所以？我就讓電視保持在省電模式
- 省電模式嘛？有電流通過，USB 也就有電
- 背光燈永遠關不掉

我就這樣開始走上了痛苦的迴圈..

當然有 App..LED 控制器上也有實體開關..但是麻煩啊...

所以咬著牙和永遠開著的 LED 背光燈一起生活..感覺睡眠時間慢慢變少了，啊..趁週末來折騰一下吧..就這樣開始了。

不過既然要做？藍牙訊號讀寫本來就是很常見的場景嘛？所以就找有沒有現成的能寫藍牙訊號的，意外地幾乎沒有。

那乾脆..做個通用的能用很久，這樣想著就動手了。

所以？這篇文章就是在做 [BLE Controller](https://github.com/Lemon-HACS/ble_controller) 時遇到的技術問題及解決過程的整理。


那麼要折騰..首先得知道..怎麼扒藍牙封包..吧？

我以前..在學校做專案時倒是做過 BLE Scan，但實際上從來沒扒過封包。(都在函式庫的上層玩)

所以看到 Raw 就有點茫然..

先在 Android 裡找到藍牙除錯，做個日誌檔 (Android 裡叫做 bug report)..

然後整個 zip 檔就直接丟給 Claude。現在情況這樣..開關了4次左右，有什麼嗎？

它說能看到日誌，我就開始瞎折騰。能跑通最好，跑不通也能在折騰中學到東西，沒什麼損失吧？

### 1. 大致結構

在故障排除故事之前..先大致過一下整體結構，要理解的話先整理一下 BLE 相關術語。

**BLE (Bluetooth Low Energy)** — 藍牙低功耗版本。主要用在低功耗小型裝置上的協定。在功耗很重要的 IoT 中也常用。

**GATT (Generic Attribute Profile)** — 與 BLE 裝置交換資料的規約。簡單來說就是定義了「對這個裝置的某地址發什麼內容能幹什麼」。結構是這樣的：
- **Service** — 功能集合(例如：「LED 控制服務」)。有一個叫 UUID 的唯一地址
- **Characteristic** — Service 中的具體功能(例如：「電源 ON/OFF」)。也有 UUID
- 最終只要知道 **Service UUID + Characteristic UUID + 要發的 hex 資料**這三樣，就能控制任何裝置

**BlueZ** — Linux 的藍牙堆疊。Home Assistant 是基於 Linux 的，所以 BLE 通訊都要經過 BlueZ。偶爾會吐一些奇怪的錯..這個後面再說。

所以做了一個讓使用者從 UI 直接輸入這三個值的結構。

```
config_flow.py   ← UI 設定(裝置掃描、UUID/資料輸入)
__init__.py      ← 進入點設定(建立 BLEDeviceManager)
ble_client.py    ← 管理所有 BLE 連線
switch.py        ← 開關實體
button.py        ← 按鈕實體
select.py        ← 選擇實體
```

#### BLEDeviceManager — 連線的心臟

`ble_client.py` 中的 `BLEDeviceManager` 管理所有 BLE 連線。

```python
class BLEDeviceManager:
    def __init__(self, hass, mac, *, keep_alive=False, keepalive_interval=10):
        self._client: BleakClientWithServiceCache | None = None
        self._connect_lock = asyncio.Lock()      # 連線串列化
        self._operation_lock = asyncio.Lock()     # GATT 操作串列化
        self._disconnect_timer: asyncio.TimerHandle | None = None
```

這裡兩個 Lock 是關鍵：
- **`_connect_lock`**：連線/中斷一次只處理一個。兩個地方同時嘗試連線 BlueZ 會亂掉。
- **`_operation_lock`**：GATT Write 也是一次一個。BLE 同時發多條命令會被丟棄。

#### 兩種連線模式

**On-demand 模式**(預設)：只在發命令時連線，15秒安靜後自動斷開。BLE 是 1:1 連線，HA 佔著連線的話廠家 App 就無法存取。需要並行使用時很有用。

```python
def _reset_disconnect_timer(self):
    if self._keep_alive:
        return  # keep_alive 模式下不需要計時器
    self._cancel_disconnect_timer()
    self._disconnect_timer = loop.call_later(
        DISCONNECT_DELAY,  # 15 秒
        lambda: asyncio.ensure_future(self._timed_disconnect()),
    )
```

**Keep Alive 模式**：HA 啟動後立即連線，背景週期性檢查連線狀態。斷了會自動重連。如果不打算用廠家 App，想讓 HA 全部控制，這種方式回應更快。

```python
async def _keepalive_loop(self):
    if await self._ensure_connected():
        await self._fire_on_connect()  # 初始狀態查詢
    while True:
        await asyncio.sleep(self._keepalive_interval)
        if self._client is None or not self._client.is_connected:
            if await self._ensure_connected():
                await self._fire_on_connect()
        else:
            await self._fire_on_connect()  # 用 ping 維持連線
```


### 2. 故障排除之旅

從這裡開始..折騰了相當多..

老實說我沒想過為了一個 On/Off 開關要折騰這麼多..

#### 2-1. Write With Response 無限 hang

**症狀**：第一條命令好好的。但從第二條開始永遠沒回應。

**原因**：測試裝置(uLamp 電視背光燈)的 Write Characteristic(功能)只支援 `Write Without Response`(0x04)，但我把 `Write With Response` 設成了 ON。

簡單說明一下..BLE Characteristic 有個 **Properties** 屬性。定義「給我發資料時用這種方式」。主要有兩種：
- **Write Without Response (0x04)** — 發了就完事。不管裝置有沒有收到
- **Write With Response (0x08)** — 發了之後等「已收到」的 ACK

可以用 [nRF Connect](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-mobile) 這個 App 確認..

<img src="./nrf.png" style="width: 400px;" alt="在 nrf 中看到的資料">

那裡寫的 NO RESPONSE 就是不給 ACK 的意思。

那 0xFFB1 是 On/Off 我是怎麼知道的？-> 這個是從 HCI 日誌裡看出來的。

總之結論..呼叫 `write_gatt_char(response=True)` 會向裝置發 Write Request 並等 ACK..但裝置如果不支援，**ACK 永遠不會來**。

而且為了等這個 ACK 內部一直佔著鎖，後面來的 Write 命令也全都進了等待狀態。第一條命令永遠完不了，第二條第三條也一個個堵住..

**解決**：給所有 GATT Write 加了 3 秒逾時。

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
            _LOGGER.error("GATT write 逾時 — 強制 disconnect")
            await self._disconnect()
            return False
```

逾時就強制中斷連線釋放 Lock，下一條命令重新連線。

**教訓**：控制 BLE 裝置前，務必用 nRF Connect 之類的 App 檢查 Characteristic Properties。先看清楚裝置給不給 ACK..


#### 2-2. 0x0e Unlikely Error — Notify 訂閱衝突

**症狀**：間歇性出現 `BleakDBusError: 0x0e Unlikely Error`。

間歇性的更讓人煩躁(..)

`0x0e Unlikely Error` 是什麼呢..是 BlueZ(Linux 藍牙堆疊)在認為「這種情況不應該發生」時吐出的錯誤。名字叫 Unlikely 聽著挺好笑，但實戰中其實經常碰到(..)

**原因**：BLE 有個 **Notify** 功能。裝置主動告訴你「狀態變了！」——要接收這個訊息，需要用 `start_notify()` 訂閱。

問題是這個訂閱會從多個地方同時進行。背景週期性檢查狀態的迴圈和使用者按開關時執行的程式碼同時嘗試訂閱同一個 Characteristic(功能)..BlueZ 就拋出 `0x0e`。

最初的實作是每次發資料時：
1. `start_notify()` — 開始訂閱
2. 發送資料
3. 等待 Notify 回應
4. `stop_notify()` — 取消訂閱

這個流程在兩個地方同時執行就衝突。

**解決**：不再每次都訂閱/取消，改成連線時只訂閱一次並一直保持。簡單說就是一個地方持續接收 Notify 放到緩衝區，其他地方從緩衝區裡讀。

```python
async def _subscribe_persistent_notify(self):
    """連線後呼叫一次。連線保持期間訂閱一直保留。"""
    if not self._persistent_notify_uuid or self._notify_subscribed:
        return
    await self._client.start_notify(self._persistent_notify_uuid, self._on_notify)
    self._notify_subscribed = True

def _on_notify(self, _handle, data):
    """所有 Notify 接收都存到緩衝區。"""
    self._notify_data_buffer.append(bytes(data))
    self._notify_event.set()
```

訂閱/取消的反覆消失後錯誤也消失了。


#### 2-3. HA 實體狀態更新延遲

**症狀**：裝置馬上就開了，但 HA UI 的開關狀態過好一會兒才變。

體感上挺明顯的問題。

**原因**：`write_and_notify()` 在 Write 後會等 1.5 秒的 Notify 回應。如果 Notify 不來或者不匹配每次都要浪費 1.5 秒..

**解決**：改成 Write 之後立即樂觀更新狀態。前面說過 Write Without Response 模式下裝置不發「收到」的 ACK。所以沒法知道是否真的發出去了，但既然連線正常那大機率是發出去了吧~就先把 UI 更新掉。如果聯動正常，發出去的機率要高得多。(順便說，Twitter 的按讚/轉推也是這麼工作的。)

```python
# Before: 用 write_and_notify() 等 Notify 回應
ok, state = await self._manager.write_and_notify(...)
if state is not None:
    self._attr_is_on = state

# After: 立即反映 expected state
ok = await self._manager.write(self._char_uuid, data, response=self._response)
if ok:
    self._attr_is_on = expected_on  # 樂觀更新
self.async_write_ha_state()
```

要是裝置實際狀態確實不一樣呢？Keepalive 迴圈會週期性取得裝置狀態，以那個為準矯正。結果就是 UI 立即回應，實際狀態的偏差幾秒內就會被糾正。


#### 2-4. BLE 連線中斷 — Keepalive 沒在 Keep Alive 的問題

**症狀**：開了 Keep Alive，但「意外斷開連線」日誌每 2 分鐘出現 8~12 次。

欸 Keep Alive 怎麼連線還會斷，那不就不是 KeepAlive 了嗎？

**原因**：初始實作只檢查連線狀態沒發資料。BLE 裝置一段時間沒資料交換的話會因為 **idle timeout** 把連線斷掉。(為了省電嘛)

```python
# Before: 只檢查連線狀態
while True:
    await asyncio.sleep(60)
    if not self._client.is_connected:
        await self._ensure_connected()  # 斷了之後才重連
```

**解決**：讓 Keepalive 迴圈週期性地**發送實際資料**。

```python
# After: 週期性發送 status query (ping)
while True:
    await asyncio.sleep(self._keepalive_interval)  # 10 秒(可設定)
    if not self._client.is_connected:
        if await self._ensure_connected():
            await self._fire_on_connect()  # 重連 + 狀態查詢
    else:
        await self._fire_on_connect()  # ping! 維持連線 + 重新整理狀態
```

這樣做結果一舉兩得：
1. 週期性資料發送讓裝置不會中斷連線
2. 狀態查詢的回應讓 HA 實體狀態也始終是最新的


#### 2-5. Keepalive 週期 — 每個裝置都不一樣

**症狀**：Keepalive 週期從 60 秒 → 30 秒 → 15 秒 → 10 秒一路減，還是會斷。

**原因**：我的背光燈就算 10 秒也經常斷..減到 5 秒才穩定..BLE 裝置的 idle timeout 各不相同。

其實最初做 Android HCI 日誌 dump 時如果把連線靜置一會兒就能知道連線是怎麼維持的..但當時嫌麻煩就只抓了連線 + 開關，這是失策..

**解決**：把 Keepalive 週期做成每個裝置可設定。

```python
class BLEDeviceManager:
    def __init__(self, hass, mac, *, keep_alive=False, keepalive_interval=10):
        self._keepalive_interval = keepalive_interval
```

在 Config Flow 和 Options Flow 中都加了 `keepalive_interval` 欄位，每個裝置都能設定最佳值。預設 10 秒，不穩定的話建議 5~7 秒。

#### 2-6. 沒有 Notify UUID 只輸入 pattern 會被悄悄忽略的問題

**症狀**：輸入了 Notify ON/OFF pattern 但狀態確認不工作。也不報錯就這麼悄悄被忽略，所以原因很難找。

**原因**：Notify 相關設定因為有些裝置不支援所以做成 Optional 了..但要用 Notify，**Notify UUID、ON pattern、OFF pattern、狀態查詢命令**這 4 個必須湊齊一套。少一個都不工作，但最初沒有這個 validation，只少了 Notify UUID 也能沒報錯就儲存。

**解決**：在儲存時加 validation。4 個欄位中只要有一個填了，其餘的也強制要求。

```python
has_patterns = notify_on or notify_off or status_query
if has_patterns and not notify_uuid:
    errors["base"] = "notify_uuid_required"
```


### 3. 實戰範例：uLamp 電視背光燈

那麼下面是實際用這個整合把 AliExpress 背光燈(UAC088-YH-C0)接入 HA 的過程。

#### 協定是怎麼搞清楚的

其實我也是第一次分析 BLE 協定..邊問 AI 邊一步步推進。萬一有處境類似的朋友，過程也寫一下。

**第 1 步：開啟 HCI snoop log**
Android 裡 `設定 → 開發人員選項 → 啟用藍牙 HCI snoop 紀錄` 開啟。開啟後手機裡來回的藍牙封包都會儲存到檔案裡。

**第 2 步：用專用 App 操作並擷取日誌**
用廠家專用 App 做 ON/OFF 之類的動作。這時候發出去的封包就是我們要找的資料。

**第 3 步：擷取日誌檔案**
用 `adb pull /data/misc/bluetooth/logs/btsnoop_hci.log` 把檔案匯出，或者從開發人員選項生成 bug report 擷取。

**第 4 步：分析**
可以用 Wireshark 開啟自己摳..但寶貴的週末不想盯著 hex 看..

就丟給 Claude Code 讓它「幫我找 BLE GATT Write 封包」。AI 自己解析了封包結構，很快就搞定了。

這樣搞清楚的封包格式是這樣的：

```
0A A5 <SEQ> <CMD> [<PARAMS>]
```

- `0AA5` — 協定標頭(固定)
- `SEQ` — 序列 ID(每次加 1，但不驗證所以用固定值也能工作)
- `CMD 01` — 電源 ON/OFF(`01`=ON，`00`=OFF)
- `CMD FF` — 狀態查詢

HCI 日誌裡擷取的電源切換：

| 傳送資料 | 動作 | Notify 回應 |
|------------|------|-------------|
| `0AA5120100` | OFF | `0AA5000A0000000000000000` |
| `0AA5130101` | ON | `0AA554050000640001001F` |

#### BLE Controller 設定值

| 項目 | 值 |
|------|-----|
| Service UUID | `0000ffb0-0000-1000-8000-00805f9b34fb` |
| Write Char UUID | `0000ffb1-0000-1000-8000-00805f9b34fb` |
| ON 資料 | `0aa5010101` |
| OFF 資料 | `0aa5010100` |
| Write With Response | **OFF**(只支援 0x04) |
| Notify UUID | `0000ffb2-0000-1000-8000-00805f9b34fb` |
| ON pattern | `0aa554` |
| OFF pattern | `0aa5000a` |
| 狀態查詢命令 | `0aa501ff` |
| Keep Alive | **ON** |
| Keepalive 週期 | **5 秒** |

#### 注意事項

- **電視 USB 電源**：通常電視關了 USB 電源也會斷，裝置就 off 了..但我的情況像前面說的讓電視保持省電模式，USB 一直供電。所以才需要 BLE 控制。一般來說只能在電視開著時進行 BLE 控制
- **Keepalive 5 秒**：這個裝置 10 秒週期還是會 idle disconnect。5 秒比較穩
- **必須關閉 Write With Response**：ffb1 Characteristic 的 Properties 是 0x06(Read + Write Without Response)。開了 Write With Response 會因為等 ACK 而 hang


### 4. 收尾

本來只是想隨便弄個 On/Off 開關..結果學到了不少東西。

特別是實體裝置的回饋迴圈總是很難轉。改程式碼 → 在 HACS 裡更新 -> 重啟 Hass → 等 BLE 連線 → 測試 → 看日誌..一個迴圈 5 分鐘就沒了，除錯真的...不容易。

連線斷、ACK 不來、同一資源並行存取會報錯、每個裝置的 idle timeout 不一樣..

這個專案裡學到的東西歸納一下：

1. **BLE Write 必須加逾時。** ACK 不來 Lock 永遠不會釋放。
2. **Notify 訂閱只訂一次，連線保持期間一直保留。** 每次 start/stop 是 race condition 的溫床。
3. **Keepalive 光檢查不夠。** 必須真的發資料才能維持連線。

BLE Controller 可以透過 [HACS](https://hacs.xyz/) 安裝。如果想在 HA 中控制沒有專用整合的 BLE 裝置，去看看 [GitHub repo](https://github.com/Lemon-HACS/ble_controller) 吧！
