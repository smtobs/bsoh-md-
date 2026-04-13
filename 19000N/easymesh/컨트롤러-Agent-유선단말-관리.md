# Controller가 Agent 측 유선 단말 정보를 관리하는 방식

Controller는 Agent의 스위치 FDB에 **직접 접근하지 않습니다.** Agent의 **1905daemon**이 로컬에서 FDB를 읽어 **Non‑1905 Neighbor Device TLV**(IEEE 1905.1)에 실어 보내고, Controller 쪽 **1905daemon + mapd**가 Topology Response를 받아 파싱하면서 원격 단말 목록을 맞춥니다.

---

## 1. 전체 데이터 흐름

```
[Agent]
  스위치 FDB + 포트 상태
    → eth_data[].clients, non_p1905_neighbor_dev[]
    → Topology Response / (변경 시 Notification)
    → 1905 멀티캐스트·유니캐스트 L2

[Controller]
  1905daemon: Topology Response 수신 → TLV 파싱 → topology_response_db (devinfo, non_p1905_nbrdb_head 등)
  mapd: 1905 라이브러리 이벤트 → topo_srv_handle_topology_event()
    → NON_P1905 TLV 파싱 → struct _1905_map_device 의 wlan_clients + (선택) 유저 노티
```

**핵심 TLV**: `NON_P1905_NEIGHBOR_DEV_TLV_TYPE` — “이 인터페이스 MAC 뒤에 붙은 비‑1905 단말”의 EUI‑48 목록을 담습니다.

---

## 2. Agent: 유선 단말 → TLV로 실어 보내기

### 2.1 로컬 후보 목록

- `non_1905_neighbor_update()` / `non_1905_neighbor_update_per_itf()` (`1905daemon/src/topology.c`)  
  - `eth_layer_port_client_update()`로 FDB를 반영한 뒤, 포트별 `ethernet_client_entry` MAC을 순회합니다.  
  - 같은 포트에 1905 Topology Discovery로 본 단말이 있으면 non‑1905 이웃을 지우고, 없으면 `non_p1905_neighbor_info`를 `non_p1905nbr_head`에 넣습니다 (`itf_mac_addr`, `port_index`).

### 2.2 TLV 생성

- `append_non_p1905_neighbor_device_type_tlv()` (`1905daemon/src/cmdu_tlv.c`)  
  - `devlist[num].local_mac_addr` + `LIST_FOREACH(..., non_p1905nbr_head)` 의 `info->itf_mac_addr` 들을 값 필드에 나열합니다.  
  - `report_own_topology_rsp()` / `create_topology_response_message()` (`topology.c`, `cmdu_message.c`)에서 인터페이스마다 이 TLV를 붙입니다.

즉 **Agent가 “유선으로 보이는 비‑1905 MAC”을 스위치 테이블 기준으로 모아 TLV에 넣는 주체**입니다.

---

## 3. Controller 1905daemon: 수신 측 파싱 (로컬 DB)

Topology Response 처리 루프에서 Non‑1905 TLV를 만나면:

- `parse_non_p1905_neighbor_device_type_tlv(&devinfo_list, len, value)` (`1905daemon/src/cmdu_tlv_parse.c`)  
  - `local_mac`에 해당하는 `device_info_db`를 찾고, 각 `neighbor_mac`에 대해 `non_p1905_neighbor_device_list_db`를 `dev_info->non_p1905_nbrdb_head`에 연결합니다.

이 경로는 **1905daemon 내부 topology response DB**를 채우는 쪽이며, MAP 상위 정책은 주로 mapd의 `topo_srv`와 연동됩니다.

---

## 4. Controller mapd: 토폴로지 이벤트와 저장 구조

### 4.1 이벤트 진입

- `mapd/src/1905_if/1905_if.c`  
  - `_1905_RECV_TOPOLOGY_RSP_EVENT` → `topo_srv_handle_topology_event(global, buf, length)`  

Topology Notification(`_1905_RECV_TOPOLOGY_NOTIFICATION_EVENT`)은 연결 변경·메트릭 등 다른 처리를 하며, **에이전트 유선 MAC 전체 목록을 다시 그리는 본 파서는 Topology Response 쪽**이 중심입니다(망에서 주기/요청에 따라 RSP가 갱신됨).

### 4.2 파싱 전: 기존 클라이언트 무효화

