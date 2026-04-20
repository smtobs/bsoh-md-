# Agent `br-lan` IP 할당 흐름 분석

## 목적
- EasyMesh Agent가 **온보딩 이후 `br-lan`에 IP를 받는 실제 코드 경로**를 함수 단위로 정리한다.
- 특히 아래 질문에 답한다.
  - 어떤 이벤트를 받으면 IP 할당이 트리거되는가?
  - 누가 그 이벤트를 보내는가?
  - `dhcpc restart`와 `dhcpc renew`는 어떻게 다른가?

---

## 결론 요약
- Agent의 `br-lan` IP 갱신은 최종적으로 `cfghandler`의 wireless client 경로에서 `udhcpc` 제어로 발생한다.
- 온보딩 후 IP가 잡히는 주된 경로는 2가지다.

1. **재시작 경로 (Restart + Init + DHCPC restart)**
   - 온보딩/BH profile 반영 -> Agent 모드 전환/재기동
   - `wlanmon` init 재진입 시 `wireless_client_dhcpc_restart`
   - `udhcpc -i br-lan ...` 재시작

2. **연결 상태 경로 (Client connect + DHCPC renew)**
   - Agent 연결 완료 이벤트
   - `wireless_client_status_timer` -> `wireless_client_status`
   - 기존 `udhcpc`에 `SIGUSR1`(renew) 전송

---

## A. 온보딩 후 재시작 경로 (가장 명확한 `br-lan` 재할당)

## A-1) BH profile set 이벤트 수신
- 파일: `mcr-wlanmon/src/mtk/mtk_mesh.c`
- 함수: `wmon_mesh_agent_event()`
- 분기:
  - `msgType == MCR_EVENTID_EASY_MESH`
  - `reason == MCR_EVENT_REASON_MESH_WAPP_SET_BH_PROFILE`
- 동작:
  - `wmon_mesh_proc(..., MCR_MESH_EVT_BH_UPDATE, NULL)` 호출

> 이벤트 원천 관점: 이름상 `MESH_WAPP_*`는 wapp 계열 상태를 의미하며, wlanmon은 이벤트 프레임(`MCR_EVENTID_EASY_MESH`)으로 전달받아 처리한다.

## A-2) BH update apply 요청
- 파일: `mcr-wlanmon/src/mtk/mtk_mesh.c`
- 함수: `wmon_mesh_agent_bh_update()`
- 동작:
  - AP 모드였으면 `wireless_easy_mesh_bh_update_1`
  - Bridge/Agent 모드면 `wireless_easy_mesh_bh_update_0`
  - `requestCfgHandlerApplyMessage(..., "<key>")`로 cfghandler 호출

## A-3) cfghandler에서 모드 전환/재시작
- 파일: `mcr-cfghandler/src/cfg_vendor/server/mercury/mercury_wireless_easymesh.c`
- 함수: `mcr_mesh_bh_update()`
- 분기:
  - `wireless_easy_mesh_bh_update_1`이면 `mcr_applyWireless(..., "wireless_easy_mesh_agent_set_0")`
- 이어서 함수: `mcr_mesh_agent_set()`
  - 내부에서 최종 `mcr_applySysRestart(..., "restart_mesh")`

즉, 온보딩 시나리오에서 **재기동이 발생하면 wlanmon init이 다시 돈다**.

## A-4) 재기동 후 wlanmon init -> dhcpc restart
- 파일: `mcr-wlanmon/src/mtk/mtk_mesh.c`
- 함수:
  - `wmon_mesh_timer()` -> 최초 `bInit==0`이면 `MCR_MESH_EVT_INIT`
  - `wmon_mesh_proc(..., MCR_MESH_EVT_INIT, ...)`
- Agent/Repeater 조건에서:
  - `requestCfgHandlerApplyMessage(..., "wireless_client_dhcpc_restart")`

## A-5) cfghandler가 실제 `udhcpc` 재시작
- 파일: `mcr-cfghandler/src/cfg_vendor/server/mercury/mercury_wireless.c`
  - `mcr_applyWireless()`에서 `"wireless_client_dhcpc_restart"` 분기
- 파일: `mcr-cfghandler/src/cfg_vendor/server/mercury/mercury_wireless_client.c`
  - 함수: `mcr_applyWirelessClientLinkChange()`
  - 기존 dhcpc 종료: `mcr_disableNorlDaemon(DHCPC_Daemon, DHCPC_Pid)`
  - WAN DHCP면 실행:
    - `mcr_getWlanClientWanName(pHandler, 1, szIfName)`
    - `udhcpc -i <ifname> -p <pid> &`

## A-6) 왜 `<ifname>`이 `br-lan`인가
- 파일: `mcr-cfghandler/src/cfg_vendor/server/config_wireless.c`
- 함수: `mcr_getWlanClientWanName(...)`
- `bUseBridgeIf=1` + bridge 동작이면:
  - `MCR_NAME_BRG_IF` 반환 (실제 장비에서 통상 `br-lan`)

---

## B. 연결 완료 후 renew 경로 (재시작 없이 IP 갱신)

### B-0) 이 이벤트가 "언제" 발생하나 (핵심)

