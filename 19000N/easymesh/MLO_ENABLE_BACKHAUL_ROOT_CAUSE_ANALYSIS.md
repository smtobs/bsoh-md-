# Backhaul 경로 `cli->mlo_enable` 누락 원인 분석

## 배경
- 현상: Agent backhaul client(2.4G/5G/6G link MAC) 연결 시 Controller `mapd`에서 `cli->mlo_enable`이 1로 세팅되지 않는 경우가 자주 발생.
- 결과: 이후 MLD 연계 정보가 지워지거나(`clear mld address`), BSS 매핑 실패(`sta mac not found`)가 연쇄적으로 발생.

## 로그 기반 관찰
- `parse_assoc_req` 경로에서 다음 패턴 확인:
  - `current operating class = 137` (6GHz 맥락)
  - `client not mlo enabled clear mld address`
  - `find_bss_from_dev_by_client_link_id`: `sta mac not found`
  - `find_bss_from_dev_by_current_operating_class`: `sta mac not found`
- 동시에 `topo_srv_update_assoc_client_info`에서는 `sta/bss` 업데이트가 수행되어, 완전 미수신이 아니라 "MLO 판정 경로 일부 누락" 가능성이 높음.

## 코드 경로 분석 결과

### 1) `parse_assoc_req()` 호출 게이트가 존재함 (핵심)
파일: `mapd/src/dot11_if/wapp_if.c`

- `MAP_R6`에서 아래 조건일 때만 `parse_assoc_req()` 호출:
  - `pevt->is_setup_link_entry == 1` **또는**
  - `is_zero_ether_addr(pevt->ap_mld_addr)`  
- 즉, **non-setup link join** (`is_setup_link_entry==0` && `ap_mld_addr!=0`)에서는 `parse_assoc_req()`가 호출되지 않음.

이 경우 Assoc Req IE 파싱 기반의 MLO 판정이 동작하지 않아 `cli->mlo_enable` 세팅 기회를 잃는다.

### 2) `cli->mlo_enable = 1` 세팅 경로가 Assoc Req ML IE 파싱에 과도 의존
파일: `mapd/src/topologySrv/topologySrv.c`

- `parse_assoc_req()` 내부 `IE_WLAN_EXTENSION` + `EID_EXT_MULTI_LINK(107)`에서만 `cli->mlo_enable = 1` 설정.
- 이 IE가 누락/잘림/비호환 형식이면 `mlo_enable`은 0 유지.

### 3) 세팅 실패 시 즉시 MLD 주소를 클리어함
파일: `mapd/src/topologySrv/topologySrv.c`

- `if (!cli->mlo_enable) memset(cli->mld_addr, 0, ETH_ALEN);`
- 위 1), 2)로 인해 일시적으로라도 `mlo_enable`이 0이면 이후 MLD 기반 매핑 로직이 불리해짐.

### 4) STA 식별자 축 불일치 가능성 (link MAC vs MLD MAC)
- 경로마다 `sta_mac`, `sta_mld_addr`, 802.11 `Addr2`를 혼용.
- `ap_sta_map_head` 조회 시 기대하는 MAC 축과 저장된 MAC 축이 다르면 `sta mac not found` 발생 가능.

### 5) MBO NPC 빈 버퍼 에러는 부수 신호
- `mapd_mbo_parse_sta_npc_element`: `len == 0`일 때 `Empty buffer received error`.
- 이는 MLO 세팅 실패의 직접 원인이라기보다, Assoc IE 파싱 품질/입력 품질 이슈의 동반 신호로 판단됨.

## "왜 자주" 발생하는지에 대한 근본 요약
1. **구조적으로** non-setup link join에서 `parse_assoc_req()`를 건너뛰는 설계.
2. `mlo_enable` 세팅은 사실상 ML IE 단일 파서 경로에 집중.
3. 실패 시 `mld_addr` 즉시 클리어로 후속 복구 여지 축소.
4. 이벤트 간 MAC 식별 축 불일치(link/MLD)로 매칭 실패가 누적.

## 영향
- Controller에서 backhaul client를 "MLO 비활성 STA"로 잘못 취급.
- 1905/WAPP 연동 시 setup/non-setup link 통합 처리 실패 가능성 증가.
- Steering, band/opclass 기반 BSS 매핑, MLD group 연동 정확도 저하.

## 단기 보강 아이디어 (분석 기반)
- WAPP 이벤트(`sta_mld_addr`, `ap_mld_addr`)를 fallback 근거로 `mlo_enable` 보강.
- `mld_addr` 클리어를 즉시 수행하지 않고, 보강 경로 확인 후 지연/조건부 처리.
- MAC 키 정규화(동일 STA를 link MAC/MLD MAC 관점에서 단일 엔트리로 수렴).

## 이벤트 전달 경로 확정 (driver direct 여부)

### 결론
- Backhaul/STA association 이벤트는 **driver -> wappd -> mapd** 경로로 전달된다.
- 즉, mapd가 driver 이벤트를 **direct**로 받는 구조가 아니라, wappd의 이벤트 래핑/전달을 거친다.

### 근거 코드 (chain)
1. **driver (`mt_wifi7`)에서 WAPP 이벤트 생성**
   - 파일: `mt_wifi7/mt_wifi/feature/wapp/wapp.c`
   - `wapp_send_cli_join_event()`에서 `event.event_id = WAPP_CLI_JOIN_EVENT;`
   - `wapp_fill_client_info_new()`로 `cli_info->mlo.*`를 채운 뒤 `wapp_send_wapp_qry_rsp()` 호출.

2. **wappd에서 map 이벤트로 변환 후 송신**
   - 파일: `wappd/common/wdev.c`
   - `map_send_assoc_cli_msg(..., cli->mlo.sta_mld_addr, cli->mlo.ap_mld_addr, ..., cli->mlo.is_setup_link_entry)` 호출로 MLO 필드 포함 전달.
   - 파일: `wappd/map/map_1905.c`
   - `map_send_assoc_cli_msg()` -> `map_build_assoc_cli()` -> `map_1905_send()`로 실제 전송 수행.

3. **mapd에서 WAPP_CLIENT_NOTIFICATION 수신/처리**
   - 파일: `mapd/src/dot11_if/wapp_if.c`
   - `wlanif_handle_client_notification()`에서 `WAPP_CLIENT_NOTIFICATION` payload를 파싱하고,
     `topo_srv_parse_wapp_client_notification()` / `parse_assoc_req()` 경로로 진입.

### 현재 이슈와의 연결
- 로그에서 `wapp_usr_intf_parse_event ... WAPP_CLIENT_NOTIFICATION`가 먼저 보이고, 이어 mapd `parse_assoc_req`가 동작하므로,
  실제 현상 분석 시 "driver direct 누락"보다 **wappd를 거친 payload 내용(길이/필드/식별자 축)과 mapd 파서 조건**을 우선 검증해야 한다.

## 참고 파일
- `mapd/src/dot11_if/wapp_if.c`
- `mapd/src/topologySrv/topologySrv.c`
- `mapd/src/topologySrv/wappEvent.c`
- `mapd/src/topologySrv/tlv_parsor.c`
