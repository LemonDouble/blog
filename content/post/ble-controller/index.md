---
title: "알리에서 산 TV 백라이트를 Home Assistant에서 쓰고 싶어서 블루투스 프로토콜 뜯어본 이야기"
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
    - 스마트홈
    - AliExpress
---

![알리 상품 이미지](./backlight.webp)

AliExpress에서 TV 백라이트를 하나 샀습니다.

사실 처음에는 연동이 되는건 별로 신경쓰지 않았고.. 대충 TV 뒤에 빛 나는데 단돈 2만5천원? 안 살 이유가 없잖아요?

대충.. TV 내용이랑 실시간으로 연동되면 정말 좋겠지만.. 

크롬캐스트면 대충 HDMI 선 있으니까 HDMI 후킹해서 쓰면 될 것 같은데, 제 TV는 그런거 없고 안드로이드 TV란 말이죠. 그러면 대충? ADB로 연결하든 별도 앱을 깔던 해서 화면 끝부분 정보 추출한 뒤 그걸 실시간으로 연동해서..?

는 대충 2만5천원짜리에 거기까지 바라면 안 되겠죠? 그래서~ 아마 카메라 센서랑 딱 붙여서 카메라가 인식한 색깔로 그대로 LED에 쏴 주는 간단한 구조인 것 같았습니다.

소소하게 그래서 TV 중앙부만 카메라가 찍으니.. 완전 TV 색깔과 완벽하게 Sync된다는 느낌과는 거리가 먼 느낌이 있지만 아무래도 그건 상관없고 TV 뒤에 불이 나온다는게 중요한거지요 암암.

그런데?

![앱/리모콘 이미지](./image1.png)

사실 배송받고나서 알았는데 뭔가 전용앱 / 리모컨같은게 있더라구요. 리모컨은 사실 쓸 일 없어서 갖다버렸고.. 전용 앱은 근데 써 보니까 블루투스로 연결하는걸 봤습니다.

근데? 알리에서? 판매량 순으로 정렬해서 제일 위에 올라가있으니.. 어디 재야의 고수가 만들어놓은 수상한 스타 10개짜리 전용 통합같은게 없을까 하고 내심 기대했는데.. 뒤적뒤적 뒤져봤는데 그런건 없었고..

블루투스 권한 확인을 10번정도 띄우는 싸구려 전용앱이 있긴 한데.. 통합 열심히 뒤져봤는데도 없는 것 같더라구요.

근데 문득 생각이 드는게.. 

- 블루투스 시그널은 대충 주변에다가 냅다 뿌리는거라.. 대충 다른데서 받아서 까볼 수 있음
- 심지어? 앱이 있고 블루투스 시그널이 내 폰에서 나가는거니까? 아마 안드로이드 디버그 로그 뜯어보면 시그널도 알 수 있을것
- 그럼 잘은 모르겠지만? ON/OFF 시그널만 어케 캡쳐하면 HomeAssistant에 붙일 수 있는거 아님?

이라는 생각이 문득 들었지만...

2만5천원짜리 LED 디바이스 하나 붙여서 끄겠다고 저만한 삽질을? 싶어서 흐린눈을 하고 있었는데....

<img src="./image2.png" style="width: 400px;" alt="TV 뒤에 백라이트가 안꺼져요 ㅠㅠ">

아뿔사.. 백라이트가 안꺼집니다.

원래대로라면

- USB 전원선을 TV에 꼽음
- TV가 꺼지면 USB에 전원도 공급 안 함
- TV가 꺼지면 `스마트` 하게 백라이트도 꺼짐

이런 워크플로운데요.. 저같은 경우는

- Android Debug Bridge로 홈어시스턴트랑 TV를 연동해둠. 이걸로 TV도 키고 뭐 이것저것 함
- 근데? 아예 TV가 완전히 꺼져버리면 ADB 연결이 안 됨
- 지금까지 만들어놨던 자동화를 못씀
- 그래서? TV를 절전모드로 살려놓음
- 절전모드라? 전원이 가니까 USB에도 전원이 감
- 백라이트가 영원히 안꺼짐

이런 고통의 굴레를 걷게 된 것입니다..

물론 앱도 있고.. 그 LED 컨트롤러에 물리적인 스위치도 있는데.. 귀찮잖아요...

그래서 이악물고 영원히 켜져있는 LED 백라이트와의 동거를 하다.. 수면시간이 점점 줄어드는 느낌이 나 아.. 주말을 맞아 삽질을 한번 해 보자.. 하고 시작하게 되었습니다.

