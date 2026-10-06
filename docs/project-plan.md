# ECG 신호 기반 1형 당뇨 저혈당 조기 탐지 시스템 설계 및 구현 — 프로젝트 계획서

**서비스명: Gluvi**

> 프로젝트의 기준 문서입니다. **사람·AI(Claude, GPT 등) 모든 팀원은 작업 전에 0장을 먼저 읽습니다.** 배경 설명과 FAQ는 [team-guide.md](team-guide.md), 시장조사는 [market-research.md](market-research.md)를 참고하세요.

### 버전 기록
| 버전 | 날짜 | 내용 |
|---|---|---|
| v8 | 2026-10-06 | 용어 통일: "저혈당 무자각" → "저혈당 무감지증(IAH)" |
| v7 | 2026-10-06 | 데이터셋 수치 정정(PhysioCGM 저혈당 기록 1,871개, D1NAMO CGM 8,414개), 품질 판정 기준 구체화, 진료 리포트 유지 결정, 무감지증 서술 정리, 현재 단계·미결 사항 갱신, PhysioCGM 데이터·코드 주소 추가 |
| v6 | 2026-10-02 | 0장 팀 합의 사항 추가(확정 결정, 하지 않기로 한 것, 결정 기록, 현재 단계, 미결 사항, 작업 규칙), 참고문헌 확인 상태 표시(✅/🔎) |
| v5 | 2026-10-02 | 소개 문서를 계획서 형식으로 개편, 팀 가이드·시장조사 분리, 제안 내용·기대 효과 근거 재배치, 참고문헌 번호 체계 도입 |
| v4 | 2026-10-02 | 시장조사 반영, Bern 2023 연구 반영해 차별점 정정 |
| v3 | 2026-10-01 | 팀 문제 정의(3가지)와 근거 반영 |
| v2 | 2026-10-01 | ECG 단독, 데이터셋 2개(D1NAMO + PhysioCGM), 실험 3축 |
| v1 | 2026-09-30 | 최초 작성 (예측 기준) |

---

## 0. 팀 합의 사항 (먼저 읽기)

### 0.1 이 문서의 역할
- 이 계획서는 팀원(사람·AI 공통)이 **같은 방향으로 작업하기 위한 기준**이다.
- 다른 문서·대화·메모와 내용이 다르면 **이 문서를 따른다.**
- 방향을 바꾸고 싶으면 먼저 팀에 제안하고, 합의되면 **0.4 결정 기록 → 본문 → 버전 기록** 순서로 반영한다.

### 0.2 확정된 결정
| 항목 | 결정 |
|---|---|
| 입력 | ECG 단독 (환자 입력 없음) |
| 과제 | 저혈당 **조기 탐지** — 정답 구간: 시작 30분 전 ~ 시작 후 15분 |
| 저혈당 정의 | 국제 합의 기준: < 3.9 mmol/L 15분 이상 시작, ≥ 3.9 mmol/L 15분 유지 시 종료 |
| 데이터 | D1NAMO + PhysioCGM (공개 데이터 2개), 메인 분석은 낮 시간 |
| 가속도계 | 모델 입력 아님. 신호 품질 판정과 분석 구간 매칭에만 사용 |
| 검증 | 처음 보는 환자(LOSO), 학습 3종 × 테스트 2종 교차 검증 |
| 메인 모델 | 특징 기반 LightGBM (딥러닝은 비교 실험) |
| 포지션 | CGM 대체가 아닌 **CGM 미사용·공백 보완용 보조 경보** |
| 주장 수준 | 가능성 확인(feasibility). 임상 적용을 주장하지 않음 |

### 0.3 하지 않기로 한 것
| 하지 않는 것 | 이유 |
|---|---|
| 혈당 **수치** 추정·회귀 | ECG가 혈당 수치를 반영한다는 근거가 약함. 이벤트 탐지만 함 |
| 식사·인슐린·활동 기록을 모델 입력에 추가 | 기록 품질이 낮아 역효과 우려, ECG 효과 증명을 흐림, PhysioCGM에 없음 (교수님 피드백) |
| CGM 과거값을 입력으로 사용 | 타깃이 CGM 미사용 상황임 |
| "30분 전 예측"으로 주장 | 직전 ECG 변화는 아직 가설. 조기 탐지로 정의하고 검증 |
| 환자를 섞은 분할 결과를 메인 성능으로 사용 | 성능이 크게 부풀려짐. 보조 비교로만 사용 |
| 테스트 결과를 보고 설정 변경 | 과적합. 바꿔야 하면 결정 기록에 이유·날짜 남김 |
| 미확인 문헌(🔎)을 제출 문서에 인용 | 직접 확인하지 않은 근거는 설명할 수 없음 |
| 측정 하드웨어 개발 | 범위 밖. 공개 데이터 신호 재생으로 검증 |
| 문제 정의에서 "무증상"을 전면에 사용 | 정상인과 혼동될 수 있음 |
| 저혈당 무감지증 환자 대상 효과를 주장 | 두 데이터셋 모두 무감지증 여부 정보가 없어 검증 불가 |

