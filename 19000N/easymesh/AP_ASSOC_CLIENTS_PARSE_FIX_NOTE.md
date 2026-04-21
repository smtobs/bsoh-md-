# AP_ASSOCIATED_CLIENTS 파싱 보정 작업 기록

## 1) 배경

Controller 관점 `dump_topology_v1`/`dump.txt` 확인 시,
- MLO backhaul SSID 연관 클라이언트에서 `6GHz`는 정상이나
- `2.4GHz`와 `5GHz`의 MAC 매핑이 높은 비율로 서로 뒤바뀌는 현상이 관찰됨.

재현 패턴상 랜덤 데이터 손상보다는 TLV 파싱 오프셋 불일치 가능성이 높다고 판단함.

---

## 2) 원인 분석 요약

대상 함수:
- `mapd/src/topologySrv/tlv_parsor.c`
- `parse_ap_associated_clients_type_tlv(...)`

핵심 이슈:
- BSS 필터 조건(`check_bssid_oper_bss_list`)에서 skip되는 BSS가 있을 때,
- 기존 코드가 해당 BSS의 STA payload를 소비하지 않고 `continue`함.
- 그 결과 `temp_buf` 오프셋이 다음 BSS 파싱 시점에 틀어질 수 있음.
- 이후 BSS/STA 매핑이 꼬이면서 band별 MAC(2.4G/5G) 스왑 형태로 나타날 수 있음.

---

## 3) 적용한 코드 변경

파일:
- `mapd/src/topologySrv/tlv_parsor.c`

함수:
- `parse_ap_associated_clients_type_tlv(...)`

변경 포인트:
1. `MCR_WIRELESS_EXTEND` 분기 추가
   - 신규 보정 로직은 `#ifdef MCR_WIRELESS_EXTEND` 내부로 적용
   - 기존 동작은 `#else`로 유지

2. skip BSS payload 소비 로직 추가 (신규 분기)
   - skip된 BSS라도 `sta_cnt * (ETH_ALEN + 2)` 만큼 `temp_buf`를 전진
   - 다음 BSS 파싱 오프셋 정렬 유지

3. TLV 경계 검증 추가 (신규 분기)
   - BSS header 파싱 전 경계 체크
   - skip payload 전진 전 경계 체크
   - STA 엔트리 파싱 전 경계 체크

4. 이유 주석 추가
   - 왜 `MCR_WIRELESS_EXTEND` 분기가 필요한지
   - 왜 skip BSS payload를 소비해야 하는지
   - legacy 경로(`#else`)가 어떤 동작인지

---

## 4) 전처리 분기 정책

요청사항 반영:
- **새로 추가된 동작**: `#ifdef MCR_WIRELESS_EXTEND`
- **기존 유지 동작**: `#else`

즉,
- `MCR_WIRELESS_EXTEND`가 정의되면 보정 로직 사용
- 정의되지 않으면 기존 로직 그대로 사용

---

## 5) 기대 효과

- AP_ASSOCIATED_CLIENTS TLV에서 일부 BSSID가 필터 skip되더라도
  이후 BSS/STA 파싱 오프셋 정합이 유지됨.
- `dump.txt`의 `2.4G/5G` 클라이언트 MAC 매핑 스왑 빈도 감소/해소 기대.
- `6G 정상 + 2.4/5G 스왑` 패턴의 주요 원인 하나를 차단.

---

## 6) 검증 권장 절차

1. `MCR_WIRELESS_EXTEND` define 적용 빌드
2. 동일 환경에서 `dump_topology_v1` 반복 수행
3. 동일 STA MAC 기준으로 `Medium`(2.4G/5G/6G) 매핑 일관성 비교
4. 변경 전/후 스왑 재현율 비교

권장 확인 포인트:
- 같은 STA가 dump마다 2.4G/5G로 번갈아 표기되는지 여부
- MLO 관련 BH STA 엔트리의 band 추론 결과 안정성

---

## 7) 참고

관련 작업 문서:
- `MAPD_DUMP_TXT_BACKHAUL_ANALYSIS.md`
