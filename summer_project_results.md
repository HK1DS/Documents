# 여름방학 추천 시스템 프로젝트 실험 결과 취합

작성 기준: 2026-09-09 · 대상: Goodreads Children, MovieLens 25M, Amazon Clothing

이 문서는 세 폴더의 마크다운 22개를 검토하여 작성했다. Goodreads는 보존된 결과 JSON도 대조했으며, MovieLens와 Amazon은 현재 복사본에 원시 결과 JSON이 없어 문서에 보고된 수치를 인용했다. 새 학습이나 LLM 호출을 수행한 결과는 아니다. 오래된 계획·중간 결과·폐기된 결과는 최종 결과와 구분했다.

## 1. 교수님께 보고할 공통 진행 사항

**세 도메인에 LLM 의도 추출, 지식 그래프, 최근성 기반 사용자 표현, 그래프 후보 생성·재정렬을 적용하고, 협업필터링 baseline과 구성요소별 비교 실험을 수행했다.** 현재 성과는 실행 가능한 실험 파이프라인과 다중 데이터셋의 효과·한계 분석이다. 동일한 결합 방식이 모든 데이터셋에서 정확도와 다양성을 동시에 개선한다는 결론은 얻지 못했다.

| 공통으로 진행한 작업 | 실제 수행 내용 | 보고할 때의 범위 |
|---|---|---|
| 데이터 전처리 | 사용자·아이템·상호작용·프로필·메타데이터를 파이프라인 입력으로 변환 | 세 도메인 모두 적용. 샘플링·평점 필터·k-core 조건은 서로 다름 |
| LLM 의도 추출 | 사용자·아이템 프로필에서 exact intent를 추출하고 RAG+LLM으로 related intent 확장 | 생성된 의도의 품질 자체를 정답 라벨로 검증한 것은 아님 |
| Intent/metadata KG | 의도 노드 및 저자·장르·브랜드·카테고리 등의 관계 구성 | 기본 KG, metadata, 결합 방식 비교 |
| IKGR 계열 학습 | 사용자·아이템 임베딩과 KG 정보를 결합해 BPR 방식으로 학습 | 원 논문의 모든 세부 구조를 그대로 재현했다는 의미는 아님 |
| DynLLM 아이디어 반영 | 학습 이력의 최근 상호작용에 더 큰 비중을 주는 동적 사용자 표현 | 현재 공통 구현은 recency 중심. 원 논문의 TGN·LLM 프로필 갱신 전체 구현과 구별 |
| CORONA 아이디어 반영 | 점수 결합, 그래프 후보 생성, 인기 편향 완화, soft rerank 비교 | 변형별로 별도 실험. 추론 시 LLM 후보 필터링은 공통 완료 항목에 포함하지 않음 |
| Baseline 및 반복 평가 | MF 성격의 KG-off, BPR, LightGCN과 비교하고 3개 시드로 평가 | 주요 비교는 12 epochs, embedding 512, seeds 2020/2021/2022 |
| 다면적 평가 | 전체 정확도, long-tail Recall, coverage, novelty 및 시간 분할 평가 | 시간 분할 실행 완료와 프로필의 시간 누수 검증 완료는 구별 |

공통으로 한 것은 **실험의 기본 구성과 비교 항목**이다. 현재 세 폴더의 모델·검색·평가 코드는 분기되어 있어, 아직 “동일 코드·동일 규칙으로 완료한 최종 3개 데이터셋 benchmark”라고 표현하기는 어렵다.

근거: [Goodreads 전체 진행 기록](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/AGENTS.md), [DynLLM 통합 범위](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/DYNLLM_INTEGRATION.md), [CORONA 통합 범위](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/CORONA_INTEGRATION.md), [MovieLens 최종 기록](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_jwkim_movielens-main/README.md:283), [Amazon primary 보고서](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_ercnard_amzn_clothing-main/AMAZON_CLOTHING_PRIMARY_RESULTS.md).

## 2. 데이터와 비교 기준

### 대표 결과에 사용한 데이터

| 항목 | Goodreads Children | MovieLens 25M | Amazon Clothing |
|---|---:|---:|---:|
| 대표 실험 | 원본에 k=30 적용 | 전체 25M에 k=30 적용 | 5-core 원본에서 후보 사용자 50,000명 샘플링 후 필터 |
| 사용자 | 62,142 | 133,684 | 10,515 |
| 아이템 | 27,975 | 15,785 | 11,020 |
| 상호작용 | 6,237,437 | 24,046,865 | 85,172 |
| 이벤트 의미·처리 | 독서 관련 상호작용; 기존 파이프라인은 rating=0도 포함 | 모든 평점 이벤트를 상호작용으로 보존 | rating≥4 리뷰 이벤트; 실제 구매 로그와 구별 |
| 메타데이터 | 저자·출판사·shelf | 제목·장르 | 브랜드·카테고리·속성·상품 설명 |
| 추가 데이터 조건 | k=100, 50도 실험 | 1k/5k 샘플로 사전 검증 | user-k=5, item-k=3; primary의 사전 365일 필터는 비활성 |

