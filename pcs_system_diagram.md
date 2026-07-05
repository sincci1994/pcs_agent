# 메인설비–PCS 구조도 설명 (Agent 온보딩용)

> **[PCS Agent 프로젝트 참고 사항]**
> 이 문서는 배관 스케줄러(pipe-scheduler) 프로젝트의 ETL 문서에서 가져온 것이다.
> 본문 내 상대 링크(`변환규칙.md`, `../실행SQL/...` 등)는 원 프로젝트의 문서를 가리키며
> 이 프로젝트(PCS Agent)에는 존재하지 않는다.
> 본 프로젝트에서의 용도:
> - PCS 배기 계통 구조의 **정본 참조** (기획서 2.1, 판단 기준 정의서의 계통 구조 기반 판단 축)
> - §6의 압축 요약문을 **Agent 시스템 프롬프트/온보딩**에 그대로 활용

> 이 문서는 "PCS 원천 데이터와 배관 스케줄러의 설비 다이어그램이 어떤 관계인가"를
> AI Agent 또는 신규 참여자가 한 번에 이해할 수 있도록 설명하는 자료다.
> 변환 규칙의 정본은 [`변환규칙.md`](./변환규칙.md), 실행 구현은 [`../실행SQL/02_pcsdb_transform_equipment_pipe.sql`](../실행SQL/02_pcsdb_transform_equipment_pipe.sql)이다.

---

## 1. 한 문장 정의

```text
PCS(PortMaster2)는 "메인설비에 어떤 서브설비(펌프/스크러버)가 어느 위치(NOR/BYP)에
달려 있는가"를 기록한 평면(flat) 목록이고,

다이어그램은 그 목록을 변환 규칙으로 전개해 만든
"메인설비 1대의 배기(排氣) 경로 그래프"다.
  - 노드(node) = equipment (8종 타입, layer 1~5)
  - 엣지(edge) = pipe (6종 타입, from → to 방향성)
```

즉 **PCS는 부품 명세서, 다이어그램은 그 부품들을 가스 흐름 순서로 이어 붙인 회로도**다.
PCS 원천에는 "연결(배관)" 정보가 없다 — 배관 6종은 전부 변환 규칙이 위치 정보(`eqp_posn`, `eqp_chamber_posn`)로부터 **생성**한다.

---

## 2. 두 데이터 세계 비교

| 구분 | PCS 원천 (pcsdb, PortMaster2) | 다이어그램 (pipe-scheduler equipment/pipe) |
|---|---|---|
| 형태 | flat row 목록 (설비 × 서브설비 조합) | 방향 그래프 (노드 + 엣지) |
| 단위 행 | "메인설비 X의 챔버 C 경로 P에 서브설비 S가 있다" | 노드 1개 또는 배관 1개 |
| 연결 정보 | **없음** (위치 코드만 존재) | pipe의 `from_equipment_code → to_equipment_code` |
| 밸브/MERGE/DUCT | **없음** (변환 시 파생 생성) | 노드로 존재 (`SCRUBBER_NOR_VALVE` 등) |
| 그래프 소속 | 개념 없음 | `main_equipment_code` = 어느 메인설비 화면에 그릴지 |
| 렌더링 묶음 | 개념 없음 | `group_code` = 스크러버군(스크러버+밸브+MERGE) 묶음 |

### PCS 원천 컬럼 → 다이어그램 요소 매핑

| PCS 컬럼 | 의미 | 다이어그램에서의 역할 |
|---|---|---|
| `eqp_id` | 메인설비 ID | `MAIN` 노드 code, 그래프의 루트(소속 기준) |
| `eqp_chamber_id` + `eqp_chamber_posn` | 챔버와 경로 위치 | `CHAMBER` 노드 (공정 챔버는 posn별 분리) |
| `eqp_posn` = `0` | 펌프 위치 | 해당 row의 `seqp_id` → `PUMP` 노드 |
| `eqp_posn` = `NOR` | 정상 배기 경로 | NOR 밸브 노드 파생 + SS/SD1/SD2 경로 기준 |
| `eqp_posn` = `BYP` | 바이패스 경로 | BYP 밸브 노드 파생 (**상대 Pair 설비의 스크러버**를 가리킬 수 있음) |
| `seqp_id` + `seqp_ch` | 서브설비 ID + 채널 | `SCRUBBER` 노드 code (`{seqp_id}{seqp_ch}`) |
| `seqp_maker` | 제조사 | `vendor` FK (협력사 배정 기준) |