### 0.4 결정 기록
| 날짜 | 결정 | 이유·출처 |
|---|---|---|
| 09-30 | D1NAMO 기반 ECG 저혈당 주제 선정 | 저혈당 시 ECG 변화의 생리적 근거 |
| 10-01 | CGM 과거값을 입력에서 제외 (B안) | 타깃을 CGM 미사용 상황으로 설정 |
| 10-01 | 과제명에서 "웨어러블" 제외 | 하드웨어 개발이 필요해 보이는 오해 방지 |
| 10-01 | 예측 → **조기 탐지**로 변경 | 확실한 근거는 저혈당 "중" 변화. 직전 변화는 가설 |
| 10-01 | 문제 정의를 CGM 미사용 공백 중심으로 유지 | "무증상"은 정상인과 혼동 가능 |
| 10-01 | **ECG 단독**으로 변경 | 교수님 피드백: 생활 기록은 역효과 우려 |
| 10-01 | **PhysioCGM 추가** | 교수님 피드백: 데이터셋 하나로는 부족 |
| 10-01 | 실험 A를 학습 3종 × 테스트 2종으로 구성 | 테스트 대상을 고정해야 공정한 비교 가능 |
| 10-02 | 저혈당 종료 규칙 추가 | 국제 합의 기준(Battelino 2023) 반영 |
| 10-02 | 소개 문서 → 계획서 형식, 영어 파일명, 깃허브 관리 | 팀원(사람·AI) 간 방향 정렬 |
| 10-02 | 제출 문서에는 팀이 직접 확인한 문헌만 인용 | 설명할 수 없는 근거 방지 |
| 10-06 | 진료 리포트 명칭·구성 유지 ("경보 기록"으로 바꾸는 안은 보류) | 팀 결정. 대신 진단·치료 결정이 아닌 참고 자료로 한정해 표현 |
| 10-06 | 용어를 "저혈당 무감지증(impaired awareness of hypoglycemia, IAH)"으로 통일 | 국내 의료 현장 표현, 인용 문헌(IAH)과 일치. "무증상 저혈당"(사건)과 구분 |
| 10-06 | 판넬에 하드웨어 미개발 명시, 사용 흐름은 "사용 시나리오(목표)"로 표기 | 하드웨어까지 구성한다는 오해 방지 |

### 0.5 현재 단계
- 완료: 문제 정의, 시장조사, 데이터셋 분석
- 진행 중: 2026 KY-FESTA 성과전시회 판넬 제작 (분석 단계 기준)
- 다음: 설계 단계. 두 데이터셋의 ECG 데이터 양 산출, 5분 단위 정렬, 낮 시간 저혈당 에피소드 카운팅 (G1, 9장 일정 참고)

### 0.6 미결 사항
- [ ] 국내 1형 당뇨 환자 수 기준 통일 (42,994명 vs 56,908명)
- [ ] PhysioCGM 데이터 라이선스·용량 확인
- [ ] 미확인(🔎) 문헌 중 제출 문서에 쓸 문헌 직접 확인 (우선: [7] Novodvorsky 2017, [10] PhysioCGM)
- [ ] 두 데이터셋의 1형 당뇨 환자 ECG 기록 시간·품질 통과 비율 산출 (판넬에는 산출값만 사용)
- [ ] 외부 검증용 제3 데이터셋 사용 여부 (추후 결정)
- [ ] 대한당뇨병학회 진료지침의 "저혈당 무감지증" 표기 확인 (판넬·제안서 사용 전)
- [ ] PhysioCGM 원문 불일치 확인: 낮·밤 ECG 품질 서술이 본문과 그림 설명에서 반대, 나이 범위(본문 29~41세 vs 표에 24세)

### 0.7 작업 규칙 (사람·AI 공통)
1. 작업이나 제안 전에 0.2~0.3을 확인한다.
2. 확정된 결정과 다른 제안은 **"결정 변경 제안"이라고 명시**하고 근거를 함께 제시한다. 합의 전에는 본문에 반영하지 않는다.
3. 0.3의 항목은 새로운 근거가 있을 때만 다시 제안하고, 그 근거를 밝힌다.
4. 수치·주장에는 출처를 붙이고, 출처 없는 수치는 쓰지 않는다.
5. AI가 찾은 문헌은 🔎로 추가한다. 팀원이 직접 읽고 확인하면 ✅로 바꾼다.
6. 문서를 수정하면 버전 기록을, 방향을 바꾸면 결정 기록을 함께 갱신한다.
7. AI와 새 대화를 시작할 때는 이 문서를 먼저 전달해 기준으로 삼게 한다.

### 0.8 문헌 확인 상태 표시
- ✅ **팀 확인**: 팀원이 직접 찾거나 읽고 확인함 → 제출 문서에 인용 가능
- 🔎 **미확인**: AI가 수집함 → 내부 근거로만 참고, 제출 문서 인용 전 직접 확인 필요

---

## 1. 과제 개요