Goodreads의 상세 rating=0 비율은 k=100에서 34%로 보고되었으며, 이를 k=30의 비율로 옮겨 쓰지 않았다. Amazon의 85,172건은 전체 원본 규모가 아닌 전처리 샘플 규모다. 데이터 준비 문서의 EDA 규모·예시 split을 실제 최종 실험 설정과 혼동하지 않아야 한다.

근거: [Goodreads 규모](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/AGENTS.md:393), [Goodreads 평점 처리](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/IKGR_REPORT.md:22), [MovieLens 규모](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_jwkim_movielens-main/README.md:283), [Amazon 데이터 계약](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_ercnard_amzn_clothing-main/AMAZON_CLOTHING_PRIMARY_RESULTS.md:28).

### 지표와 해석

| 표기 | 의미 |
|---|---|
| TO | 사용자별 시간순 train/valid/test 분할, 설정 비율 80/10/10. 아래 대표 결과는 학습 이력이 있는 사용자 평가 |
| TO_GLOBAL | 전체 시간순 70/10/20 분할. train 이력이 없는 사용자가 test에 등장할 수 있음 |
| NDCG@10 | 상위 10개 추천의 관련성과 순위를 함께 반영하는 정확도 지표 |
| Recall@10 | 사용자의 정답 아이템 중 상위 10개 추천으로 회수한 비율 |
| Tail R@10 | 학습 인기도 기준 하위 80% 아이템에 대한 Recall@10 |
| Coverage@10 | 전체 평가 사용자의 top-10 추천에 등장한 서로 다른 아이템의 카탈로그 대비 비율 |
| Novelty | 학습 인기도에 기반한 추천 아이템의 희귀성. 의미적 다양성이나 설명 품질과는 다른 지표 |
| cold0_train | 학습 상호작용이 0개이고 평가에 등장한 사용자 |

표의 NDCG는 가능한 경우 3시드 평균±모표준편차(ddof=0), 나머지는 3시드 평균을 적었다. **세 데이터셋의 점수를 서로 평균하거나 절대값으로 성능 순위를 매기지 않는다.** 각 데이터셋의 동일 평가표 안에서 baseline 및 구성요소 변형을 비교한다. 소수 넷째 자리까지 반올림된 수치의 작은 차이는 통계적 유의성의 증명이 아니다.

### 모델명 대응

| 코드의 이름 | 보고서에서의 의미 |
|---|---|
| `IKGR_kgoff` | KG를 끈 MF 성격의 대조군 |
| `IKGR_full_hetero` | Intent KG + metadata KG |
| `IKGR_dyn` | KG 구성 + 최근성 기반 사용자 표현 |
| `IKGR_full` | CF·KG·recency 점수를 명시적으로 결합한 late fusion |
| `IKGR_cand` | 그래프 후보군으로 추천 대상을 제한한 변형 |
| `IKGR_cand_db` | 인기 편향 완화를 적용한 후보 제한 변형 |
| `IKGR_rerank_db_rel` | 전체 추천 점수에 graph prior를 더하는 soft rerank |

`IKGR_full`은 특정 점수 결합 실험의 이름이며, 원래 계획한 모든 기능이 완성되었다는 뜻이 아니다. `IKGR_cand_db`와 soft rerank 역시 하나의 직렬 파이프라인에 반드시 함께 들어가는 단계가 아니라 비교한 대안들이다.

## 3. 공통 구성요소의 대표 TO 결과

### Goodreads Children: k=30

**기존 프로필을 사용한 과거 실험 기록이다.** 아래 수치는 JSON으로 확인했지만, 프로필 생성 시점의 train-only 보장은 아직 재검증이 필요하다. TO_GLOBAL뿐 아니라 최종 TO 실험에서도 이 조건을 확인해야 한다.

| 모델 | NDCG@10 | Recall@10 | Tail R@10 | Tail R@30 | Coverage@10 | Novelty |
|---|---:|---:|---:|---:|---:|---:|
| KG-off | 0.0795 ± 0.0012 | 0.0854 | 0.0105 | 0.0218 | 0.3647 | 10.2731 |
| Intent + metadata KG | 0.0771 ± 0.0012 | 0.0840 | 0.0139 | 0.0280 | 0.5008 | 10.7101 |
| KG + recency | 0.0822 ± 0.0005 | 0.0899 | 0.0170 | 0.0325 | 0.5164 | 11.0065 |
| CORONA de-biased candidate | 0.0127 ± 0.0002 | 0.0126 | 0.0273 | 0.0454 | 0.4581 | 16.7904 |
| BPR | 0.0814 ± 0.0008 | 0.0864 | 0.0110 | 0.0221 | 0.3641 | 10.2572 |
| LightGCN | 0.0820 ± 0.0006 | 0.0841 | 0.0047 | 0.0101 | 0.1757 | 9.2672 |