근데 이왕 만드는거? 블루투스 신호 쓰고 빼는거 되게 흔한 케이스란말이죠? 그래서 블루투스 신호 WRITE 되는게 뭐 있나 헀는데 의외로 잘 없더라구요. 

그래서 겸사겸사.. 범용적인거 만들어서 두고두고 쓰자 하고 덤벼보았읍니다.

그래서? 이 글은 [BLE Controller](https://github.com/Lemon-HACS/ble_controller)를 만들면서 겪은 기술적 문제들과 해결 과정을 정리한 글입니다.

---

자 근데.. 삽질을 하려면.. 일단 블루투스 패킷 어떻게 뜯음..? 부터는 알아야하잖아요?

저도 옛날옛적.. 학교에서 프로젝트 한다고 BLE Scan은 해 본 적 있는데, 실제로 패킷을 뜯어본적은 없었거든요. (라이브러리 위에서 놀았으니)

그래서 Raw 보려니까 막막한게 있는고로.. 

일단 안드로이드에서 블루투스 디버깅 찾아서 로그 파일 만든 뒤 (안드로이드에서는 버그 리포트라고 부르더라구요)..

그냥 냅다 클로드에 zip파일쨰로 던졌습니다. 지금 상황 이렇고.. 껐다켰다 네번정도 했는데 뭐 있냐고..

그랬더니 로그 보인다고 해서 일단 냅다 삽질부터 해 봤읍니다. 이거 작동되면 좋은거고, 아니어도 삽질하면서 이것저것 배울테니 손해볼건 없잖아요?

### 1. 대충 어떻게 생겼나

트러블슈팅 이야기 전에.. 전체 구조를 대충 훑고 갈건데, 이해를 하려면 BLE 관련 용어를 좀 정리해볼게요.

**BLE (Bluetooth Low Energy)** — 블루투스 저전력 버전. 저전력 소형 디바이스에서 주로 쓰는 프로토콜이에요. 소비전력 중요한 IoT에서도 많이 씁니다.

**GATT (Generic Attribute Profile)** — BLE 디바이스랑 데이터를 주고받는 규약입니다. 쉽게 말하면 "이 디바이스한테 어떤 주소로 뭘 보내면 뭘 할 수 있다"를 정의한 거예요. 구조가 이렇게 생겼는데:
- **Service** — 기능 묶음 (예: "LED 제어 서비스"). UUID라는 고유 주소가 있음
- **Characteristic** — Service 안의 개별 기능 (예: "전원 ON/OFF"). 얘도 UUID가 있음
- 결국 **Service UUID + Characteristic UUID + 보낼 hex 데이터**, 이 세 가지만 알면 어떤 디바이스든 제어할 수 있습니다

**BlueZ** — 리눅스의 블루투스 스택. Home Assistant가 리눅스 기반이라 BLE 통신은 전부 BlueZ를 거쳐요. 가끔 이상한 에러를 뱉는데.. 그건 뒤에서 다룰게요.

그래서 이 세 가지 값을 UI에서 직접 입력받는 구조로 만들었어요.

```
config_flow.py   ← UI 설정 (디바이스 스캔, UUID/데이터 입력)
__init__.py      ← 엔트리 셋업 (BLEDeviceManager 생성)
ble_client.py    ← 모든 BLE 연결 관리
switch.py        ← 스위치 엔티티
button.py        ← 버튼 엔티티
select.py        ← 셀렉트 엔티티
```

#### BLEDeviceManager — 연결의 심장

`ble_client.py`의 `BLEDeviceManager`가 모든 BLE 연결을 관리합니다.

```python
class BLEDeviceManager:
    def __init__(self, hass, mac, *, keep_alive=False, keepalive_interval=10):
        self._client: BleakClientWithServiceCache | None = None
        self._connect_lock = asyncio.Lock()      # 연결 직렬화
        self._operation_lock = asyncio.Lock()     # GATT 오퍼레이션 직렬화
        self._disconnect_timer: asyncio.TimerHandle | None = None
```

여기서 두 개의 Lock이 핵심인데요:
- **`_connect_lock`**: 연결/해제를 한 번에 하나씩만 처리합니다. 동시에 두 곳에서 연결 시도하면 BlueZ가 혼란스러워하거든요.
- **`_operation_lock`**: GATT Write도 한 번에 하나씩만. BLE는 동시에 여러 명령을 보내면 씹어버려요.

#### 두 가지 연결 모드

**On-demand 모드** (기본): 명령 보낼 때만 연결하고, 15초간 조용하면 자동 해제. BLE는 1:1 연결이라 HA가 연결을 물고 있으면 제조사 앱에서 접근이 안 되거든요. 병행해서 쓸 때 유용합니다.

```python
def _reset_disconnect_timer(self):
    if self._keep_alive:
        return  # keep_alive 모드에서는 타이머 불필요
    self._cancel_disconnect_timer()
    self._disconnect_timer = loop.call_later(
        DISCONNECT_DELAY,  # 15초
        lambda: asyncio.ensure_future(self._timed_disconnect()),
    )
```

**Keep Alive 모드**: HA 시작하면 바로 연결하고, 백그라운드에서 주기적으로 연결 상태 확인. 끊어지면 알아서 재연결합니다. 제조사 앱같은거 쓸 생각 없고, HA가 모두 제어하게 하려면 요렇게 하는게 동작이 빠릿빠릿해요.

```python
async def _keepalive_loop(self):
    if await self._ensure_connected():
        await self._fire_on_connect()  # 초기 상태 조회
    while True:
        await asyncio.sleep(self._keepalive_interval)
        if self._client is None or not self._client.is_connected:
            if await self._ensure_connected():
                await self._fire_on_connect()
        else:
            await self._fire_on_connect()  # ping으로 연결 유지
```

---

### 2. 트러블슈팅 여정

여기서부터.. 삽질을 좀 많이 했는데요.. 

솔직히 난 On/Off 하나 만들자고 이렇게까지 삽질할줄은 몰랐지..

#### 2-1. Write With Response 무한 hang

**증상**: 첫 번째 명령은 잘 됩니다. 근데 두 번째부터 영원히 응답이 없어요.

**원인**: 테스트 디바이스(uLamp TV 백라이트)의 Write Characteristic (기능)이 `Write Without Response`(0x04)만 지원하는데, `Write With Response`를 ON으로 설정한 상태였거든요.

잠깐 설명하면.. BLE Characteristic에는 **Properties**라는 속성이 있습니다. "나한테 데이터를 보낼 때 이런 방식으로 보내라"를 정의하는 건데요. 주요한 건 두 가지:
- **Write Without Response (0x04)** — 데이터를 보내고 끝. 디바이스가 받았는지 확인 안 함
- **Write With Response (0x08)** — 데이터를 보내고 "잘 받았음" ACK를 기다림

이걸 [nRF Connect](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-mobile)라는 앱으로 확인할 수 있는데..

<img src="./nrf.png" style="width: 400px;" alt="nrf에서 본 데이터">

대략 저기 그여있는 NO RESPONSE가 ACK 안 준단 뜻이네요.

근데 OxFFB1이 On/Off인건 어떻게 알았냐고요? -> 요건 HCI 로그로 알았습니다.

어쨌든 결론적으로.. `write_gatt_char(response=True)`를 호출하면 디바이스에 Write Request를 보내고 ACK를 기다리는데.. 디바이스가 이걸 지원하지 않으면 **ACK가 영원히 안 옵니다**.

게다가 이 ACK를 기다리느라 내부적으로 락을 잡고 있으니까, 뒤이어 오는 Write 명령들도 전부 대기 상태에 빠져요. 첫 번째 명령이 영원히 안 끝나니까 두 번째, 세 번째도 줄줄이 막히는 거죠..

**해결**: 모든 GATT Write에 3초 타임아웃을 걸었습니다.

```python
async def write(self, char_uuid, data, response=False):
    async with self._operation_lock:
        if not await self._ensure_connected():
            return False
        try:
            await asyncio.wait_for(
                self._client.write_gatt_char(char_uuid, data, response=response),
                timeout=GATT_TIMEOUT,  # 3초
            )
            return True
        except TimeoutError:
            _LOGGER.error("GATT write 타임아웃 — 강제 disconnect")
            await self._disconnect()
            return False
```

타임아웃 나면 연결을 강제로 끊어서 Lock을 풀고, 다음 명령에서 새로 연결하도록 했습니다.

**교훈**: BLE 디바이스를 제어하기 전에, nRF Connect 같은 앱으로 Characteristic Properties를 꼭 확인합시다. 디바이스가 ACK 주는지를 일단 좀 봐야합니다..

---

#### 2-2. 0x0e Unlikely Error — Notify 구독 충돌

**증상**: 간헐적으로 `BleakDBusError: 0x0e Unlikely Error` 발생.

간헐적이라서 더 짜증나는 에러였습니다(..)

`0x0e Unlikely Error`가 뭐냐면.. BlueZ(리눅스 블루투스 스택)가 "이건 일어나면 안 되는 상황인데?" 하고 뱉는 에러입니다. 이름이 Unlikely인게 웃기지만 실전에서는 꽤 자주 마주칩니다(..)

**원인**: BLE에는 **Notify**라는 기능이 있습니다. 디바이스가 "상태 바뀌었어!"하고 먼저 알려주는 건데, 이걸 받으려면 `start_notify()`로 구독을 걸어야 하거든요.

문제는 이 구독을 여러 곳에서 동시에 거는 경우였어요. 백그라운드에서 상태를 주기적으로 확인하는 루프와, 사용자가 스위치를 눌렀을 때 실행되는 코드가 동시에 같은 Characteristic(기능)에 구독을 시도하면.. BlueZ가 `0x0e`를 던집니다.

원래 구현은 데이터를 보낼 때마다:
1. `start_notify()` — 구독 시작
2. 데이터 전송
3. Notify 응답 대기
4. `stop_notify()` — 구독 해제

이걸 두 곳에서 동시에 실행하면 충돌이 나는 거죠.

**해결**: 구독을 매번 걸었다 풀었다 하지 말고, 연결할 때 딱 1회만 구독해놓고 쭉 유지하는 방식으로 바꿨습니다. 쉽게 말하면 한 곳에서 Notify를 계속 받아서 버퍼에 쌓아두고, 나머지는 그 버퍼에서 읽어가는 구조예요.

```python
async def _subscribe_persistent_notify(self):
    """연결 직후 1회 호출. 연결 유지 동안 구독 유지."""
    if not self._persistent_notify_uuid or self._notify_subscribed:
        return
    await self._client.start_notify(self._persistent_notify_uuid, self._on_notify)
    self._notify_subscribed = True

def _on_notify(self, _handle, data):
    """모든 Notify 수신을 버퍼에 저장."""
    self._notify_data_buffer.append(bytes(data))
    self._notify_event.set()
```

구독/해제 반복이 사라지니 에러도 사라졌어요.

---

#### 2-3. HA 엔티티 상태 업데이트 지연

**증상**: 디바이스는 바로 켜지는데, HA UI의 스위치 상태가 한참 뒤에 바뀝니다.

체감상 꽤 거슬리는 문제였어요.

**원인**: `write_and_notify()`에서 Write 후 Notify 응답을 1.5초까지 기다리고 있었거든요. Notify 응답이 안 오거나 매칭이 안 되면 매번 1.5초씩 날리는 구조..

**해결**: Write 후 바로 optimistic하게 상태를 업데이트하도록 바꿨습니다. 위에서 말했듯이 Write Without Response 모드에서는 디바이스가 "잘 받았어" ACK를 안 보내거든요. 그래서 제대로 보내졌는지는 알 방법이 없지만? 정상 연결됐으니 대충 보내졌겠지~ 하고 일단 UI부터 냅다 업데이트하는거에요. 정상 연동됐으면 보내졌을 확률이 훨씬 높으니까. (TMI지만, 트위터 마음찍기 / 리트윗 같은것도 이렇게 작동한답니다)

```python
# Before: write_and_notify()로 Notify 응답 대기
ok, state = await self._manager.write_and_notify(...)
if state is not None:
    self._attr_is_on = state

# After: 즉시 expected state 반영
ok = await self._manager.write(self._char_uuid, data, response=self._response)
if ok:
    self._attr_is_on = expected_on  # 낙관적 업데이트
self.async_write_ha_state()
```

만약 디바이스 상태가 실제로 다르면? 그건 Keepalive 루프에서 주기적으로 디바이스 상태를 받아오니, 그걸 기준으로 교정합니다. 결과적으로 UI는 즉시 반응하고, 실제 상태와의 차이는 몇 초 내에 보정되는 구조예요.

---

#### 2-4. BLE 연결 끊김 — Keepalive가 Keep Alive를 안 하던 문제

**증상**: Keep Alive 켰는데 "예기치 않은 연결 끊김" 로그가 2분에 8~12회씩 계속 찍힘.

아니 Keep Alive인데 연결이 끊기면 KeepAlive가 아니지 않나? 

**원인**: 초기 구현은 연결 상태만 체크하고 데이터를 안 보냈습니다. BLE 디바이스는 일정 시간 데이터 교환이 없으면 **idle timeout**으로 연결을 끊어버리거든요. (배터리 아껴야죠)

```python
# Before: 연결 상태만 체크
while True:
    await asyncio.sleep(60)
    if not self._client.is_connected:
        await self._ensure_connected()  # 끊어진 후에야 재연결
```

**해결**: Keepalive 루프에서 주기적으로 **실제 데이터를 전송**하도록 바꿨습니다.

```python
# After: 주기적으로 status query 전송 (ping)
while True:
    await asyncio.sleep(self._keepalive_interval)  # 10초 (설정 가능)
    if not self._client.is_connected:
        if await self._ensure_connected():
            await self._fire_on_connect()  # 재연결 + 상태 조회
    else:
        await self._fire_on_connect()  # ping! 연결 유지 + 상태 갱신
```

이렇게 하니 근데 결과적으로.. 두 마리 토끼를 잡은게:
1. 주기적 데이터 전송으로 디바이스가 연결을 안 끊음
2. 상태 조회 응답으로 HA 엔티티 상태도 항상 최신

---

#### 2-5. Keepalive 주기 — 디바이스마다 다르더라

**증상**: Keepalive 주기를 60초 → 30초 → 15초 → 10초로 줄여봤는데 여전히 끊김.

**원인**: 제 백라이트는 10초인데도 자주 끊겨서.. 5초로 줄이니까 그제서야 안정적.. BLE 디바이스마다 idle timeout이 다르더라구요.

사실 이건 처음에 안드로이드 HCI 로그 덤프뜰 때 연결을 가만히 냅둬봤으면 연결 어떻게 유지하는지 알 수 있었을 텐데.. 귀찮아서 대충 연결 + 껐다켰다만 캡처한 게 잘못했던 것 같읍니다.. 

**해결**: Keepalive 주기를 디바이스별로 설정 가능하게 뺐습니다.

```python
class BLEDeviceManager:
    def __init__(self, hass, mac, *, keep_alive=False, keepalive_interval=10):
        self._keepalive_interval = keepalive_interval
```

Config Flow와 Options Flow 모두에 `keepalive_interval` 필드를 넣어서, 디바이스마다 최적 값을 설정할 수 있게 했습니다. 기본값은 10초, 불안정하면 5~7초 권장.

#### 2-6. Notify UUID 없이 패턴만 입력하면 조용히 무시되는 문제

**증상**: Notify ON/OFF 패턴을 입력했는데 상태 확인이 안 됩니다. 에러도 안 나고 그냥 조용히 무시돼서 원인 찾기가 어려웠어요.

**원인**: Notify 관련 설정은 지원하지 않는 디바이스도 있어서 Optional로 빼긴 했는데.. Notify를 쓰려면 **Notify UUID, ON 패턴, OFF 패턴, 상태 조회 커맨드** 이 4개가 세트로 다 있어야 합니다. 하나라도 빠지면 동작을 안 하는데, 처음에는 이 validation이 없어서 Notify UUID만 빠뜨려도 아무 에러 없이 저장이 됐거든요.

**해결**: 저장 시점에 validation 추가. 4개 필드 중 하나라도 입력됐으면 나머지도 필수로 요구하도록.

```python
has_patterns = notify_on or notify_off or status_query
if has_patterns and not notify_uuid:
    errors["base"] = "notify_uuid_required"
```

---

### 3. 실전 예시: uLamp TV 백라이트

자 그러면 실제로 이 통합을 써서 AliExpress 백라이트(UAC088-YH-C0)를 HA에 연동한 과정입니다.

#### 프로토콜은 어떻게 파악했나

사실 저도 BLE 프로토콜 분석은 처음이었는데.. AI한테 물어보면서 step-by-step으로 진행했습니다. 혹시 비슷한 상황인 분들을 위해 과정을 적어볼게요.

**1단계: HCI 스눕 로그 켜기**
안드로이드 `설정 → 개발자 옵션 → 블루투스 HCI 스눕 로그 사용`을 켭니다. 이걸 켜면 폰에서 오가는 블루투스 패킷을 전부 파일로 저장해줘요.

**2단계: 전용 앱으로 조작하면서 로그 캡처**
제조사 전용 앱으로 ON/OFF 같은 동작을 해봅니다. 이때 나가는 패킷이 곧 우리가 알아내야 할 데이터예요.

**3단계: 로그 파일 추출**
`adb pull /data/misc/bluetooth/logs/btsnoop_hci.log`로 파일을 뽑거나, 개발자 옵션에서 버그 리포트를 생성해서 추출할 수 있습니다.

**4단계: 분석**
Wireshark로 열어서 직접 끼볼수도 있지만.. 황금같은 주말에 hex 보면서 즐거운 시간 가지고싶진 않아서..

Claude Code에 던지고 "이거 BLE GATT Write 패킷 찾아줘"하고 시켰습니다. AI가 패킷 구조를 알아서 파싱해줘서 금방 끝났어요.

이렇게 파악한 패킷 포맷은 이렇게 생겼어요:

```
0A A5 <SEQ> <CMD> [<PARAMS>]
```

- `0AA5` — 프로토콜 헤더 (고정)
- `SEQ` — 시퀀스 ID (매번 1 증가, 근데 검증 안 해서 고정값 써도 동작은 함)
- `CMD 01` — 전원 ON/OFF (`01`=ON, `00`=OFF)
- `CMD FF` — 상태 조회

HCI 로그에서 캡처한 전원 토글:

| 전송 데이터 | 동작 | Notify 응답 |
|------------|------|-------------|
| `0AA5120100` | OFF | `0AA5000A0000000000000000` |
| `0AA5130101` | ON | `0AA554050000640001001F` |

#### BLE Controller 설정값

| 항목 | 값 |
|------|-----|
| Service UUID | `0000ffb0-0000-1000-8000-00805f9b34fb` |
| Write Char UUID | `0000ffb1-0000-1000-8000-00805f9b34fb` |
| ON 데이터 | `0aa5010101` |
| OFF 데이터 | `0aa5010100` |
| Write With Response | **OFF** (0x04만 지원) |
| Notify UUID | `0000ffb2-0000-1000-8000-00805f9b34fb` |
| ON 패턴 | `0aa554` |
| OFF 패턴 | `0aa5000a` |
| 상태 조회 커맨드 | `0aa501ff` |
| Keep Alive | **ON** |
| Keepalive 주기 | **5초** |

#### 주의사항

- **TV USB 전원**: 보통은 TV 꺼지면 USB 전원도 끊어져서 디바이스가 off되지만.. 제 경우는 위에서 말했듯이 TV를 절전모드로 살려놓고 있어서 USB에 항상 전원이 공급됩니다. 그래서 BLE로 제어할 필요가 있었던 거고요. 일반적으로는 TV가 켜진 상태에서만 BLE 제어 가능
- **Keepalive 5초**: 이 디바이스는 10초 주기에서도 idle disconnect 발생. 5초가 안정적
- **Write With Response OFF 필수**: ffb1 Characteristic의 Properties가 0x06(Read + Write Without Response). Write With Response 켜면 ACK 기다리다 hang

---

### 4. 마무리

대충 On/Off 스위치 하나만 돌려보고 싶었는데.. 새삼 뭔가 많이 배우게 됐네요.

특히 근데 물리 디바이스는 항상 피드백 돌리기가 힘들더라구요. 코드 수정하고 → HACS에서 업데이트하고 -> Hass 재시작하고 → BLE 연결 기다리고 → 테스트하고 → 로그 확인하고.. 한 사이클에 5분씩 날리니까 디버깅이 정말... 쉽지않았슴다.

연결 끊기고, ACK 안 오고, 같은 리소스에 동시 접근하면 에러 나고, 디바이스마다 idle timeout이 다르고..

이 프로젝트에서 배운 것들을 정리하면:

1. **BLE Write에는 반드시 타임아웃.** ACK 안 오면 Lock이 영원히 안 풀립니다.
2. **Notify 구독은 1회만, 연결 유지 동안 계속.** 매번 start/stop하면 race condition 온상.
3. **Keepalive는 체크만으론 부족.** 실제 데이터를 보내야 연결이 유지됩니다.

BLE Controller는 [HACS](https://hacs.xyz/)를 통해 설치할 수 있습니다. 전용 통합 없는 BLE 디바이스를 HA에서 제어하고 싶다면 [GitHub 레포](https://github.com/Lemon-HACS/ble_controller) 한번 확인해보세요!
