# mcr-connmon: MTK 빌드 · 브리지 모드 · 유선 단말 정보

`mcr-connmon`은 IP/MAC 매핑을 `/tmp/connmon_v4.out` 등에 갱신하는 Mercury용 데몬입니다. **NAT 모드**에서는 DHCP lease·ARP 등을 병합하고, **브리지 모드(`-b`)**에서는 **드라이버(ioctl)가 제공하는 브리지 단말 목록**을 주 소스로 사용합니다.  
이 문서는 **`CONFIG_MTK`가 켜진 빌드**에서 브리지 모드일 때 **유선 단말**이 어떻게 걸러져 출력되는지 정리합니다.

---

## 1. 빌드·소스 구성

- 패키지: `mcr-connmon/`  
- 바이너리는 `src/Makefile` 기준 **`connMon_devices_mtk.c`가 항상 링크**됩니다(`connMon_devices_mtk.o`).  
- 칩 분기는 **`#if defined(CONFIG_MTK)`** (전처리기; 실제 정의는 빌드 시스템 `mcr_config.in` 등에서 `-DCONFIG_MTK` 형태로 주입되는 전제).

MTK 전용 ioctl 래퍼는 `mercury/libmcr-ioctl/mcr_ioctl.h` 쪽 API를 사용합니다(예: `McrIoctlGetBridgeMacList`, `McrIoctlGetPortByMacAddr`, `McrIoctlGetPortStatus`).

---

## 2. 브리지 모드 진입

| 항목 | 위치 | 내용 |
|------|------|------|
| 옵션 | `connMon.c` `cmon_ArgParse()` | `-b` / `--bridge` → `pInfo->bBridgeMode = 1` |
| 갱신 분기 | `connMon_devices.c` `cmon_refreshConnInfo()` | `bBridgeMode == 0` → `cmon_mergeFromDhcp` + `cmon_mergeFromArp` (NAT) <br> `bBridgeMode != 0` → **`cmon_mergeFromBridgeList`만** 호출 (브리지) |

브리지 모드에서는 DHCP/ARP 병합을 타지 않고, **브리지 ioctl 목록 한 줄기**로 리스트를 채웁니다.

---

## 3. 브리지 단말 목록: `cmon_mergeFromBridgeList()`

**파일**: `mcr-connmon/src/src/connMon_devices.c`

### 3.1 드라이버에서 목록 받기

1. `struct bridge_terminal_mng br_term_list[BRIDGE_TERM_MANAGE]` 초기화 (`BRIDGE_TERM_MANAGE == 64`, `connMon_devices_vendor.h`).
2. **`mcr_get_br_maclist(&(br_term_list[0]))`** 호출.

**MTK 구현** (`connMon_devices_mtk.c`, `#ifdef CONFIG_MTK`):

- **`McrIoctlGetBridgeMacList(pBridgeList)`**  
  - 주석: HSP 브리지 모드 단말 관리, 2025-09-11 커밋 메시지에 “AP Info → Client 페이지에서 데몬 크래시 보완” 언급.  
  - 커널/드라이버가 채운 배열에 대해 `ip != 0`인 슬롯만 상위에서 유효로 씀.

`struct bridge_terminal_mng` 필드:

- `ip`, `mac[6]`, `dummy[2]`, `ifname[16]` — **어느 논리 인터페이스(예: LAN1, wlan0) 뒤에 붙은 단말인지** `ifname`으로 구분하는 전제.

### 3.2 항목별 분류·필터

`br_term_list[i].ip != 0` 인 항목만 처리하며 `ConnMon_Item_t`로 merge합니다.

| 조건 | 동작 |
|------|------|
| **`CONFIG_MTK` + `ifname`이 `apcli`로 시작** | **스킵** (백홀/클라이언트 측 인터페이스로 보고 리스트에서 제외) |
| `wlan` / `wl` / `apcli` / `ra` 로 시작 | **무선**으로 간주. `CONFIG_MCR_WIRELESS`일 때 `cmon_wlanItem_find_fromMac`으로 **현재 STA 목록에 없으면 표시 안 함** |
| `USB`로 시작 (`CONFIG_MCR_DONGLE`) | USB 동글 등 `portType`에 ifname 복사 |
| **그 외 (유선으로 간주)** | **`is_wired_port_up_ifname(br_term_list[i].ifname)` 가 0이면 `continue`** → **링크 다운이면 유선 단말로 출력하지 않음** <br> 통과 시 `strcpy(item.portType, ifname)` (예: `LAN1`) |
| 공통 | `sourceType = 4` (코멘트: bridge only), `leaseTime = 0` |