KG에 recency를 추가하면 NDCG는 0.0771에서 0.0822, Tail R@10은 0.0139에서 0.0170으로 증가했다.

관찰: KG+recency는 BPR와 비슷한 수준의 NDCG를 기록하면서 Tail R@10은 약 54.5%, coverage는 약 41.8% 높았다. 이는 반올림된 평균의 상대 변화이며, NDCG에서 baseline을 확실히 이겼다고 단정할 차이는 아니다. De-biased candidate는 tail을 더 높였지만 전체 정확도가 크게 낮아졌다.

출처: [k=30 진행 기록](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/AGENTS.md:393), [TO 결과 JSON](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/run_k30/slice_eval_TO_result.json).

### MovieLens 25M: 전체 데이터 k=30

| 모델 | NDCG@10 | Recall@10 | Tail R@10 | Coverage@10 | Novelty |
|---|---:|---:|---:|---:|---:|
| KG-off | 0.0769 ± 0.0010 | 0.0574 | 0.0030 | 0.4495 | 10.2167 |
| Intent + metadata KG | 0.0770 ± 0.0010 | 0.0579 | 0.0041 | 0.5142 | 10.3562 |
| KG + recency | 0.0796 ± 0.0006 | 0.0600 | 0.0033 | 0.4602 | 10.3268 |
| Late fusion (`IKGR_full`) | 0.0822 ± 0.0003 | 0.0625 | 0.0033 | 0.4488 | 10.3246 |
| CORONA de-biased candidate | 0.0012 ± 0.0000 | 0.0005 | 0.0022 | 0.2148 | 18.7689 |
| BPR | 0.0769 ± 0.0010 | 0.0574 | 0.0030 | 0.4510 | 10.2215 |
| LightGCN | 0.0854 ± 0.0005 | 0.0611 | 0.0013 | 0.3366 | 9.8725 |

관찰: LightGCN이 NDCG 기준 가장 높았다. KG+recency와 late fusion은 KG-off 대비 정확도를 높였고, intent+metadata 구성은 주요 full-sort IKGR 변형 중 tail/coverage가 높았다. 반면 de-biased candidate는 novelty만 크게 높이고 정확도뿐 아니라 tail/coverage도 KG+recency보다 낮았다. 따라서 “CORONA 편향 제거는 모든 데이터셋에서 long-tail을 개선한다”는 주장은 성립하지 않는다. 표준편차 0.0000은 문서의 반올림 표기다.

Naive candidate는 NDCG 0.0824였지만 Tail R@10 0.0000, coverage 0.0809로 추천 범위가 매우 좁아졌다. Soft rerank λ=0.50은 KG+recency 대비 NDCG 0.0796→0.0787, Tail R@10 0.0033→0.0043, coverage 0.4602→0.4997의 교환 관계를 보였다.

출처: [전체 MovieLens TO 최종 표·해석](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_jwkim_movielens-main/README.md:346). 원시 JSON은 현재 복사본에 없음.

### Amazon Clothing: primary sample

| 모델 | NDCG@10 | Recall@10 | Tail R@10 | Tail R@30 | Coverage@10 | Novelty |
|---|---:|---:|---:|---:|---:|---:|
| KG-off | 0.1744 ± 0.0012 | 0.1976 | 0.0440 | 0.0529 | 0.5757 | 11.4234 |
| Metadata only | 0.1359 ± 0.0028 | 0.1690 | 0.0437 | 0.0612 | 0.4255 | 11.6374 |
| Intent + metadata KG | 0.0381 ± 0.0016 | 0.0739 | 0.0126 | 0.0200 | 0.2079 | 10.2742 |
| KG + recency | 0.1287 ± 0.0007 | 0.1674 | 0.0351 | 0.0504 | 0.3681 | 11.7259 |
| Late fusion (`IKGR_full`) | 0.1781 ± 0.0008 | 0.2029 | 0.0484 | 0.0586 | 0.6383 | 11.9374 |
| CORONA naive candidate | 0.1336 ± 0.0007 | 0.1750 | 0.0394 | 0.0587 | 0.4629 | 11.9383 |
| CORONA de-biased candidate | 0.1530 ± 0.0022 | 0.1933 | 0.0501 | 0.0734 | 0.6198 | 13.1237 |
| BPR | 0.1744 ± 0.0011 | 0.1972 | 0.0442 | 0.0535 | 0.5722 | 11.4306 |
| LightGCN | 0.1575 ± 0.0010 | 0.1802 | 0.0292 | 0.0381 | 0.2159 | 10.1243 |