| 항목 | 내용 |
|---|---|
| 과제명 | ECG 신호 기반 1형 당뇨 저혈당 조기 탐지 시스템 설계 및 구현 |
| 서비스명 | Gluvi |
| 한 줄 요약 | CGM과 채혈 없이 흉부 ECG만으로 저혈당을 조기에 탐지해 알려주는 웹 시스템 |
| 대상 | CGM을 착용하지 않은 1형 당뇨 환자의 일상생활 (낮 시간대) |
| 입력 | ECG 신호 (환자가 따로 입력할 정보 없음) |
| 출력 | 저혈당 위험 상태(정상·주의·경보·판단 불가), 진료용 리포트 |
| 범위 | 신호를 받아 탐지하는 소프트웨어. 측정 하드웨어 개발은 범위에서 제외하고, 공개 데이터셋 신호를 재생해 검증한다 |

---

## 2. 문제 정의 및 근거

### 2.1 문제 정의
1. 1형 당뇨 환자에게 일상적으로 반복되는 치명적 저혈당 위험
2. 손끝 채혈의 통증과 번거로움으로 인한 혈당 측정 기피
3. 높은 비용과 불편함으로 인한 CGM 사용률 저조

> 1형 당뇨 환자는 저혈당을 일상적으로 반복해서 겪으며, 저혈당은 부정맥·사망 위험 증가와 연관된다. 그러나 손끝 채혈은 아프고 번거로워 측정을 기피하게 되고, 대안인 CGM은 비용과 절차의 부담으로 국내 1형 당뇨 환자의 89.3%가 지속적으로 사용하지 못하고 있다. 본 과제는 저혈당 시 나타나는 ECG 변화를 분석해, 채혈이나 CGM 없이 저혈당을 조기에 탐지하는 시스템을 설계·구현한다.

### 2.2 근거

**① 반복되는 치명적 저혈당 위험**

| 근거 | 내용 |
|---|---|
| 사망 위험 | 저혈당을 겪은 당뇨 환자는 정상 혈당 환자보다 전체 사망 위험이 1.68배 높았다 (RR 1.68, 95% CI 1.49–1.90) [1] |
| 심혈관 사망 | 심혈관 질환 사망 위험 1.59배 (RR 1.59, 95% CI 1.24–2.04) [1] |
| 부정맥 | 부정맥 발생 위험 1.42배 (RR 1.42, 95% CI 1.21–1.68) [1] |
| 빈도 | 1형 당뇨 환자의 83.0%가 4주 안에 저혈당을 경험했으며, 1인당 연간 73.3회(주당 약 1.4회)였다 [2] |
| 흔함 | 저혈당은 당뇨 환자에게 매우 흔한 임상 문제다 [3] |

**② 손끝 채혈 기피**

| 근거 | 내용 |
|---|---|
| 통증 | 반복적인 손끝 채혈은 통증을 유발하고 하루 중 혈당 변동을 포착하지 못하며, 환자들은 채혈 자체를 측정을 주저하게 만드는 요인으로 기술했다 [4] |
| 번거로움 | 측정에 시간이 들고 일상 활동을 중단해야 해서, 바쁜 일상에서는 측정을 우선순위에 두지 않게 된다 [4] |
| 국내 환자 인식 | 국내 1형 당뇨 CGM 사용 환자들이 꼽은 CGM의 가장 큰 장점은 "채혈 없는 혈당 측정"이었다 (57.9%, 중복 응답 중 1위) [5] |

**③ CGM 사용률 저조**

| 근거 | 내용 |
|---|---|
| 전체 사용률 | 국내 1형 당뇨 환자 56,908명 중 CGM 사용 경험 19.0%, 지속 사용 10.7% → 89.3%가 지속 사용하지 않는다 [6] |
| 성인 격차 | 성인 처방 경험 16.0%, 지속 처방 8.8% (소아 61.4%, 37.0%), 60세 이상 지속 사용 3.9%. 1형 당뇨 환자의 93.4%가 성인이다 [6] |
| 원인 | 병원 밖 선구매 후 영수증을 제출해 환급받는 번거로운 절차가 도입 장벽이다 [6] |

### 2.3 근거 인용 시 유의사항
- [1]은 1형·2형 당뇨를 함께 분석한 관찰 연구 메타분석이다. "저혈당을 겪은 환자의 위험이 높았다(연관)"로 표현하고 인과로 표현하지 않는다.
- [5]의 57.9%는 CGM 사용 환자들이 꼽은 CGM의 장점이다. "1형 당뇨 환자의 57.9%가 희망한다"로 쓰지 않는다.
- [2]는 환자 자가 기록 기반이라 인지하지 못한 저혈당은 빠져 있을 수 있다. 실제 빈도는 더 높을 수 있다.

---

## 3. 제안 내용

### 3.1 해결 방법
흉부 단일 리드 ECG를 5분 단위로 분석해, 저혈당 시작 전후의 ECG 변화를 탐지하고 경보를 제공한다. 평소에는 채혈 없이 ECG로 감시하고, **경보가 울릴 때만** 혈당측정기로 확인하도록 안내한다.