**요지**: 브리지 모드 유선 단말은 **ioctl이 내려준 MAC/IP + ifname**을 쓰되, **유선은 LAN 포트 링크가 UP일 때만** 남깁니다.

---

## 4. MTK 유선 포트·MAC 조회 (`connMon_devices_mtk.c`)

브리지 merge에서 쓰는 것과, NAT 모드에서 쓰는 것이 나뉩니다.

### 4.1 포트 링크 UP 여부

- **`is_wired_port_up(int32 port)`**  
  - `McrIoctlGetPortStatus(port, &up, &speed, &duplex)`  
- **`is_wired_port_up_ifname(const char *pszIfname)`**  
  - `LAN` + 한 자리 숫자(`LAN1`~`LAN4` 가정)일 때만 위 API 호출. **그 외 ifname은 0(다운) 처리**.

### 4.2 MAC → 물리 포트 (NAT 경로에서 사용)

- **`McrIoctlGetPortByMacAddr`** (`McrIoctlPortByMacAddr_t`에 MAC 설정 후 호출)  
- **`wired_port_read_mac_cmp(host_mac)`**  
  - 포트가 1~4이고 **`is_wired_port_up(port)`** 이면 포트 번호 반환, 아니면 0.  
  - 포트 ≥7 등은 0으로 정리하는 분기 있음.  
- **`mcr_getPortByMac(...)`**  
  - 포트 1~(MAX_PORT-1)에서 링크 UP일 때 문자열로 포트 번호 반환; 포트 5 등 특수 케이스 `LEAF_SUCCESS`만 반환.

NAT 모드 `cmon_mergeFromDhcp`에서는 lease에 포트가 있어도 **`wired_port_read_mac_cmp(lease.mac)`** 로 다시 읽어 오고, **0이면 리스트에서 제외**합니다.

---

## 5. Easymesh와의 관계 (참고)

`CONFIG_MCR_EASY_MESH`가 켜지면 `mcr_refreshWlanMeshStaInfo()`가 **`/etc/mcr_mesh_set.sh dump_client`** 실행 후 **`/tmp/MESH_TOPOLOGY_INFO_CLIENT`** 를 파싱해, **Ethernet** 줄을 유선(mesh 하위) STA로 분류합니다. 이 경로는 **`cmon_refreshConnInfo` 초기에 wlan/mesh 캐시를 채우는 용도**이고, **브리지 모드에서는 여전히 `cmon_mergeFromBridgeList`만** 호출되므로 **브리지 유선 단말의 1차 소스는 `McrIoctlGetBridgeMacList`** 입니다.  
(Easymesh 분류는 NAT 쪽 DHCP/ARP 병합 시 `portNum == 6` 등 보조 판별에 쓰일 수 있음.)

---

## 6. 요약 흐름도 (브리지 + MTK + 유선)

```
connmon -b
  → cmon_refreshConnInfo()
       → cmon_mergeFromBridgeList()
            → McrIoctlGetBridgeMacList(br_term_list[])
            → 각 항목 (ip != 0):
                 apcli* (MTK) → skip
                 wlan*/wl*/ra*/apcli* → 무선 규칙 + STA 테이블 매칭
                 기타 → 유선으로 보고 is_wired_port_up_ifname(LANn) == 1 일 때만 merge
            → cmon_setItem → /tmp/connmon_v4.out 등 출력
```

---

## 7. 관련 파일

| 파일 | 역할 |
|------|------|
| `mcr-connmon/src/src/connMon.c` | `-b` 브리지 플래그 |
| `mcr-connmon/src/src/connMon_devices.c` | `cmon_refreshConnInfo`, `cmon_mergeFromBridgeList`, NAT 시 DHCP/ARP |
| `mcr-connmon/src/src/connMon_devices_mtk.c` | `mcr_get_br_maclist`, `is_wired_port_up*`, `wired_port_read_mac_cmp`, `mcr_getPortByMac` |
| `mcr-connmon/src/src/connMon_devices_vendor.h` | `bridge_terminal_mng`, `BRIDGE_TERM_MANAGE` |

커널 쪽 **`McrIoctlGetBridgeMacList` 구현체**는 이 워크스페이스의 `mcr-connmon` 밖(드라이버/`libmcr-ioctl`)에 있으므로, FDB/호스트 테이블과의 정확한 대응은 해당 모듈 스펙을 봐야 합니다.
