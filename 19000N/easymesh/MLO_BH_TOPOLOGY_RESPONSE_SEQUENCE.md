# MLO Backhaul Topology Response 시퀀스

컨트롤러가 Topology Response(0x0003)를 보낼 때, MLO backhaul SSID에 연결된 링크 STA(d6/d7/d8)가 E2 TLV에 실리는 과정을 데몬/함수 단위로 정리한다.

## End-to-End Sequence

```mermaid
sequenceDiagram
    autonumber
    participant DRV as Driver/FW
    participant WAPPD as wappd
    participant MAPD as mapd
    participant LIB1905 as map_1905 lib (IPC)
    participant D1905 as 1905daemon
    participant CTRL as Controller Peer(요청자)

    Note over DRV,WAPPD: [1] BH MLO 링크별 연결 이벤트 발생 (d6/d7/d8)
    DRV->>WAPPD: WAPP_CLIENT_NOTIFICATION (sta,bssid,setup_link,sta_mld,ap_mld,assoc_req,...)

    Note over WAPPD,MAPD: [2] wappd가 이벤트를 mapd로 전달
    WAPPD->>MAPD: map_build_assoc_cli / WAPP_CLIENT_NOTIFICATION

    Note over MAPD: [3] mapd 수신 처리
    MAPD->>MAPD: wlanif_handle_client_notification()
    MAPD->>MAPD: parse_assoc_req()  %% assoc_req IE/ML IE 파싱 (조건부)
    MAPD->>MAPD: topo_srv_parse_wapp_client_notification()  %% STA notification 경로

    alt setup_link == 1
        Note over MAPD: [4-A] setup 링크에서만 MLD Config 생성 경로 진입
        MAPD->>MAPD: topo_srv_parse_wapp_sta_mld_configuration()
        MAPD->>MAPD: cli->mld_sta_config 복사
        MAPD->>MAPD: non_setup_info[].bssid -> mlo_sta_info[n+1].bssid 반영
        MAPD->>MAPD: mcr_topo_srv_mld_merge_aff_sta_setup_only() %% mcr patch
        Note over MAPD: 조건: setup-link 이벤트 && same MLD && same BSSID && 빈 affiliate 슬롯만 보강
        MAPD->>LIB1905: map_1905_Set_Sta_Mld_Configuration(mlo_sta_config_evt)
    else setup_link != 1
        Note over MAPD: [4-B] 현재 정책상 MLD Config 전송 경로 미진입
    end

    Note over CTRL,D1905: [5] Controller peer가 Topology Query(0x0002) 송신
    CTRL->>D1905: Topology Query CMDU

    Note over D1905: [6] 1905daemon이 Topology Response 구성
    D1905->>D1905: report_own_topology_rsp()/create_topology_response()
    D1905->>D1905: append_assoc_sta_mld_cfg_rpt_tlv()  %% E2 TLV serialize
    Note over D1905: update_sta_mld_cfg_rpt DB 기반으로<br/>sta_mld/ap_mld/affiliate(bssid,aff_sta) 직렬화

    D1905->>CTRL: Topology Response(0x0003) + E2 TLV
```

## 단계별 핵심 함수

- `wappd`
  - `map_build_assoc_cli` (mapd로 client notification 전달)
- `mapd`
  - `wlanif_handle_client_notification`
  - `parse_assoc_req` (`wapp_if.c`에서 호출, 구현은 `topologySrv.c`)
  - `topo_srv_parse_wapp_client_notification`
  - `topo_srv_parse_wapp_sta_mld_configuration` (setup-link 기준)
  - `mcr_topo_srv_mld_merge_aff_sta_setup_only` (setup-link 표 보강)
  - `map_1905_Set_Sta_Mld_Configuration` (1905 lib 전달)
- `1905daemon`
  - `update_sta_mld_cfg_rpt` (내부 DB 갱신 경로)
  - `report_own_topology_rsp` / `create_topology_response`
  - `append_assoc_sta_mld_cfg_rpt_tlv` (E2 직렬화)

## 로그 대조 포인트

- `mapd`
  - `MCR BSOH DBG5: wapp_if client notif ...`
  - `MCR BSOH DBG5: wappEvent parse notif ...`
  - `MCR BSOH DBG7: topo_srv_parse_wapp_sta_mld ->1905 ...`
  - `MCR BSOH DBG7: merge_setup_only ...` (mcr patch 동작 시)
- `1905daemon`
  - `LIB_STA_MLD_CONFIG_REPORT ...`
  - `append_assoc_sta_mld_cfg_rpt_tlv ...`

## 정리

- E2(Associated STA MLD Config)는 최종적으로 1905daemon이 serialize하지만, 소스 데이터는 mapd가 `map_1905_Set_Sta_Mld_Configuration`으로 넘긴 값이다.
- 따라서 `wappd -> mapd 구성 -> 1905daemon serialize` 3단계를 시간축으로 맞춰 보는 것이 필수다.