관찰: Late fusion은 BPR 대비 NDCG 평균이 약 2.1%, coverage가 약 11.6% 높았다. De-biased candidate는 tail/novelty가 높지만 NDCG는 BPR보다 낮았다. Intent+metadata의 단순 결합이 크게 부진한 반면 late fusion은 개선되어, **부가 정보를 넣는 것 자체보다 결합 방식이 중요할 수 있다는 후속 가설**을 준다. 원인과 유의성은 별도 검증이 필요하다.

이 표는 primary 보고서의 실험 묶음이다. 이후 가중치 ablation에서 재실행한 uniform 모델의 NDCG는 0.0337로, 여기의 0.0381과 다르다. 두 숫자를 같은 실행 결과로 섞지 않았다.

출처: [Amazon primary TO 결과](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_ercnard_amzn_clothing-main/AMAZON_CLOTHING_PRIMARY_RESULTS.md:77). 원시 JSON은 현재 복사본에 없음.

## 4. Global temporal / cold-start 결과

**MovieLens와 Amazon은 학습 시간 구간으로 제한한 프로필을 사용했다고 문서에 기록되어 있다. Goodreads의 기존 global 결과는 같은 조건을 보장하지 못하므로 참고 기록으로 분리한다.** Static catalog metadata를 사용하는 평가와 test-only 아이템의 그래프 연결까지 제외하는 strict inductive 평가는 서로 다른 설정이다.

### MovieLens 전체 k=30: global-train 프로필

평가 사용자 27,854명 중 cold0_train 21,636명. 전체 사용자 중 빈 프로필인 34,888명과 평가에 포함된 cold0 수를 구별한다.

| 모델 | 전체 NDCG@10 | cold0 NDCG@10 | Tail R@10 | Coverage@10 |
|---|---:|---:|---:|---:|
| KG-off | 0.0343 ± 0.0004 | 0.0139 | 0.0003 | 0.8527 |
| Metadata only | 0.0366 ± 0.0002 | 0.0169 | 0.0003 | 0.8676 |
| Intent + metadata KG | 0.0393 ± 0.0004 | 0.0203 | 0.0003 | 0.8730 |
| KG + recency | 0.0421 ± 0.0002 | 0.0244 | 0.0003 | 0.8549 |
| Late fusion | 0.0400 ± 0.0001 | 0.0211 | 0.0002 | 0.8600 |
| CORONA naive candidate | 0.0238 ± 0.0001 | 0.0011 | 0.0002 | 0.0822 |
| CORONA de-biased candidate | 0.0035 ± 0.0004 | 0.0011 | 0.0005 | 0.0929 |
| BPR | 0.0345 ± 0.0003 | 0.0143 | 0.0003 | 0.9181 |
| LightGCN | 0.0714 ± 0.0007 | 0.0581 | 0.0004 | 0.6931 |

LightGCN이 전체와 cold0 정확도에서 가장 높았다. KG 계열 일부는 KG-off/BPR보다 높지만, 가장 강한 baseline 대비 cold-start 개선으로 보고할 수 없다. Soft rerank λ=0.50의 전체 NDCG는 0.0417로, λ=0의 0.0421보다 낮고 반올림된 tail/coverage 변화는 거의 없었다.

출처: [MovieLens global 최종 결과](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_jwkim_movielens-main/README.md:283).

### Amazon primary: global-train 프로필

평가 사용자 4,765명 중 cold0_train 721명. 전체 데이터의 빈 프로필 1,252명과는 다른 집계다.

| 모델 | 전체 NDCG@10 | cold0 NDCG@10 | cold0 Recall@10 | Tail R@10 | Coverage@10 |
|---|---:|---:|---:|---:|---:|
| KG-off | 0.0103 ± 0.0004 | 0.0254 | 0.0220 | 0.0033 | 0.3088 |
| Metadata only | 0.0085 ± 0.0003 | 0.0100 | 0.0093 | 0.0062 | 0.3768 |
| Intent + metadata KG | 0.0063 ± 0.0004 | 0.0048 | 0.0047 | 0.0017 | 0.3096 |
| KG + recency | 0.0098 ± 0.0004 | 0.0041 | 0.0035 | 0.0050 | 0.4140 |
| Late fusion | 0.0091 ± 0.0002 | 0.0074 | 0.0073 | 0.0047 | 0.5581 |
| CORONA naive candidate | 0.0100 ± 0.0002 | 0.0023 | 0.0020 | 0.0053 | 0.4160 |
| CORONA de-biased candidate | 0.0080 ± 0.0001 | 0.0023 | 0.0020 | 0.0070 | 0.5504 |
| BPR | 0.0090 ± 0.0004 | 0.0129 | 0.0130 | 0.0042 | 0.4830 |
| LightGCN | 0.0101 ± 0.0008 | 0.0301 | 0.0245 | 0.0025 | 0.1598 |