- `topo_srv_handle_topology_event()` (`mapd/src/topologySrv/topologySrv.c`)  
  - 원격 장치 `struct _1905_map_device *dev`에 대해 **`clear_all_client_info(ctx, dev)`** 호출 (`tlv_parsor.c`).  
  - `dev->wlan_clients`를 순회하며 `topo_srv_get_bss_by_bssid()`로 **무선 BSS에 매핑되는 항목만** 살리고, **이더넷 인터페이스 쪽(대응 BSS 없음)** 은 `entry_valid = FALSE`로 둡니다.

### 4.3 NON_P1905 TLV 파싱

- `parse_non_p1905_neighbor_device_type_tlv(ctx, temp_buf, dev)` (`mapd/src/topologySrv/tlv_parsor.c`)  
  - TLV의 `local_mac`이 `dev->_1905_info.first_iface` 중 하나와 일치하는지 확인.  
  - 각 `neighbor_mac`에 대해 `dev->wlan_clients`에 같은 STA MAC이 없으면 `connected_clients`를 할당해 넣고:  
    - `client_addr` = 단말 MAC  
    - `_1905_iface_addr` = Agent의 **해당 이더넷 인터페이스 MAC**  
    - `is_bh_link = 0`, `is_APCLI = 0`  
  - **이더넷 매체**인 경우(`iface->media_type < IEEE802_11_GROUP`):  
    - `mapd_user_eth_client_join(global, client->client_addr, dev->_1905_info.al_mac_addr)`  
    - 유저/앱 쪽으로 `ETHERNET_CLIENT_JOIN_NOTIF` 전송 (`mapd_ctrl_iface_send`).

### 4.4 파싱 후: 사라진 유선 단말 정리

- 같은 `topo_srv_handle_topology_event()` 마지막 부근에서 **`clear_invalid_client_info(ctx, dev)`**  
  - `wlan_clients` 중 `topo_srv_get_bss_by_bssid`가 NULL이고 **`entry_valid == FALSE`** 인 항목을 제거하면서  
  - `mapd_user_eth_client_leave(global, sta_mac, al_mac)` → **`ETHERNET_CLIENT_LEAVE_NOTIF`**.

정리하면 Controller mapd는 **원격 Agent를 나타내는 `struct _1905_map_device`의 `wlan_clients` 리스트**에 유선 단말을 “연결 클라이언트” 형태로도 쌓고(이름은 역사적으로 무선과 공유), **이더넷 매체일 때만** join/leave 유저 이벤트를 쏩니다.

---

## 5. 역할 정리 표

| 주체 | 하는 일 |
|------|---------|
| **Agent 1905daemon** | FDB·포트 → `non_p1905_neighbor_dev` → Non‑1905 TLV 송신 |
| **Controller 1905daemon** | Topology RSP TLV 파싱 → `non_p1905_nbrdb_head` 등 로컬 1905 DB |
| **Controller mapd** | `_1905_RECV_TOPOLOGY_RSP_EVENT` → `topo_srv_handle_topology_event` → `dev->wlan_clients` + `mapd_user_eth_client_{join,leave}` |
| **Controller wappd** | 이 경로에서는 **유선 단말을 직접 보지 않음** (무선 assoc과 무관) |

---

## 6. 지연·일관성

- Agent 쪽 FDB 반영은 `SWITCH_TABLE_DUMP_TIME`(예: 45초) 주기와 포트 이벤트에 묶입니다.  
- Controller view는 **마지막으로 수신·파싱한 Topology Response** 기준이므로, **실시간 스위치 상태와의 lag**가 생길 수 있습니다.

---

## 7. 참고 파일 목록

| 파일 | 설명 |
|------|------|
| `1905daemon/src/topology.c` | `non_1905_neighbor_update*`, `report_own_topology_rsp` |
| `1905daemon/src/cmdu_tlv.c` | `append_non_p1905_neighbor_device_type_tlv` |
| `1905daemon/src/cmdu_tlv_parse.c` | `parse_non_p1905_neighbor_device_type_tlv` (1905daemon DB) |
| `1905daemon/src/cmdu_message_parse.c` | Topology RSP TLV 디스패치 |
| `mapd/src/1905_if/1905_if.c` | `_1905_RECV_TOPOLOGY_RSP_EVENT` |
| `mapd/src/topologySrv/topologySrv.c` | `topo_srv_handle_topology_event` |
| `mapd/src/topologySrv/tlv_parsor.c` | `parse_non_p1905_neighbor_device_type_tlv`, `clear_all_client_info`, `clear_invalid_client_info`, `mapd_user_eth_client_join/leave` |