| 문제 | 대응 |
|---|---|
| ① 반복되는 저혈당 | 일상 중 저혈당을 조기에 탐지해 경보 |
| ② 손끝 채혈 기피 | 일상적인 채혈을 "경보 시 확인"으로 줄임 (처치 전 확인은 안전상 유지) |
| ③ CGM 사용률 저조 | CGM 없이 동작하는 경보 수단 제공 |

### 3.2 ECG로 저혈당을 탐지할 수 있는 근거
- 저혈당이 오면 교감신경·부신 반응(아드레날린 분비)과 혈중 칼륨 감소가 일어나고, 그 결과 심박수 증가, HRV 변화, QT 간격 연장, T파 평탄화가 나타난다.
- 일상생활 중 저절로 생긴 저혈당에서도, 1형 당뇨 환자가 휴대형 ECG와 블라인드 CGM을 함께 착용한 조건에서 QTc 연장과 T파 변화가 확인됐다 [7].
- 문제 정의의 "저혈당 시 부정맥 위험 1.42배"[1]는 저혈당이 심장에 영향을 준다는 점을 보여주며, ECG를 탐지 신호로 쓰는 근거와 이어진다.

### 3.3 저혈당 정의 (국제 합의 기준)
- **시작**: CGM < 3.9 mmol/L(70 mg/dL)가 15분 이상 지속된 첫 시점 [8]
- **종료**: CGM ≥ 3.9 mmol/L가 15분 이상 유지된 시점 [8]
- < 3.0 mmol/L(54 mg/dL)는 레벨 2로 구분해 보조 보고한다 [8]

### 3.4 데이터
ECG 원신호, CGM, 1형 당뇨를 모두 갖춘 공개 데이터셋 2개를 사용한다. 두 데이터셋 모두 같은 흉부 스트랩(Zephyr BioHarness)으로 측정되어 같은 처리 코드를 적용할 수 있다.

| 항목 | D1NAMO [9] | PhysioCGM [10] |
|---|---|---|
| 환자 | 1형 당뇨 9명 | 1형 당뇨 10명 (29~41세) |
| 기간 | 약 4일, 낮만 | 최대 17일, 밤낮 |
| ECG | 단일 리드, 250Hz | 단일 리드, 250Hz |
| 가속도계 | 흉부 3축 | 흉부 3축 100Hz |
| CGM | iPro2 | Dexcom G6 (실시간 경보) |
| CGM 기록 | 8,414개 (손끝 채혈 값은 제외하고 CGM만 사용) | 환자별 표 제공 |
| 저혈당 기록 | 에피소드 수 산출 예정 | <70 mg/dL CGM 기록 1,871개 (최소: c1s04 10개, c1s02 34개) |
| ECG 가용률 | 산출 예정 | 환자별 44.7~84.0% (기기 탈착·충전) |
| ECG 기록 시간 | 산출 예정 | 산출 예정 |
| 라이선스 | CC BY-SA 4.0 | 확인 필요 |

- 가속도계는 모델 입력이 아니라 신호 품질 판정과 분석 구간 매칭에만 사용한다.
- 혈당 단위가 다르다 (D1NAMO mmol/L, PhysioCGM mg/dL). mmol/L로 통일한다.
- ECG 데이터 양은 논문 수치(D1NAMO 약 1,550시간은 건강인 20명 포함)로 대신하지 않고 직접 산출한다.
- 식사·인슐린 기록은 사용하지 않는다 (정확도가 낮고, 모델이 ECG 대신 이 변수로 맞출 위험이 있으며, PhysioCGM에는 없다).

### 3.5 선행연구와 차별점

| 연구 | 신호 | 과제 | 검증 | 한계 |
|---|---|---|---|---|
| Porumb 2020 [11] | ECG | 야간 저혈당 검출 | 환자별 학습 | 건강인 대상 |
| Tseng 2025 [12], PhysioCGM 2025 [10] | 흉부 ECG | 저혈당 예측·검출 | 환자별 학습 | 처음 보는 환자 검증 없음, 비공개 데이터(Tseng) |
| Lehmann 2023 [13] | 손목 기기 2개 (심박·HRV·피부전도) | 저혈당 검출 | 처음 보는 사람 (22명, AUROC 0.76) | 저혈당 중 검출, 기기 2개, 비공개 데이터 |
| Dave 2024 [14] | ECG | 저·고혈당 탐지 | 개인 맞춤 | 소규모 |
| **Gluvi** | **흉부 단일 ECG** | **조기 탐지 (시작 30분 전 ~ 직후)** | **처음 보는 환자 + 공개 데이터 2개 교차 검증** | — |

**차별점**
1. **ECG 파형**: 손목 기기 수준의 심박·HRV를 넘어 QT·T파 변화까지 활용한다.
2. **조기 탐지**: 저혈당 중 검출이 아니라 시작 전후를 탐지하고, 탐지 시점 분포로 사전 탐지 가능성을 평가한다.
3. **검증 방식**: 공개 데이터 2개로 처음 보는 환자(LOSO) 및 교차 데이터셋 검증을 수행해 재현 가능하게 한다.

