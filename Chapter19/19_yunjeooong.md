
# 19장 페이징: 더 빠른 변환(TLB)
## 1. 페이징 성능 저하와 TLB 도입

- **페이징**: 모든 메모리 참조 시 페이지 테이블 접근 → **추가 메모리 읽기**로 성능 저하
- **해결책**: MMU 내에 작고 빠른 하드웨어 캐시, **변환-색인 버퍼(TLB)** 도입
- **TLB 역할**: 자주 참조되는 **가상→물리** 매핑 정보 저장 (address-translation cache)

---

## 2. TLB 기본 알고리즘

```pseudo
1. VPN = (VA & VPN_MASK) >> OFFSET_BITS
2. (hit, entry) = TLB_Lookup(VPN)
3. if hit then                                  # TLB 히트
4.   if !CanAccess(entry.prot) then Trap(PROT_FAULT)
5.   PA = (entry.PFN << OFFSET_BITS) | (VA & OFFSET_MASK)
6.   AccessMemory(PA)
7. else                                        # TLB 미스
8.   PTEAddr = PTBR + VPN * sizeof(PTE)
9.   PTE = AccessMemory(PTEAddr)
10.  if !PTE.Valid then Trap(SEG_FAULT)
11.  if !CanAccess(PTE.prot) then Trap(PROT_FAULT)
12.  TLB_Insert(VPN, PTE.PFN, PTE.prot)
13.  RetryInstruction()
```

- **히트**: 즉각 PFN 획득 → 오프셋 결합 → 메모리 접근
- **미스**: 페이지 테이블 접근 → TLB 갱신 → 명령 재실행

---

## 3. 예제: 배열 접근과 지역성

- **공간 지역성 (Spatial Locality)**
    - 한 페이지 내 연속 요소 접근 → 첫 번째만 미스, 이후 히트 연속 발생
    - **페이지 크기 두 배**(예: 16→32B) 시 미스 횟수 절반으로 감소 (일반적으로 4KB)
- **시간 지역성 (Temporal Locality)**
    - 한 번 참조된 페이지는 짧은 시간 내 재참조 ↑ → TLB 히트 유지

| 순차 접근    | a[0] | a[1] | a[2] | a[3] | a[4] | a[5] | a[6] | a[7] | a[8] | a[9] |
|------------|------|------|------|------|------|------|------|------|------|------|
| VPN        | 6    | 6    | 6    | 6    | 6    | 6    | 6    | 8    | 8    | 8    |
| TLB 결과    | Miss | Hit  | Hit  | Miss | Hit  | Hit  | Hit  | Miss | Hit  | Hit  |
| 히트율      |
|   70%     |      |      |      |      |      |      |      |      |      |

---

## 4. TLB 미스 처리: 하드웨어 vs 소프트웨어 

### 4.1 하드웨어 관리 TLB (CISC, x86)
- **MMU**가 멀티레벨 페이지 테이블 구조 파악
- **CR3** 레지스터에 페이지 테이블 베이스 주소 보유
- 미스 시 하드웨어가 PTE 탐색 → TLB 갱신 → 명령 재실행
- 예: Intel x86

### 4.2 소프트웨어 관리 TLB (RISC, MIPS/SPARC)
- 미스 시 **예외(Exception)** 발생 → **트랩 핸들러**로 제어 이동
- 핸들러가 페이지 테이블 검색 → **특권 명령**으로 TLB 갱신 → 명령 재실행
- 유연도↑, 하드웨어 단순화, **무한 미스 방지** 위해 핸들러는 물리 주소 참조 또는 *wired* 엔트리 사용

> _“하드웨어 신뢰도보다 소프트웨어 유연도가 중요한 경우에도 TLB를 관리할 수 있다.”_

---

## 5. TLB 구성 요소 

- **엔트리 수**: 32, 64, 128개 등
- **완전 연관(fully associative)**: 모든 엔트리 병렬 비교

| 필드        | 설명                                           |
| ----------- | ---------------------------------------------- |
| VPN         | 가상 페이지 번호 (예: 20~19bit)               |
| PFN         | 물리 페이지 번호 (예: 24bit)                  |
| valid bit   | 엔트리 유효 여부                              |
| prot bits   | 읽기/쓰기/실행 권한                            |
| ASID        | 주소 공간 식별자 (optional)                   |
| dirty bit   | 페이지 더티 여부                               |
| global bit  | 공유 페이지 ASID 무시 여부                    |
| page mask   | 지원하는 페이지 크기 식별                      |

---

## 6. 문맥 교환 문제와 ASID 

- **문제**: TLB 엔트리는 특정 프로세스 매핑만 유효 → 문맥 교환 시 이전 정보 오염
- **기본 해법**: 문맥 교환 시 TLB **플러시** (모든 valid bit=0) → 많은 TLB 미스
    - **소프트웨어**: 특권 명령어로 플러시
    - **하드웨어**: PTBR 변경 시 자동 플러시
- **ASID(Address-Space ID)** 추가 → 프로세스별 TLB 공유 가능
    - OS가 문맥 교환 시 **ASID 레지스터** 갱신
    - **global bit**를 통해 공유 페이지 ASID 무시

---

## 7. 교체 정책 이슈 

- **LRU**: 가장 오랫동안 사용 안 된 엔트리 교체
- **Random**: 무작위 엔트리 교체
- **주의**: LRU는 연속 n+1개 페이지 순환 접근 시 최악의 미스율 유발

---

## 8. 실제 TLB 사례: MIPS R4000 

- **32bit 주소 공간**, 4KB 페이지
- **구성**:
    - VPN: 19bit (유저/커널 반반)
    - PFN: 24bit (64GB 물리 메모리)
    - ASID: 8bit, G(global), C(coherence)
    - D(dirty), V(valid), page mask 필드
- **명령어**:
    - `TLBP`: 탐색
    - `TLBR`: 레지스터 로드
    - `TLBWI/TLBWR`: 엔트리 교체

---

## 9. 추가 이슈: TLB 커버리지 & 캐시 상호작용

- **TLB 커버리지**: TLB가 커버할 수 있는 페이지 수 초과 시 **미스 급증** (_Culler’s Law: “RAM은 언제나 RAM이 아니다”_)
- **캐시 인덱싱**:
    - 물리 인덱스 캐시: 주소 변환 후 접근 → 추가 지연
    - 가상 인덱스 캐시: 병렬 참조 가능 → aliasing/new HW 문제

---

## 10. 요약 및 결론 

- TLB는 페이징 성능을 **실용** 수준으로 끌어올리는 **핵심** 하드웨어
- **공간·시간 지역성** 활용 시 높은 히트율 달성
- **문맥 교환, 교체 정책, 커버리지** 한계 이해 → ASID, Huge Pages(DBMS 활용), 캐시 설계로 보완