Cold0에서는 LightGCN과 KG-off가 강했다. De-biased candidate는 전체 tail/coverage를 높이는 방향이지만 cold0 정확도는 낮았다. 후보 수 M=500에서 candidate Recall@M은 TO의 naive/de-biased가 각각 0.3686/0.3555, TO_GLOBAL은 0.1289/0.1318이었다. 정답이 후보 밖에 있으면 뒤의 ranker가 회수할 수 없어, 후보 수 감소를 효과나 속도 개선으로만 해석하면 안 된다.

출처: [Amazon global 결과·cold0·candidate ceiling](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_ercnard_amzn_clothing-main/AMAZON_CLOTHING_PRIMARY_RESULTS.md:109).

### Goodreads k=30: 재검증이 필요한 과거 global 결과

| 모델 | 전체 NDCG@10 | cold0 NDCG@10 | cold0 Recall@10 |
|---|---:|---:|---:|
| KG-off | 0.0862 | 0.0140 | 0.0022 |
| Intent + metadata KG | 0.1209 | 0.3867 | 0.0677 |
| KG + recency | 0.1145 | 0.3704 | 0.0638 |
| CORONA de-biased candidate | 0.0232 | 0.0669 | 0.0093 |
| BPR | 0.0885 | 0.0171 | 0.0029 |
| LightGCN | 0.0932 | 0.0475 | 0.0078 |

문서 및 JSON에는 cold0 사용자 2,736명에서 높은 KG 성능이 기록되어 있다. 그러나 사용자 프로필이 global train 이전 정보만으로 만들어졌다는 보장이 없다. 현재 전처리 코드는 timestamp를 읽으면서도 프로필 생성에 시간 컷을 적용하지 않는다. 따라서 위 수치를 “엄격한 cold-start에서 baseline을 수십 배 개선했다”는 최종 성과로 사용하지 않는다. 외부에서 이미 관측 가능한 사용자 프로필을 쓰는 별도 cold-start 설정으로 해석하려 해도, 해당 정보의 이용 가능 시점을 먼저 입증해야 한다.

출처: [기존 결과 JSON](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/run_k30/slice_eval_TO_GLOBAL_result.json), [프로필 생성 코드](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/goodreads_preprocess.py:112), [후속 MovieLens 문서의 Goodreads 비교 제한](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_jwkim_movielens-main/AGENTS.md:207).

## 5. 데이터셋별 추가 실험

### Goodreads: k-core 확장과 실패한 결합 방식도 검증

아래는 TO의 Tail R@10 평균이다. 각 열은 다른 데이터 부분집합이므로 k 감소의 인과효과나 성능의 단조 증가를 입증하는 표는 아니다. 기존 프로필 조건의 재검증 필요성도 그대로 적용된다.

| 모델 | k=100 | k=50 | k=30 |
|---|---:|---:|---:|
| KG-off | 0.0089 | 0.0067 | 0.0105 |
| Intent + metadata KG | 0.0108 | 0.0102 | 0.0139 |
| KG + recency | 0.0110 | 0.0120 | 0.0170 |
| CORONA de-biased candidate | 0.0275 | 0.0246 | 0.0273 |
| BPR | 0.0092 | 0.0064 | 0.0110 |
| LightGCN | 0.0052 | 0.0028 | 0.0047 |

세 k값에서 KG+recency의 tail 평균이 BPR보다 높다는 관찰은 유지된다. 다만 일부 문서의 “희소해질수록 이득이 계속 커진다”는 표현은 이 세 점 전체에서 단조롭게 성립하지 않는다.

실험 과정에서 확인한 사항은 다음과 같다.

- k=100 RS에서 초기 고정 휴리스틱 점수의 문제를 수정했다. KG-off NDCG가 약 0.045에서 0.294 수준으로 회복하여 BPR 수준에 근접했다. 이는 구현 수정 효과이며 새 모델의 독립적인 성능 기여와 구별한다.
- k=100의 intent-only KG 단일시드 tail 이득은 12-epoch 다중시드 실험에서 유지되지 않았다. 학습 가능한 intent 노드는 frozen 변형보다 결과 변동이 컸다.
- Metadata-only KG는 k=100 RS에서 Tail R@10 0.0658, MF는 0.0597이었다. 전체 NDCG는 0.2677 대 0.2922로 낮았다.
- k=100 TO에서 recency에 attention을 추가하면 Tail R@10 0.0110→0.0097, coverage 약 0.670→0.642로 낮아져 채택하지 않았다.
- 같은 k=100 TO에서 late fusion은 KG+recency 대비 NDCG 0.0718→0.0678, Tail R@10 0.0110→0.0101로 낮아졌다.
- k=30 TO_GLOBAL soft rerank는 결정성 검증 후 λ=0/0.01/0.03/0.05를 3시드로 확인했다. λ=0→0.05에서 NDCG 0.1150→0.1147, tail 0.0038→0.0039, coverage 0.3477→0.3503이었다. 기존 모델 결과와는 다른 재실행 묶음이며, 유의한 성능 개선으로 채택하지 않았다.