---

## 4. 기대 효과

### 4.1 저혈당 조기 인지와 대처 시간 확보
- 저혈당 무감지증(impaired awareness of hypoglycemia, IAH) 환자에서도 ECG의 QTc는 반응한다는 보고가 있다 (낮 저혈당 중 평균 QTc 443ms vs 정상 혈당 422ms) [15]. 따라서 **본인이 알아차리기 전에 경보를 줄 수 있다.** 다만 심박수 반응은 약할 수 있고(11장), 본 데이터에는 무감지증 여부 정보가 없어 이 집단에 대한 효과는 주장하지 않는다.
- 소아 야간 데이터에서 저혈당 70분 전부터 HRV가 변했다는 예비 보고가 있다 [16]. 성인 낮 데이터에서 이것이 확인되면 **대처 시간을 앞당길 수 있다.** 이는 본 과제의 연구 질문 1로 직접 검증한다.

### 4.2 측정 부담 감소
- 평소 감시는 채혈 없이 이루어지고, 채혈은 경보 시 확인으로 한정된다.

### 4.3 CGM 공백 보완
- CGM 미사용 환자, 센서 교체·부착 초기, 센서 탈락·통신 끊김 등 CGM이 커버하지 못하는 구간에 보조 경보를 제공한다.

### 4.4 진료 지원
- 경보 이력과 고위험 시간대를 리포트로 정리해 진료 시 회고 자료로 활용할 수 있다.
- 리포트는 진단이나 치료 결정을 위한 것이 아니라 **진료 시 참고 자료**다. CGM이 없으므로 혈당 수치가 아닌 경보 이력과 환자 확인 결과를 담는다.

> 본 과제의 주장 수준은 임상 적용이 아닌 **가능성 확인(feasibility)**이다. 기대 효과는 검증 결과에 따라 조정한다.

---

## 5. 연구 질문 및 목표

### 5.1 연구 질문
1. 저혈당이 오기 전에도 ECG가 변하기 시작하는가? 변한다면 몇 분 전부터인가?
2. ECG 정보를 깊게 쓸수록(심박수 → HRV → 파형 → 개인 기준선) 탐지가 좋아지는가?
3. 두 데이터셋을 합쳐 학습하면 탐지가 좋아지는가? 다른 데이터셋에서도 통하는가?
4. 처음 보는 환자에게 적용할 때, 특징 기반과 딥러닝 중 어느 쪽이 더 잘 버티는가?

### 5.2 목표
| 구분 | 목표 |
|---|---|
| 분석 | 저혈당 직전 ECG 변화 여부와 시점 확인 (연구 질문 1) |
| 모델 | 처음 보는 환자에 대한 저혈당 조기 탐지 성능을 공정하게 측정하고, 심박수 단독(E1) 대비 개선 확인 |
| 검증 | 두 데이터셋 교차 검증으로 일반화 수준 확인 |
| 시스템 | 경보 화면과 진료 리포트를 갖춘 웹 시스템 구현 |

---

## 6. 연구 방법

### 6.1 탐지 과제 정의
| 항목 | 내용 |
|---|---|
| 판단 단위 | 5분마다 |
| 입력 | 직전 60분 ECG |
| 정답(양성) | 저혈당 시작 30분 전 ~ 시작 후 15분 |
| 제외 | 시작 후 15분 ~ 종료, 종료 후 60분(회복), 신호 품질 불량, 기록 시작 후 첫 6시간(기준선 준비) |
| 사용 구간 | 낮 시간 창 (두 데이터셋 동일 적용) |

혈당 **수치**가 아니라 저혈당 **발생**을 탐지한다. 30분 전 변화는 아직 가설이므로, 시작 직후까지 포함하는 조기 탐지로 정의해 결과와 무관하게 과제가 성립하도록 한다.

### 6.2 시스템 흐름
```
[ECG] + [가속도계(품질 판정용)]
   → ① 신호 품질 확인 (불량 시 "판단 불가")
   → ② 특징 추출 (5분 단위: 심박수, HRV, 대표 박동의 QTc·T파·ST)
   → ③ 개인 기준선 대비 변화량 (직전 24시간 중간값 기준)
   → ④ 탐지 모델 (LightGBM 메인)
   → ⑤ 경보(10분 연속 초과, 30분 불응기) / 진료 리포트
```

- **품질 판정**: 기기 요약값 기준 **심박 신뢰도(HRC) = 100, ECG 노이즈 < 0.001** (PhysioCGM 원 연구 기준[10], 두 데이터셋 모두 Zephyr 요약 파일에 같은 값이 있어 동일 적용), 보조로 R피크 검출기 일치도(bSQI)[17], 심박수 범위(40~180), 가속도계 움직임
- **파형 추출**: 5분 동안의 박동을 평균 낸 대표 박동에서 측정한다. 파형 경계 추출은 웨이블릿 기반 방법[18]과 NeuroKit2[19]를 사용한다.
- **QTc**: 보정 공식과 측정 방법에 따라 저혈당 시 결과가 크게 달라지므로[20][21], 개인 기준선 대비 변화량을 메인으로 쓰고 Bazett·Fridericia를 함께 기록한다.