---

## 3. 다이어그램의 노드와 엣지

### 3-1. 노드 = equipment 8종 (layer가 화면 세로 단계)

| layer | type_code | 물리적 의미 |
|---|---|---|
| 1 | `MAIN` | 메인설비(공정 장비) 본체 |
| 1 | `CHAMBER` | 공정 챔버 (가스 발생 지점) |
| 2 | `PUMP` | 진공 펌프 |
| 3 | `SCRUBBER` | 유해가스 정화 장치 |
| 3 | `SCRUBBER_NOR_VALVE` | 정상 경로 밸브 |
| 3 | `SCRUBBER_BYP_VALVE` | 바이패스 경로 밸브 |
| 4 | `SCRUBBER_MERGE` | 스크러버 후단 합류 지점 |
| 5 | `LATERAL_DUCT` | 최종 배기 덕트 |

### 3-2. 엣지 = pipe 6종 (가스가 흐르는 배관)

| type_code | from → to | 경로 |
|---|---|---|
| `FORELINE` | CHAMBER → PUMP | 정상 |
| `PS` | PUMP → NOR밸브 | 정상 |
| `SD1` | NOR스크러버 → 자기 MERGE | 정상 |
| `SD2` | MERGE → DUCT | 정상 |
| `SS` | NOR밸브 → BYP밸브 | 바이패스 분기 |
| `ALLBYPASS` | BYP밸브 → BYP스크러버의 MERGE | 바이패스 |

### 3-3. 구조도 (메인설비 1대의 배기 경로)

```text
┌─ MAIN (eqp_id) ──────────────────────────────────────────────┐
│                                                              │
│  CHAMBER ──FORELINE──▶ PUMP ──PS──▶ NOR밸브                   │
│  (layer 1)            (layer 2)      (layer 3)               │
│                                        │        │            │
│                              [스크러버군: group_code 공유]      │
│                                        │        └─SS─▶ BYP밸브 ─┐
│                                        ▼                (상대 스크러버 소속)
│                                    SCRUBBER                  │ │
│                                        │ SD1                 │ │
│                                        ▼                     │ │
│                                      MERGE ◀──ALLBYPASS──────┘ │
│                                     (layer 4)   (상대 설비의 BYP밸브로부터)
│                                        │ SD2                   │
│                                        ▼                       │
│                                      DUCT (layer 5)            │
└──────────────────────────────────────────────────────────────┘
```

- **정상 흐름**: 챔버에서 발생한 가스 → 펌프 → NOR밸브 → 자기 스크러버 → MERGE → 덕트.
- **바이패스 흐름**: 자기 스크러버 점검 시 NOR밸브 → (SS) → 상대 스크러버의 BYP밸브 → (ALLBYPASS) → 상대 MERGE → 상대 덕트.
- 배관 작업 스케줄링 관점에서 이 그래프가 곧 **작업 대상(pipe 단위 워크셋)의 지도**다.

---

## 4. Pair 설비 — 다이어그램이 메인설비 경계를 넘는 유일한 지점

일부 설비(예: DES3753/DES3754)는 2대가 서로의 스크러버를 백업으로 공유한다.

```text
DES3753: 자기 스크러버(DES3753_SC011)를 NOR로 사용
         + 상대 스크러버(DES3754_SC011)를 BYP로 참조
DES3754: 그 반대 (상호 대칭)
```

이 때문에 다음이 성립한다 — **agent가 가장 오해하기 쉬운 지점**:

1. 하나의 물리 스크러버 노드가 **두 메인설비의 다이어그램 양쪽에 필요**하다
   (자기 그래프에서는 NOR 스크러버로, 상대 그래프에서는 BYP밸브/MERGE로).