출처: [IKGR 실험](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/IKGR_REPORT.md), [DynLLM 실험](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/DYNLLM_REPORT.md), [CORONA 실험](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/CORONA_REPORT.md), [최신 low-lambda 기록](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/AGENTS.md:474), [k=50 JSON](C:/Users/jwkim/Desktop/Summer_Vacation/Goodreads_experiment-main/run_k50/slice_eval_TO_result.json).

### MovieLens: 소규모 검증에서 전체 데이터 실험까지 확장

- 1k 샘플: 953 users / 1,855 items / 121,178 interactions. 파이프라인 점검용이며 초기 ID 매핑 문제 이전 수치는 최종 성과로 사용하지 않는다.
- 5k 샘플: 4,945 users / 4,954 items / 723,917 interactions. ID 매핑 수정 후 TO에서 KG-off NDCG/Tail R@10/coverage는 0.0790/0.0031/0.3089, KG+recency는 0.0799/0.0063/0.3714였다.
- 5k global-safe 재실험: 전체 NDCG는 LightGCN 0.1402, KG-off 0.0870, KG+recency 0.0608. 단순히 모델을 결합한다고 global 성능이 좋아지지 않았다.
- 전체 k=30은 TO와 TO_GLOBAL의 프로필·LLM cache·banks·KG·결과를 분리하고 각 13개 결과 그룹×3시드 평가를 완료했다고 기록되어 있다.
- 전체 global의 exact/related 추출 대상 프로필 텍스트는 113,448개씩이다. 기록된 215,231개 usage 응답, 187,759,897 토큰은 해당 ledger의 집계 범위이며 프로젝트 전체 비용으로 간주하지 않는다.

출처: [MovieLens 준비·중간 검증·최종 실험 기록](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_jwkim_movielens-main/README.md:176), [세부 진행 기록](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_jwkim_movielens-main/AGENTS.md:13).

### Amazon: LLM 호출 통합과 intent 가중치 ablation

TO, 12 epochs, 3시드의 별도 ablation이다. 여기서 “기존 3단계”는 **exact 추출 → related 확장 → 그래프 생성**을 뜻한다. IKGR·DynLLM·CORONA라는 세 모델 구성 전체를 한 LLM 호출로 대체했다는 뜻이 아니다. 통합 방식에서도 그래프 생성과 추천 모델 학습은 별도로 수행된다.

| 방식 | 실제 API 호출 | Raw billable token | 그래프 생성 시간 | NDCG@10 | Recall@10 |
|---|---:|---:|---:|---:|---:|
| 기존 exact/related 분리 추출 | 41,656 | 29,129,669 | 721.27초 | 0.0337 ± 0.0013 | 0.0666 ± 0.0013 |
| exact/related 통합 추출 | 20,486 | 7,118,217 | 1,323.44초 | 0.0444 ± 0.0010 | 0.0782 ± 0.0015 |

보고된 수치 기준 API 호출은 약 50.8%, raw token은 약 75.6% 감소했다. 반면 그래프 생성 시간은 약 83.5% 증가했다. Raw billable은 재시도·재호출을 포함하며, 실제 금액 절감률은 토큰 단가와 입력/출력 구성 확인 없이 동일하게 단정하지 않는다. 통합 방식의 related intent는 기존 고정 후보 vocab 기반 RAG 선택과 생성 조건도 달라, 후속 비교에서는 의도 품질과 후보 범위까지 확인할 필요가 있다.

| Intent edge | NDCG@10 | Recall@10 | NDCG std |
|---|---:|---:|---:|
| Uniform | 0.0337 | 0.0666 | 0.0013 |
| IDF weighted | 0.0361 | 0.0701 | 0.0018 |

문서는 NDCG +7.12%를 보고하지만 표준편차도 증가했다. 따라서 가중치의 성능·안정성 개선이 모두 검증되었다고 쓰지 않는다. 현재 코드에서는 1-layer 집계에 가중치가 들어가고, 다층 희소 전파 행렬은 균등 정규화를 사용한다. 추가 ablation은 사용할 전파 경로를 명시해야 한다.

별도 smoke 실험에서는 strict train-only KG와 validation으로 선택한 λ=0.75 설정의 NDCG 0.3712±0.0077이 보고되어 있다. 2,998 interactions / 292 users / 275 items의 소규모 실험이므로 primary의 85,172건 결과와 직접 비교하지 않는다. 이전 NDCG 0.8756은 문서에서 폐기된 결과다.

출처: [Amazon ablation 보고서](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_ercnard_amzn_clothing-main/AMAZON_CLOTHING_ABLATION_RESULTS.md), [strict smoke 최종 기록](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_ercnard_amzn_clothing-main/README.md:232), [통합 추출 코드](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_ercnard_amzn_clothing-main/step_unified.py), [가중치 적용 코드](C:/Users/jwkim/Desktop/Summer_Vacation/hk1ds_ercnard_amzn_clothing-main/ikgr_core/model_ikgr.py:316).