### 6.3 결과를 두 층으로 구성
| 층 | 내용 | 역할 |
|---|---|---|
| 1층: 통계 분석 | 저혈당 직전 구간 vs 평소 구간(시간대·활동량 매칭) ECG 비교, 혼합효과 모델, 데이터셋별·통합 | 연구 질문 1, 데이터셋 결합 판단 |
| 2층: 탐지 모델 | 실험 A·B·C | 연구 질문 2~4 |

### 6.4 데이터셋 결합 원칙
- 같은 원본에서 같은 코드로 처리하고, 단위·라벨·낮 시간 창을 통일한다.
- 환자별 가중치를 적용해 기록이 긴 PhysioCGM이 학습을 지배하지 않게 한다.
- 결과는 항상 데이터셋별로도 보고한다.
- **결합 중단 기준 (학습 전 고정)**: 1층에서 주요 특징의 변화 방향이 두 데이터셋에서 반대이거나, 정상 혈당 구간 대표 박동에 생리로 설명되지 않는 차이가 남거나, 라벨 기준을 맞출 수 없으면 합친 결과를 메인으로 쓰지 않는다.

---

## 7. 실험 설계 및 평가

### 7.1 실험 구성
| 실험 | 바꾸는 것 | 고정하는 것 | 우선순위 |
|---|---|---|---|
| A. 데이터 | 학습 데이터 | LightGBM, E1·E4 | 필수 |
| B. 특징 | E1 → E2 → E3 → E4 | 통합 LOSO, LightGBM | 필수 |
| C. 기법 | 규칙 / 로지스틱 / LightGBM / 1D CNN | 통합 LOSO, 같은 입력 구간 | CNN 외 필수 |

**실험 A: 학습 3종 × 테스트 2종**

| 학습 \ 테스트 | D1NAMO 환자 | PhysioCGM 환자 |
|---|---|---|
| D1NAMO만 | D1NAMO 내 LOSO | 외부 검증 |
| PhysioCGM만 | 외부 검증 | PhysioCGM 내 LOSO |
| 둘 다 | 통합 LOSO (D1NAMO 환자) | 통합 LOSO (PhysioCGM 환자) |

**실험 B: ECG 정보 깊이**

| 모델 | 입력 |
|---|---|
| E1 | 심박수 (일반 웨어러블 수준) |
| E2 | + HRV |
| E3 | + 파형 형태 (QTc, T파, ST) |
| E4 | + 개인 기준선 변화량 (최종 제안) |

### 7.2 평가 방식
- **LOSO**: 한 명씩 빼고 학습해 빠진 사람으로 테스트한다. 환자 데이터를 섞어 나누면 성능이 크게 과대평가되므로[22] 보조 실험으로 그 차이도 보고한다.
- **Nested**: 임계값·보정·하이퍼파라미터는 학습 환자 안에서만 결정한다.
- **통계 비교**: 같은 테스트 환자에서의 차이를 환자 단위 부트스트랩으로 비교한다.

### 7.3 평가 지표
| 지표 | 설명 |
|---|---|
| AUPRC | 메인 지표. 불균형 데이터에서 ROC보다 적절하다[23]. 양성 비율(바닥값)과 함께 보고 |
| 탐지율 | 저혈당 에피소드 중 정답 구간 안에 경보를 준 비율 |
| 사전 탐지 비율 | 탐지한 에피소드 중 시작 전에 경보가 울린 비율 |
| 탐지 시점 분포 | 경보 시각 − 시작 시각 (음수 = 시작 전) |
| 기록 10시간당 오경보 수 | 정답 구간 밖 경보 수 |

### 7.4 기대 성능에 대한 관점
선행연구에서 같은 환자 안에서도 날짜 단위로 나눠 검증하면 정확도가 약 60%였고(기록 단위로 섞으면 81%), 환자 독립 모델이 환자별 모델보다 나아지려면 참가자 50명 이상이 필요했다고 보고됐다[12]. 본 과제의 목표는 높은 수치가 아니라 **공정한 검증에서의 한계와 개인 기준선 정규화의 효과를 보여주는 것**이다.

---

## 8. 시스템 구현

| 항목 | 내용 |
|---|---|
| 구성 | FastAPI(백엔드, 추론) + React(프론트엔드) |
| 원칙 | 연구와 서비스가 같은 특징 추출 코드를 사용한다. **일치 테스트**: 같은 ECG 입력에 대해 연구 파이프라인과 웹 추론이 같은 특징값·확률을 내는지 확인해, 검증한 모델이 웹에서 그대로 동작함을 보장한다 |
| 환자 화면 | 상태 4단계(확률 숫자 미표시), 최근 60분 추이, 판단 근거, 대처 안내("혈당측정기로 확인하세요"), 경보 후 혈당 확인 입력 |
| 진료 리포트 | 기간 요약(기록 시간, 경보 횟수, 판단 불가 비율), 시간대 패턴, 경보 목록(시각, 확률, 주요 근거), 환자 확인 결과, PDF. 진료 시 참고 자료로 한정 |
| 시연 모드 | 데이터셋 신호 배속 재생 + 실제 CGM 곡선 겹침, 탐지 시점 표시 |