`MCR_MESH_EVT_CLIENT_CONNECT`는 아래 **두 단계를 모두 만족**할 때 발생한다.

1. **연결 이벤트 수신 시점**
   - `wmon_mesh_agent_event()`에서 `MCR_EVENTID_CONNECT` 수신
   - 이때 바로 `wmon_mesh_proc(...CLIENT_CONNECT...)`를 호출하지 않고
   - `wmon_mesh_agent_initConnect(..., 1, devName)`로 `bConnectEvent=1` 및 지연 타이머만 세팅

2. **지연 타이머 만료 시점**
   - `wmon_mesh_timer()`(1초 주기)에서 `bConnectEvent`를 감시
   - `nTimerExpire_Connect`가 0이 되면
   - 그때 `wmon_mesh_proc(..., MCR_MESH_EVT_CLIENT_CONNECT, NULL)` 실행

즉, **무선 링크가 CONNECT 된 "즉시"가 아니라, CONNECT 후 짧은 안정화 대기(`MCR_MESH_TIMEOUT_CLIENT_CONNECT`)가 지난 시점**에 renew 경로 이벤트가 발생한다.

## B-1) Client connect 이벤트
- 파일: `mcr-wlanmon/src/mtk/mtk_mesh.c`
- 함수: `wmon_mesh_proc()`
- 이벤트: `MCR_MESH_EVT_CLIENT_CONNECT`
- 동작:
  - `requestCfgHandlerApplyMessage(..., "wireless_client_status_timer")`

## B-2) cfghandler timer/status 처리
- 파일: `mcr-cfghandler/src/cfg_vendor/server/mercury/mercury_wireless.c`
  - `wireless_client_status_timer` -> 타이머 처리 후 `wireless_client_status`
- 파일: `mcr-cfghandler/src/cfg_vendor/server/mercury/mercury_wireless_client.c`
  - `mcr_applyWirelessClientLinkChange()`의 `wireless_client_status` 분기
  - link up 시 `udhcpc` PID에 `SIGUSR1` 전송 (renew)

이 경로는 프로세스를 재시작하지 않고 **기존 lease 갱신**이다.

---

## 이벤트 주체 정리

- `wlanmon -> cfghandler` 호출:
  - `requestCfgHandlerApplyMessage()` / `requestCfgHandlerSetMessage()`
  - 즉, wlanmon이 cfghandler에 apply/set을 보내는 구조

- 온보딩 관련 Mesh 상태 이벤트 원천:
  - 코드 이벤트 이름상 `MCR_EVENT_REASON_MESH_WAPP_*` (wapp 계열)
  - wlanmon은 이를 `MCR_EVENTID_EASY_MESH` 프레임으로 수신해 처리
  - 직접 함수 호출이 아니라 **이벤트 레이어 경유 수신**

### 추가 구분: WAPP 이벤트와 `CLIENT_CONNECT` 이벤트는 다르다

- **WAPP가 직접 보내는 것**
  - `MCR_EVENT_REASON_MESH_WAPP_WPS_START`
  - `MCR_EVENT_REASON_MESH_WAPP_WPS_OVERLAP/FAIL`
  - `MCR_EVENT_REASON_MESH_WAPP_SET_BH_PROFILE`
  - 위 reason들은 `wlanmon`의 `MCR_EVENTID_EASY_MESH` 분기에서 처리된다.

- **renew 경로를 직접 여는 이벤트**
  - `MCR_MESH_EVT_CLIENT_CONNECT`이며,
  - 이 이벤트는 `wmon_mesh_agent_event()`가 `MCR_EVENTID_CONNECT`를 수신했을 때
    `wmon_mesh_agent_initConnect(...,1,...)`를 세팅하고,
    `wmon_mesh_timer()` 만료 시점에 발생한다.

- **즉**
  - WAPP 이벤트는 온보딩/WPS/BH profile 상태를 전달하는 주체가 맞다.
  - 하지만 `wireless_client_status_timer -> udhcpc renew`의 **직접 트리거는 `MCR_EVENTID_CONNECT` 계열**이다.
  - WAPP 동작은 링크 연결을 유도해 CONNECT 발생에 **간접 기여**할 수 있다.

---

## 실무 디버깅 체크포인트

1. `MCR_EVENT_REASON_MESH_WAPP_SET_BH_PROFILE` 로그 존재 여부
2. `wireless_easy_mesh_bh_update_1` 실행 여부
3. `wireless_easy_mesh_agent_set_0` -> `restart_mesh` 발생 여부
4. 재기동 후 `MCR_MESH_EVT_INIT` 로그 여부
5. `wireless_client_dhcpc_restart` apply 실행 여부
6. 실제 프로세스:
   - `udhcpc -i br-lan ...` 실행 또는
   - `SIGUSR1` renew 발생 여부

---

## 최종 정리
- “온보딩 후에만 `br-lan` IP를 받는 것처럼 보이는” 이유는,
  - 온보딩 과정에서 BH profile 반영/모드 전환으로 재기동이 일어나고
  - 그 뒤 `wlanmon INIT -> wireless_client_dhcpc_restart`가 다시 실행되거나,
  - 또는 연결 완료 이벤트로 `udhcpc renew`가 발생하기 때문이다.