## 6. 현재 결과로 말할 수 있는 기여와 남은 한계

| 보고할 수 있는 내용 | 아직 확정하기 어려운 내용 |
|---|---|
| 세 도메인에 공통 추천 파이프라인을 적용하고 구성요소별 비교 실험을 수행 | 동일 최종 코드로 세 데이터셋의 모든 조건을 통일해 검증 완료 |
| KG·recency·검색 방식에 따라 정확도, tail, coverage의 교환 관계가 달라짐 | Full 결합이 항상 부분 조합·강한 baseline보다 우수 |
| Amazon primary TO의 late fusion에서 BPR 대비 평균 정확도·coverage 개선 관찰 | 모든 데이터셋 또는 실제 서비스에서 같은 개선을 보장 |
| MovieLens에서 큰 데이터와 엄격한 global profile 분리로 평가 진행 | KG 계열의 보편적인 cold-start 우위 |
| Amazon에서 통합 추출의 API 호출·토큰 감소와 성능 변화 측정 | 같은 intent 품질 보장, end-to-end 처리 시간 단축, 동일 비율의 금액 절감 |
| 실패한 attention·candidate·fusion 변형도 결과로 보존 | 낮은 성능만으로 해당 메커니즘 일반의 무효성을 입증 |

최종 결과물에서는 다음 사항을 정리할 필요가 있다.

1. **프로필·그래프의 정보 이용 시점:** 특히 Goodreads의 train-only 프로필을 재구성·확인하고, static metadata 허용 범위와 cold-start 정의를 명시한다.
2. **동일 비교 조건:** 공통 코드 버전, 후보군 생성 방식, ID 매핑, 평점 정책, 학습·튜닝 예산을 기록한다. 12 epochs에서의 결과를 충분히 튜닝된 baseline 대비 우위와 동일시하지 않는다.
3. **구성요소의 실제 기여:** metadata만으로 얻는 효과와 LLM intent 추가 효과, recency 효과, 검색/재정렬 효과를 같은 조건에서 분리한다. 후보 수와 λ는 validation으로 선택한다.
4. **결과의 추적 가능성:** seed별 JSON, config, 데이터·프로필 manifest, cache 식별값, 최종 코드 버전을 연결한다. Goodreads 과거 주요 JSON에는 실행 signature가 없고, 현재 복사본에는 MovieLens·Amazon 원시 JSON과 세 데이터셋의 학습 입력이 없다.
5. **제품 시연:** 현재 자료는 오프라인 실험 중심이다. 사용자 입력부터 실제 추천까지 이어지는 시연과 응답시간 측정이 필요하다면 별도 구현·평가해야 한다. Coverage는 사용자 만족도나 추천 설명의 정확성을 대체하지 않는다.

## 7. 졸업작품 완성을 위해 교수님께 제안할 범위

아래는 현재 자료를 바탕으로 한 **협의안**이다. 학과나 교수님의 확정된 졸업작품 기준을 뜻하지 않는다. 성능 우위를 새로 만들어내는 것보다, 주장 범위에 맞는 비교와 재현 가능한 결과물을 먼저 완성하는 방향으로 제안할 수 있다.

| 구분 | 제안하는 결과물 | 완료 기준의 예 |
|---|---|---|
| 실험 정리 | 공통 코드·설정·최종 표 | 핵심 모델의 seed별 수치가 설정·데이터·코드와 연결되고, 폐기된 결과가 분리됨 |
| 검증 보강 | 누수 점검과 최소 구성요소 ablation | 동일 데이터에서 baseline, intent-only, metadata-only, intent+metadata, +recency, 선택한 fusion/rerank를 비교 |
| 시연 시스템 | 사용자 선택/이력 입력 → top-10 추천 → baseline 비교 | 저장된 intent·KG·모델을 사용해 실제 추천이 나오고, 최근성/롱테일 차이를 예시로 확인 가능 |
| 설명 자료 | 방법·실험·한계·재현 절차·시연 영상 또는 발표 자료 | 원 논문 아이디어와 직접 구현·변경한 부분을 구별하고 실패한 실험도 설명 |
| 선택적 연구 확장 | 통합 LLM 추출 또는 가중치 설계 중 한 가지 추가 검증 | 비용·품질·정확도를 같은 조건에서 비교하고, 필요 시 다른 데이터셋에서 확인 |

최소 공통 실험을 설계할 때는 현재 완료된 변형을 최대한 재사용하되, metadata-only와 intent-only 대조군을 포함하는 편이 LLM의 추가 가치를 설명하기 좋다. 단순히 모든 변형을 다시 돌리기보다, 누수·코드 변경으로 영향을 받는 결과와 핵심 주장에 필요한 결과를 우선 확정하는 방식이다.