---

## 9. 추진 일정 및 역할

### 9.1 일정 (10월 초 ~ 12월 중순)
| 주차 | 내용 | 판단 지점 |
|---|---|---|
| 1~2주 | 두 데이터셋 확인, 에피소드 카운팅, 라벨 기준 확정 | G1: 에피소드 수 |
| 3~4주 | 통일, 품질 판정, 특징 추출 / 웹 뼈대(더미 모델) | G2: QT·T파 추출률 |
| 5주 | 1층 분석, 결합 여부 판단 | G3: 직전 변화, 결합 기준 |
| 6주 | 실험 B, C | |
| 7주 | 실험 A, 통계 비교 | |
| 8~9주 | 최종 모델 → 웹 연동 / (여유 시) 1D CNN | |
| 10주 | 통합 테스트, 시연 준비 | |
| 11주 | 보고서, 발표 | |

### 9.2 역할
| 파트 | 담당 | 내용 |
|---|---|---|
| 데이터 | | 데이터 확인, 통일, 라벨링 |
| 신호처리 | | 품질 판정, 특징 추출 |
| 분석·모델 | | 1층 분석, 실험 A·B·C |
| 웹 | | API, 화면, 리포트 |

---

## 10. 위험 요소 및 대응

| 위험 | 대응 |
|---|---|
| 낮 저혈당 에피소드 부족 | 라벨을 저혈당 근접(< 4.4 mmol/L)으로 완화 |
| D1NAMO 단독 학습 불가 | 결과로 보고(데이터 부족 근거), 외부 검증은 PhysioCGM → D1NAMO 중심 |
| 저혈당 직전 ECG 변화 없음 | 1층 결과를 보고하고 시작 직후 탐지 중심으로 해석 |
| 데이터셋 결합 기준 해당 | 데이터셋별·교차 검증 결과만 보고 |
| QT·T파 추출 실패 | 대표 박동 평균, 안 되면 심박수·HRV 중심 또는 1D CNN |
| 저혈당이 특정 환자에 몰림 | 환자별 결과 보고, 한계로 명시 |
| 기기 간 시계 불일치 | 공통 이벤트로 오프셋 추정·보정 |

---

## 11. 한계
- 환자 19명(유효 약 17명), 낮 데이터 중심 → 일반화와 야간 저혈당은 범위 밖
- CGM 라벨 오차: 저혈당 구간 정확도 저하, 혈중 혈당 대비 5~10분 지연, 혈액 미검증, 밤 압박 아티팩트
- PhysioCGM 환자군은 자동 인슐린 펌프 사용자로 타깃 환자와 배경이 다르다
- 저혈당 무감지증 환자는 교감신경 반응이 둔해 심박수 변화가 약할 수 있다 (QTc 반응은 남아 있다는 보고가 있음[15]). 두 데이터셋 모두 무감지증 여부 정보가 없어 이 집단은 따로 검증할 수 없다
- 흉부 스트랩 데이터 기준 → 다른 기기(패치형, 스마트워치)는 미검증
- 저혈당 경보는 의료 목적이라 실제 서비스 시 의료기기 소프트웨어(SaMD) 검토가 필요하다

---

## 12. 참고문헌

> ✅ 팀 확인 (제출 문서 인용 가능) / 🔎 미확인 (내부 참고용, 인용 전 확인 필요). 0.8 참고.

**문제 정의**
1. ✅ Li G., Zhong S., Wang X., Zhuge F. (2023). Association of hypoglycaemia with the risks of arrhythmia and mortality in individuals with diabetes – a systematic review and meta-analysis. *Frontiers in Endocrinology*, 14, 1222409.
2. ✅ Khunti K. et al., HAT Investigator Group (2016). Rates and predictors of hypoglycaemia in 27 585 people from 24 countries with insulin-treated type 1 and type 2 diabetes: the global HAT study. *Diabetes, Obesity and Metabolism*, 18(9), 907–915.
3. ✅ Polonsky W.H., Guzman S.J., Fisher L. (2023). The hypoglycemic fear syndrome: understanding and addressing this common clinical problem in adults with diabetes. *Clinical Diabetes*, 41(4), 502–509.
4. ✅ Natale P. et al. (2023). Patient experiences of continuous glucose monitoring and sensor-augmented insulin pump therapy for diabetes: a systematic review of qualitative studies. *Journal of Diabetes*, 15(12), 1048–1069.
5. ✅ Choi Y.-J. et al. (2025). Experiences of healthcare providers and patients with diabetes mellitus regarding continuous glucose monitoring use in South Korea: a multicenter, cross-sectional survey study. (학술지명 확인 필요)
6. ✅ Kim J.Y., Kim S., Kim J.H. (2025). Current status of continuous glucose monitoring use in South Korean type 1 diabetes mellitus population – pronounced age-related disparities: nationwide cohort study. *Diabetes & Metabolism Journal*, 49, 1040–1050. https://doi.org/10.4093/dmj.2024.0804

