# mapd `dump.txt` 생성 및 Backhaul Address 분석 (Controller 관점, MLO 포함)

## 1) 목적

이 문서는 `mapd`에서 `/tmp/dump.txt`가 생성되는 경로와, Controller 관점에서 Backhaul 주소가 어떤 구조/로직으로 수집되는지 정리한다.  
특히 MLO 환경에서 setup link와 non-setup link 주소가 어떻게 유지되는지에 초점을 둔다.

---

## 2) `/tmp/dump.txt` 생성 경로

`dump.txt`는 `mapd_cli dump_topology_v1` 명령 체인에서 생성된다.

1. `mapd_cli dump_topology_v1` 실행
2. `mapd_interface_get_topology()` 호출
3. 기본 파일 경로를 `/tmp/dump.txt`로 설정 (`file_path_ == NULL`일 때)
4. mapd ctrl iface로 `dump_topology_v1 <index> <file_path>` 명령 전송
5. daemon 측 `ctrl_iface.c`에서 해당 명령 처리 후 파일 open/write
6. `topo_srv_dump_topology_v1()`가 만든 topology 문자열을 `fprintf()`로 파일 기록

핵심 포인트:
- 파일 생성 주체는 CLI가 아니라 **mapd daemon(ctrl iface)** 이다.
- 기본 dump 파일은 `/tmp/dump.txt`이며, 인자를 주면 다른 경로도 가능하다.

---

## 3) Backhaul 주소의 내부 저장 구조

Controller가 관리하는 topology DB에서 backhaul 링크는 neighbor 단위로 유지된다.

- neighbor 엔트리: `map_neighbor_info`
- backhaul 링크 리스트: `neighbor->bh_head`
- 각 링크 엔트리: `backhaul_link_info`

핵심 주소 필드:
- `connected_iface_addr`: 로컬 쪽 연결 인터페이스 주소(실무적으로 setup link APCLI MAC 성격)
- `neighbor_iface_addr`: 상대측 backhaul 인터페이스 주소(실무적으로 BH BSSID 성격)

즉, Controller가 보는 backhaul 주소 쌍은 기본적으로 다음 쌍이다.
- `connected_iface_addr` <-> `neighbor_iface_addr`

---

## 4) Backhaul 주소가 채워지는 시점

### 4.1 초기 neighbor/bh 생성 시

`topo_srv_create_insert_neighbor_entry()`에서 새 `backhaul_link_info`를 할당하고:
- `own_iface`가 있으면 `connected_iface_addr` 설정
- `n_iface`가 있으면 `neighbor_iface_addr` 설정

### 4.2 토폴로지 업데이트 시

기존 엔트리가 있을 때:
- `topo_srv_update_neighbor_entry()`에서 `connected_iface_addr` 갱신 가능
- `topo_srv_update_neighbor_entry_peer()`에서 `neighbor_iface_addr` 갱신 가능

매체 타입(무선/유선), skip 조건, 인증/연동 상황에 따라 어느 쪽 주소를 우선 확정할지 분기한다.

### 4.3 링크 메트릭 TLV 반영 시

`insert_new_tx_link_metrics_info()` / `insert_new_rx_link_metrics_info()`에서
수신한 TLV의 링크 식별값으로 기존 `bh_head` 엔트리를 찾아:
- 주소 매칭
- `connected_iface_addr`, `neighbor_iface_addr`, tx/rx metric 갱신

즉, 초기 생성 후에도 runtime metric 수신으로 주소/상태가 보정된다.

---

## 5) Controller 관점에서 dump 출력 시 주의점

`topo_srv_dump_topology_v1()` 내부의 `topo_srv_dump_1905_dev_info_v1()`는 `"BH Info"`를 구성하지만,
BH 항목 덤프 시 아래 조건이 있다.

- `dev->device_role == CONTROLLER/CONTRAGENT` 인 디바이스는 `"BH Info"` 루프에서 skip

따라서 `dump.txt`의 BH 정보는:
- 컨트롤러 자신의 BH가 아니라,
- **네트워크 내 Agent 디바이스 엔트리 기준 BH 정보**가 주로 기록된다.

---

## 6) MLO Backhaul 주소 처리

`MTK_MLO_MAP_SUPPORT` 빌드에서는 setup link 외에 non-setup link metric이 분리 저장된다.

- TX 측: `bh_info->mlo_tx[i].connected_iface_addr`
- RX 측: `bh_info->mlo_rx[i].connected_iface_addr`

`insert_new_tx_link_metrics_info()` / `insert_new_rx_link_metrics_info()`에서
non-setup link로 판단되면 `mlo_tx/mlo_rx` 배열에 주소/metric을 채우고 `mlo_link_no`를 업데이트한다.

정리:
- setup link 주소: `connected_iface_addr`, `neighbor_iface_addr`
- MLO non-setup 주소: `mlo_tx[*].connected_iface_addr`, `mlo_rx[*].connected_iface_addr`

---

## 7) `BH_INFO` CLI와 주소 의미 매핑

`mapd_cli bh_info` 출력 경로(`topo_srv_dump_bh_all_info()` 계열)에서
MLO 케이스는 다음 라벨로 직접 확인 가능하다.

- `Backhaul BSSID` -> `neighbor_iface_addr`
- `Setup Link APCLI-MAC` -> `connected_iface_addr`

운영 시 확인 포인트:
- AL-MAC별로 위 2개 주소가 기대한 Agent uplink와 일치하는지
- MLO 환경이면 non-setup link metric 항목이 같이 노출되는지

---

## 8) 실무 디버깅 체크리스트

1. `mapd_cli /tmp/mapd_ctrl dump_topology_v1` 실행 후 `/tmp/dump.txt` 생성/갱신 확인
2. `mapd_cli /tmp/mapd_ctrl bh_info`로 AL-MAC 단위 BH 주소 매핑 확인
3. 특정 Agent에 대해:
   - `connected_iface_addr` (setup APCLI MAC)
   - `neighbor_iface_addr` (BH BSSID)
   - MLO인 경우 `mlo_tx/mlo_rx` 주소들
4. 주소 mismatch 시:
   - neighbor 생성 경로 (`topo_srv_create_insert_neighbor_entry`)
   - update 경로 (`topo_srv_update_neighbor_entry*`)
   - metric 삽입 경로 (`insert_new_tx/rx_link_metrics_info`)
   순서로 추적

---

## 9) 결론

- `/tmp/dump.txt`는 mapd daemon의 `dump_topology_v1` 처리에서 생성된다.
- Controller 관점 backhaul 주소의 원천은 `neighbor->bh_head`의 `backhaul_link_info` 구조다.
- MLO에서는 setup link와 non-setup link 주소가 별도 필드로 유지되며, 링크 메트릭 TLV 수신으로 지속 보정된다.