2. 그러나 `equipment.code`는 전역 unique이고 `main_equipment_code`는 단일 값이라,
   "어느 그래프 소속으로 저장할지"를 변환 규칙이 결정론적으로 정해야 한다
   (결정 기준 미비 시 렌더링 실패 — [`PCS_Equipment_Pipe_현상분석보고서.md`](../PCS_Equipment_Pipe_현상분석보고서.md),
   수정 설계는 [`Pair설비_렌더링_결정론화_수정설계.md`](./Pair설비_렌더링_결정론화_수정설계.md)).
3. 소유 기준은 **NOR**이다: 스크러버 노드는 자기(NOR) 설비 그래프에 소속되고,
   상대 그래프에는 BYP밸브·MERGE 노드로만 등장한다.

NOR-only 설비(예: WTCB7G1)는 BYP가 없으므로 SS/ALLBYPASS 배관이 생성되지 않는다 — 이는 데이터 누락이 아니라 정상이다.

---

## 5. 렌더링 관점 요약

- 프론트 설비정보 화면은 **`main_equipment_code`로 필터한 equipment(노드) + pipe(엣지)** 서브셋으로 그래프 1개를 그린다.
- 따라서 "pipe가 참조하는 from/to 노드가 같은 main 서브셋 안에 존재"해야 렌더링이 성립한다. 이 정합성이 다이어그램 품질의 제1 불변식이다.
- `group_code`는 스크러버·NOR밸브·BYP밸브·MERGE를 하나의 스크러버군으로 묶는 시각적 그룹 키다 (프론트 `scrubberGroupUtils`).
- `layer_order`(1~5)는 화면 세로 배치 단계다.

상위 계층 맥락 (다이어그램 밖의 소속 정보):

```text
Division → Site → Line → Area → [MainEquipment ← 여기부터가 다이어그램] → Chamber → Pipes
```

---

## 6. Agent에게 전달할 때 권장 요약문

프롬프트/문서에 그대로 붙여 쓸 수 있는 압축 설명:

```text
이 시스템의 "설비 다이어그램"은 메인설비 1대를 루트로 하는 배기 경로 방향 그래프다.
- 노드 = equipment 테이블 (MAIN/CHAMBER/PUMP/SCRUBBER/NOR밸브/BYP밸브/MERGE/DUCT, layer 1~5)
- 엣지 = pipe 테이블 (FORELINE/PS/SS/ALLBYPASS/SD1/SD2, from→to)
- 가스 흐름: CHAMBER → PUMP → NOR밸브 → SCRUBBER → MERGE → DUCT (정상),
  NOR밸브 → 상대 설비 BYP밸브 → 상대 MERGE (바이패스)

원천은 레거시 PCS(PortMaster2)의 flat row(메인설비×서브설비×위치)이며,
배관(엣지)과 밸브/MERGE/DUCT 노드는 원천에 없고 ETL 변환 규칙이 생성한다.

주의: Pair 설비는 서로의 스크러버를 BYP로 공유하므로 하나의 물리 스크러버가
두 다이어그램에 걸친다. 노드 소속(main_equipment_code)은 NOR(소유) 기준이며,
pipe의 from/to 노드가 같은 main 서브셋에 존재해야 렌더링이 성립한다.
```

---

## 7. 관련 문서

| 문서 | 내용 |
|---|---|
| `변환규칙.md` (원 프로젝트) | 명명 규칙·챔버 분리·배관 6종 생성 규칙 (정본 레퍼런스) |
| `기준정보_전처리.md` (원 프로젝트) | 조직 enrichment, 라인 매핑 |
| `Pair설비_렌더링_결정론화_수정설계.md` (원 프로젝트) | Pair 소속 결정론화 수정 설계 |
| `PCS_Equipment_Pipe_현상분석보고서.md` (원 프로젝트) | Pair 렌더링 실패 현상 분석 |
| [`pcs_agent_planning.md`](./pcs_agent_planning.md) | PCS Agent 적용 검토 기획서 (본 프로젝트) |
| [`pcs_agent_judgment_criteria.md`](./pcs_agent_judgment_criteria.md) | 판단 기준 정의서 (본 프로젝트) |