**제안 내용**

7. 🔎 Novodvorsky P. et al. (2017). Diurnal differences in risk of cardiac arrhythmias during spontaneous hypoglycemia in young people with type 1 diabetes. *Diabetes Care*, 40(5), 655–662.
8. 🔎 Battelino T., Alexander C.M., Amiel S.A., et al. (2023). Continuous glucose monitoring and metrics for clinical trials: an international consensus statement. *The Lancet Diabetes & Endocrinology*, 11(1), 42–57. https://doi.org/10.1016/S2213-8587(22)00319-9
9. 🔎 Dubosson F. et al. (2018). The open D1NAMO dataset: a multi-modal dataset for research on non-invasive type 1 diabetes management. *Informatics in Medicine Unlocked*, 13, 92–100. https://doi.org/10.1016/j.imu.2018.09.003
10. 🔎 Quamer W. et al. (2025). A multimodal physiological dataset for non-invasive blood glucose estimation (PhysioCGM). *Scientific Data*, 12, 1822. https://doi.org/10.1038/s41597-025-06090-6 / 데이터: https://doi.org/10.6084/m9.figshare.28136294 / 코드: https://github.com/PSI-TAMU/PhysioCGM / 논문 라이선스 CC BY-NC-ND 4.0 (데이터 라이선스는 figshare에서 확인 필요)
11. 🔎 Porumb M. et al. (2020). Precision medicine and artificial intelligence: a pilot study on deep learning for hypoglycemic events detection based on ECG. *Scientific Reports*, 10, 170. https://doi.org/10.1038/s41598-019-56927-5
12. 🔎 Tseng M.-R. et al. (2025). Hypoglycemia prediction in type 1 diabetes with electrocardiography beat ensembles. *Journal of Diabetes Science and Technology*. https://doi.org/10.1177/19322968251319347
13. 🔎 Lehmann V. et al. (2023). Noninvasive hypoglycemia detection in people with diabetes using smartwatch data. *Diabetes Care*, 46(5), 993–997. https://doi.org/10.2337/dc22-2290
14. 🔎 Dave D. et al. (2024). Hypoglycemia and hyperglycemia detection using ECG: a multi-threshold based personalized fusion model. *Biomedical Signal Processing and Control*.

**기대 효과**

15. 🔎 Novodvorsky P. et al. (2025). Electrocardiographic responses during spontaneous hypoglycaemia in people with type 1 diabetes and impaired awareness of hypoglycaemia. *Diabetic Medicine*, e70019. https://doi.org/10.1111/dme.70019
16. 🔎 Heart rate variability changes precede onset of nocturnal hypoglycemia in children with type 1 diabetes. HAL 03321168 (학회 초록, 저자·정식 출판 여부 확인 필요)

**연구 방법**

17. 🔎 Li Q., Mark R.G., Clifford G.D. (2008). Robust heart rate estimation from multiple asynchronous noisy sources using signal quality indices and a Kalman filter. *Physiological Measurement*, 29(1), 15–32.
18. 🔎 Martínez J.P. et al. (2004). A wavelet-based ECG delineator: evaluation on standard databases. *IEEE Transactions on Biomedical Engineering*, 51(4), 570–581. https://doi.org/10.1109/TBME.2003.821031
19. 🔎 Makowski D. et al. (2021). NeuroKit2: a Python toolbox for neurophysiological signal processing. *Behavior Research Methods*, 53(4), 1689–1696.
20. 🔎 Christensen T.F. et al. (2010). QT interval prolongation during spontaneous episodes of hypoglycaemia in type 1 diabetes: the impact of heart rate correction. *Diabetologia*, 53(9), 2036–2041.
21. 🔎 Christensen T.F. et al. (2010). QT measurement and heart rate correction during hypoglycemia: is there a bias? *Cardiology Research and Practice*. https://doi.org/10.4061/2010/961290
22. 🔎 Saeb S. et al. (2017). The need to approximate the use-case in clinical machine learning. *GigaScience*. https://doi.org/10.1093/gigascience/gix019
23. 🔎 Saito T., Rehmsmeier M. (2015). The precision-recall plot is more informative than the ROC plot when evaluating binary classifiers on imbalanced datasets. *PLoS ONE*, 10(3), e0118432.

**기타 참고**
- 🔎 Cichosz S.L. et al. (2017). Are changes in heart rate variability during hypoglycemia confounded by the presence of cardiovascular autonomic neuropathy? *Diabetes Technology & Therapeutics*.
- 🔎 Song et al. (2024). Predicting dysglycemia in patients with diabetes using electrocardiogram. *Diagnostics*. (세부 서지사항 확인 필요)