Cold-start를 핵심 기여로 유지한다면 세 데이터셋에서 같은 정보 이용 조건으로 재검증하는 범위가 추가된다. 정확도·tail·coverage의 관계와 실용적인 추천 시연을 중심으로 삼는다면 cold-start는 한계 분석으로 남기는 범위도 교수님께 제안할 수 있다.

### 교수님께 여쭤볼 핵심 질문

1. **완성 기준:** 재현 가능한 세 데이터셋 비교, 구성요소별 효과·한계 분석, 동작하는 추천 데모까지를 졸업작품의 기본 완성 범위로 잡아도 될까요? 추가 알고리즘 기여나 명확한 성능 우위가 어느 정도 필요한가요?
2. **추가 실험 우선순위:** 세 데이터셋의 동일 조건 재검증, cold-start 보완, Amazon의 통합 추출·가중치 확장 중 어떤 부분을 학기 내 필수 범위로 두는 것이 좋을까요?
3. **서비스 구현 깊이:** 저장된 모델을 사용하는 오프라인/로컬 데모면 충분할까요, 아니면 웹 서비스·실시간 갱신·응답시간 평가까지 필요한가요?

현재 진행을 설명하는 문장으로는 다음이 적절하다.

> 여름방학 동안 LLM으로 추출한 의도, 지식 그래프, 최근성 기반 사용자 표현, 그래프 검색·재정렬을 결합한 추천 파이프라인을 세 데이터셋에 적용하고 구성요소별 비교 평가를 수행했습니다. 결합 방식에 따라 정확도와 롱테일·커버리지에 서로 다른 효과가 나타났으며, 이를 바탕으로 공통 검증 기준과 재현 가능한 실험 결과, 추천 시연을 갖춘 졸업작품으로 정리하고자 합니다.

## 8. 문서 채택 기준 및 검토 목록

Goodreads의 `README.md`는 주로 초기 IKGR 단계, `explain.md`는 후속 Goodreads 종료 시점의 해석이다. 최신 k=30 수치와 soft rerank 상태는 `AGENTS.md` 후반 및 JSON을 우선했다. 기존 cold-start 해석은 MovieLens 후속 문서와 현재 프로필 코드에 비추어 제한했다. `result_example.md`의 초기 샘플 결과와 잘못된 호출 단가 기반 비용 예측은 최종 결과에 채택하지 않았다.

MovieLens는 `README.md`와 `AGENTS.md`의 2026-08-06 전체 k=30 완료 기록을 우선했다. 초기 ID 연결 문제 이전 결과 및 per-user 프로필로 수행한 구 TO_GLOBAL 결과는 제외했다. `movie_lens.md`는 데이터 준비 예시·계약 자료로 사용했다.

Amazon은 `AMAZON_CLOTHING_PRIMARY_RESULTS.md`를 primary 수치, `AMAZON_CLOTHING_ABLATION_RESULTS.md`를 추가 실험 수치로 사용했다. `README.md`의 최종 strict smoke와 과거 폐기 결과를 구분했다. 2026-09-08 진행 보고서의 설명·한계도 참고했지만, 가중치 적용 여부는 현재 코드의 L1/L2 경로를 구별했다. `amzn_clothing.md`는 입력 계약·EDA 참고 자료로 사용했다.

| 폴더 | 검토한 마크다운 | 용도 |
|---|---|---|
| Goodreads | `AGENTS.md`, `README.md`, `IKGR_REPORT.md`, `DYNLLM_REPORT.md`, `CORONA_REPORT.md`, `explain.md` | 실험 경과·수치·해석 |
| Goodreads | `DYNLLM_INTEGRATION.md`, `CORONA_INTEGRATION.md`, `IKGR_README.md` | 설계·원래 목표와 실제 구현 범위 |
| Goodreads | `path.md`, `result_example.md` | 산출물 위치·초기 샘플 및 채택 제외 항목 |
| Goodreads | `CORONA-main/README.md`, `dynmLLM/README.md` | 참조 코드 설명. 프로젝트 자체의 실험 성과로 집계하지 않음 |
| MovieLens | `AGENTS.md`, `README.md`, `movie_lens.md` | 최종 실험·후속 수정·입력 계약 |
| Amazon | `AGENTS.md`, `README.md`, `amzn_clothing.md` | 공통 파이프라인·smoke 경과·입력 계약 |
| Amazon | `AMAZON_CLOTHING_PRIMARY_RESULTS.md`, `AMAZON_CLOTHING_ABLATION_RESULTS.md`, `졸업프로젝트_교수님피드백_진행보고서 (1).md` | primary·추가 실험·교수님 보고 관점 |

Goodreads의 저장된 JSON에서 282개 집계 평균을 seed별 수치와 대조했으며, 반올림 허용 범위를 넘는 불일치는 없었다. 이 확인은 기록 내부의 정합성 점검이며, 학습 재현이나 데이터 누수 부재를 보증하는 검증은 아니다.
