# Driver/Daemon 이벤트 상호작용 다이어그램 (함수명 포함)

## 목적
- `mt_wifi7` 드라이버, `wappd`, `wlanmon`, `cfghandler` 사이의 이벤트 송수신을
  **함수명 기준**으로 한눈에 확인한다.

---

## 1) Station 연결 시 `CONNECT -> renew` 경로

```mermaid
sequenceDiagram
    participant STA_FSM as mt_wifi7 STA FSM<br/>sta_mgmt_cntl.c
    participant DRV_EVT as mt_wifi7 Event Builder<br/>ap_mcr_util.c
    participant KERNEL as cfg80211/vendor event
    participant WMON_ING as wlanmon ingest<br/>wlanmon_mtk.c
    participant MESH as wlanmon mesh<br/>mtk_mesh.c
    participant CFG as cfghandler<br/>mercury_wireless*.c
    participant DHCP as udhcpc

    STA_FSM->>STA_FSM: sta_cntl_assoc_conf()<br/>Reason == MLME_SUCCESS
    STA_FSM->>STA_FSM: LinkUp(...)
    STA_FSM->>DRV_EVT: mcr_wl_user_event_client_simple(...,<br/>MCR_EVENTID_CONNECT,...)
    DRV_EVT->>DRV_EVT: mcr_wl_user_event()<br/>mcr_wl_event_set_Header()
    DRV_EVT->>KERNEL: cfg80211_mcr_vendor_event(...,<br/>MCR_NL80211_MCR_EVENT_IND)

    KERNEL->>WMON_ING: wireless_event_process(...)
    WMON_ING->>MESH: pFunc->fpTaskEvent(...,<br/>SubEventId=MCR_EVENTID_CONNECT,...)

    MESH->>MESH: wmon_mesh_event() -> wmon_mesh_agent_event()
    MESH->>MESH: wmon_mesh_agent_initConnect(...,1,...)<br/>bConnectEvent=1
    MESH->>MESH: wmon_mesh_timer() tick (1s)
    MESH->>MESH: nTimerExpire_Connect==0 -><br/>wmon_mesh_proc(MCR_MESH_EVT_CLIENT_CONNECT)
    MESH->>CFG: requestCfgHandlerApplyMessage(...,<br/>"wireless_client_status_timer")

    CFG->>CFG: mcr_applyWireless("wireless_client_status_timer")
    CFG->>CFG: mcr_applyWirelessClientLinkChange("wireless_client_status")
    CFG->>DHCP: kill(udhcpc_pid, SIGUSR1)  %% renew
```

---

## 2) WAPP EasyMesh 이벤트 경로 (`MCR_EVENTID_EASY_MESH`)

```mermaid
sequenceDiagram
    participant WAPP as wappd
    participant CLI as wcli usercmd
    participant DRV_EVT as mt_wifi7 Event Builder<br/>ap_mcr_util.c
    participant KERNEL as cfg80211/vendor event
    participant WMON_ING as wlanmon ingest<br/>wlanmon_mtk.c
    participant MESH as wlanmon mesh<br/>mtk_mesh.c
    participant CFG as cfghandler

    WAPP->>CLI: system("wcli -i ra0 usercmd 1 25 X")
    Note over WAPP,CLI: 예) X=0 WPS_START, X=3 WPS_OVERLAP, X=6 SET_BH_PROFILE
    CLI->>DRV_EVT: mcr_wl_user_event_userCmd(...,<br/>MCR_EVENTID_EASY_MESH, reason=X)
    DRV_EVT->>KERNEL: cfg80211_mcr_vendor_event(..., 25, ...)

    KERNEL->>WMON_ING: wireless_event_process(...)
    WMON_ING->>MESH: pFunc->fpTaskEvent(...,<br/>SubEventId=MCR_EVENTID_EASY_MESH,...)
    MESH->>MESH: wmon_mesh_agent_event()

    alt reason == MCR_EVENT_REASON_MESH_WAPP_WPS_START
        MESH->>MESH: wmon_mesh_proc(MCR_MESH_EVT_WPS_START)
    else reason == MCR_EVENT_REASON_MESH_WAPP_SET_BH_PROFILE
        MESH->>MESH: wmon_mesh_proc(MCR_MESH_EVT_BH_UPDATE)
        MESH->>CFG: requestCfgHandlerApplyMessage(...,<br/>"wireless_easy_mesh_bh_update_0/1")
    else reason == MCR_EVENT_REASON_MESH_WAPP_WPS_OVERLAP/FAIL
        MESH->>MESH: wmon_mesh_proc(MCR_MESH_EVT_WPS_FAIL)
    end
```

---

## 3) `br-lan` IP 재할당(재시작 기반) 경로

```mermaid
sequenceDiagram
    participant MESH as wlanmon mesh<br/>mtk_mesh.c
    participant CFG as cfghandler<br/>mercury_wireless_easymesh.c
    participant REBOOT as system restart
    participant WMON as wlanmon restart/init
    participant CFG2 as cfghandler<br/>mercury_wireless_client.c
    participant DHCP as udhcpc

    MESH->>CFG: requestCfgHandlerApplyMessage(...,<br/>"wireless_easy_mesh_bh_update_1")
    CFG->>CFG: mcr_mesh_bh_update()
    CFG->>CFG: mcr_applyWireless("wireless_easy_mesh_agent_set_0")
    CFG->>CFG: mcr_mesh_agent_set()
    CFG->>REBOOT: mcr_applySysRestart(...,"restart_mesh")

    REBOOT->>WMON: wlanmon 재시작
    WMON->>WMON: wmon_mesh_timer() 초기 진입<br/>bInit==0
    WMON->>WMON: wmon_mesh_proc(MCR_MESH_EVT_INIT)
    WMON->>CFG2: requestCfgHandlerApplyMessage(...,<br/>"wireless_client_dhcpc_restart")
    CFG2->>CFG2: mcr_applyWirelessClientLinkChange("wireless_client_dhcpc_restart")
    CFG2->>DHCP: udhcpc -i br-lan -p /var/run/udhcpc.pid &
```

---

## 4) 핵심 포인트
- `MCR_EVENTID_CONNECT` 직접 발생 주체: **드라이버 STA FSM(LinkUp 경로)** 또는 supplicant ctrl 변환 경로.
- `MCR_EVENTID_EASY_MESH` reason(`MESH_WAPP_*`) 주체: **wappd usercmd 경로**.
- WAPP 이벤트는 온보딩/WPS/BH 상태를 알리고, CONNECT는 실제 링크 업 이벤트를 반영한다.

