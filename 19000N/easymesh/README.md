# Easymesh 문서 (19000N)

이 디렉터리는 MTK Easymesh 스택에서 **유·무선 단말 연결/이탈**, **Agent 온보딩 후 `br-lan` IP 할당**, **Driver↔Daemon 이벤트 전달**을 정리한 문서를 둡니다.

| 문서 | 내용 |
|------|------|
| [유무선-단말-연결-이벤트.md](./유무선-단말-연결-이벤트.md) | `wappd` → `mapd` → `1905daemon` 무선 STA 흐름, `1905daemon` 유선(스위치 FDB) 흐름, 주요 변수·리스트 |
| [컨트롤러-Agent-유선단말-관리.md](./컨트롤러-Agent-유선단말-관리.md) | Controller가 Agent 유선 단말을 **직접 보지 않고** Non‑1905 TLV·Topology Response로 수집하는 경로, `mapd` 저장·유저 노티 |
| [AGENT_BRLAN_IP_ASSIGN_FLOW.md](./AGENT_BRLAN_IP_ASSIGN_FLOW.md) | Agent 온보딩 이후 `br-lan` IP 갱신 경로 정리 (재시작 경로 vs renew 경로) |
| [AGENT_BRLAN_IP_ASSIGN_SEQUENCE_DIAGRAM.md](./AGENT_BRLAN_IP_ASSIGN_SEQUENCE_DIAGRAM.md) | Driver/`wappd`/`wlanmon`/`cfghandler` 간 이벤트 송수신을 함수명 포함 Mermaid 시퀀스로 정리 |

### 추천 열람 순서
1. `AGENT_BRLAN_IP_ASSIGN_FLOW.md`
2. `AGENT_BRLAN_IP_ASSIGN_SEQUENCE_DIAGRAM.md`
3. `유무선-단말-연결-이벤트.md`
4. `컨트롤러-Agent-유선단말-관리.md`

분석 대상 소스 트리(워크스페이스 기준): `mt_wifi7/`, `mcr-wlanmon/`, `mcr-cfghandler/`, `wappd/`, `mapd/`, `1905daemon/`.
