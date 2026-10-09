# Intent-Aware Recommendation with LLMs and Knowledge Graphs

**Hanyang University · Data Science Graduation Project · 2026**  
**Team:** 김진욱 (Jinwook Kim), 유은선 (Eunseon Yoo) · **Advisor:** 김현준

> **Research question:** Under which data conditions and integration strategies can LLM-derived intent, item metadata, and recent interaction history improve or complement collaborative filtering?

This repository is the **documentation and results hub** for our graduation project. It contains our proposal, progress reports, and experiment summaries. Implementations and dataset-specific experiments are maintained in the [linked code repositories](#code-repositories).

## Overview

Conventional recommender systems can capture collaborative signals effectively, but user preferences can also depend on *why* an item was consumed, its attributes, and how interests change over time. We study whether such signals improve top-*K* recommendation when combined with collaborative filtering (CF).

Our research progressed from a hypothesis about **category switching with attribute retention** (e.g., changing product categories while retaining a preferred brand) to a **cross-domain empirical evaluation** of intent-aware and knowledge-graph-based recommendation methods.

- **Domains:** Goodreads Children (books), MovieLens 25M (movies), and Amazon Clothing, Shoes and Jewelry (e-commerce).
- **Core signals:** LLM-extracted user/item intent, item metadata, knowledge-graph relationships, and recency-weighted interaction history.
- **Comparisons:** BPR, LightGCN, a KG-disabled matrix-factorization ablation, KG augmentation, score-level late fusion, candidate retrieval, and reranking.
- **Evaluation:** Temporal splits, three-seed experiments, NDCG@10, Recall@10, Tail Recall@10, and catalog Coverage@10.

**Scope clarification:** The original report title refers to *generative retrieval*, but the evaluated system currently focuses on LLM-based **intent extraction and KG/CF integration**. Autoregressive generation of item identifiers has **not** been implemented. The approaches below are **inspired by**, rather than exact reproductions of, their cited frameworks.

## Research Approach

```text
User interactions, reviews, and item metadata
                  |
                  v
        LLM-based intent extraction
          + related-intent expansion
                  |
                  v
       Intent / metadata knowledge graph
                  |
                  +---- Recency-aware user representation
                  |
                  +---- CF and KG-based scoring
                  |
                  v
     Compare integration / retrieval strategies
       - Direct KG augmentation
       - Score-level late fusion
       - Graph-based candidate retrieval
       - Popularity-debiased retrieval
       - Soft graph-prior reranking
                  |
                  v
      Temporal and slice-based evaluation
```

These are **alternative configurations evaluated through ablations**, not necessarily stages of one fully deployed serial recommender.

| Research component | Implementation focus |
| --- | --- |
| **Intent-aware KG (IKGR-inspired)** | Extract semantic intents from user/item text and represent intents and metadata as graph relations. |
| **Dynamic profiles (DynLLM-inspired)** | Reweight historical interactions with a recency-based user representation; this is not a full reproduction of DynLLM's dynamic-graph architecture. |
| **Candidate selection and reranking (CORONA-inspired)** | Evaluate graph-based retrieval, popularity debiasing, and soft reranking as separate alternatives. |
| **CF integration** | Compare KG augmentation and late fusion against BPR, LightGCN, and KG-off baselines. |

## Preliminary Results (October 2026)

Representative **per-user temporal (TO)** results are reported below. NDCG@10 shows the **three-seed mean ± population standard deviation**; the other metrics show three-seed means.

| Dataset | Configuration | NDCG@10 | Tail Recall@10 | Coverage@10 |
| --- | --- | ---: | ---: | ---: |
| Goodreads Children (k=30)† | BPR | 0.0814 ± 0.0008 | 0.0110 | 0.3641 |
| Goodreads Children (k=30)† | LightGCN | 0.0820 ± 0.0006 | 0.0047 | 0.1757 |
| Goodreads Children (k=30)† | KG + recency | 0.0822 ± 0.0005 | 0.0170 | 0.5164 |
| MovieLens 25M (k=30) | LightGCN | 0.0854 ± 0.0005 | 0.0013 | 0.3366 |
| MovieLens 25M (k=30) | Late fusion | 0.0822 ± 0.0003 | 0.0033 | 0.4488 |
| Amazon Clothing | BPR | 0.1744 ± 0.0011 | 0.0442 | 0.5722 |
| Amazon Clothing | LightGCN | 0.1575 ± 0.0010 | 0.0292 | 0.2159 |
| Amazon Clothing | Late fusion | 0.1781 ± 0.0008 | 0.0484 | 0.6383 |

† **Goodreads caveat:** Historical profile construction still requires a train-only audit; these results should not be treated as fully leakage-validated. Dataset processing, sample sizes, filtering, and interaction definitions differ, so **compare models within a dataset, not absolute scores across datasets**.

### What we found

- **Goodreads:** KG + recency maintained approximately comparable ranking accuracy while improving long-tail recall and catalog coverage relative to CF baselines in the reported evaluation.
- **MovieLens:** LightGCN remained the strongest NDCG@10 baseline. Late fusion broadened catalog exposure but did not exceed LightGCN on ranking accuracy.
- **Amazon Clothing:** Late fusion improved mean NDCG@10 over BPR (0.1744 → 0.1781) and broadened coverage (0.5722 → 0.6383) in the recorded experiment. Equivalent tuning-budget comparisons and significance checks remain necessary.
- **Across domains:** More intent, graph information, or retrieval complexity did **not** consistently improve accuracy. Hard candidate restriction could substantially damage ranking performance, even when certain long-tail metrics improved.

**Current conclusion:** The principal result is not a universally superior recommender, but evidence that the value of semantic and temporal augmentation is **dataset- and integration-dependent**.

## Limitations and Ongoing Work

The findings above are preliminary and should not be read as evidence of causal benefits from LLM-derived intent alone. Our next steps are to:

1. **Revalidate evaluation integrity:** Audit train-only user profiles, intent features, popularity definitions, and comparable baseline tuning budgets.
2. **Analyze user-level gains and losses:** Test whether history length, recency, category diversity, and intent specificity predict when augmentation helps.
3. **Test controlled intent weighting:** Compare intent-off, uniform weighting, IDF-style weighting, and frequent-intent removal under the same fusion architecture.
4. **Consider conditional model selection:** If validation shows predictable user-level benefits, evaluate a lightweight binary gate choosing between CF and the augmented model. **This is planned work, not a completed result.**
5. **Report failure cases:** Keep cold-start limitations, accuracy–coverage trade-offs, and negative ablations visible alongside positive results.

## Documents

| Document | Contents |
| --- | --- |
| [Project Proposal](https://github.com/HK1DS/Documents/blob/main/%5BDS_AI_PRJ%5D_Proposal_%5B2023036299%5D_%5B%EA%B9%80%EC%A7%84%EC%9A%B1%5D_%5B2023054275%5D_%5B%EC%9C%A0%EC%9D%80%EC%84%A0%5D.pdf) | Initial motivation, behavioral hypotheses, and proposed architecture. |
| [Progress Report 1](https://github.com/HK1DS/Documents/blob/main/%5BDS_AI_PRJ%5D_2_ProgressReport_1_%5B2023036299%5D_%5B%EA%B9%80%EC%A7%84%EC%9A%B1%5D_%5B2023054275%5D_%5B%EC%9C%A0%EC%9D%80%EC%84%A0%5D.pdf) | Revised cross-domain EDA, framework refinement, and the initial IKGR baseline. |
| [Progress Report 2](https://github.com/HK1DS/Documents/blob/main/%5BDS_AI_PRJ%5D_ProgressReport_2023036299_%EA%B9%80%EC%A7%84%EC%9A%B1_2023054275_%EC%9C%A0%EC%9D%80%EC%84%A0.pdf) | Cross-domain implementation, temporal evaluation, preliminary results, and next steps. |
| [Conference Report](https://github.com/HK1DS/Documents/blob/main/%5BDS_AI_PRJ%5D_6_Conference_Report_%5B2023036299%5D_%5B%EA%B9%80%EC%A7%84%EC%9A%B1%5D_%5B2023054275%5D_%5B%EC%9C%A0%EC%9D%80%EC%84%A0%5D.pdf) | Conference-related project document. |
| [Summer Experiment Summary](https://github.com/HK1DS/Documents/blob/main/summer_project_results.md) | Consolidated summer experiment records and implementation notes (September 2026 snapshot). |

## Code Repositories

| Repository | Scope |
| --- | --- |
| [graduate_eda](https://github.com/HK1DS/graduate_eda) | Initial and revised exploratory analysis of category transitions and retained attributes. |
| [Goodreads_experiment](https://github.com/HK1DS/Goodreads_experiment) | Goodreads experiments and the early IKGR / KG evaluation pipeline. |
| [hk1ds_jwkim_movielens](https://github.com/HK1DS/hk1ds_jwkim_movielens) | Reusable pipeline and MovieLens 25M experiments. |
| [hk1ds_ercnard_amzn_clothing](https://github.com/HK1DS/hk1ds_ercnard_amzn_clothing) | Amazon Clothing experiments and integration ablations. |

Consult each implementation repository for its own configuration, dataset preparation, environment, and execution instructions. This documents repository does not contain a one-command reproduction package.

## Related Research

The experimental design draws on **[IKGR](https://arxiv.org/abs/2505.10900)** (intent-augmented knowledge graph recommendation), **[DynLLM](https://arxiv.org/abs/2405.07580)** (dynamic recommendation), and **[CORONA](https://doi.org/10.1145/3726302.3729937)** (coarse-to-fine graph-based recommendation). Please refer to the progress reports for the broader related-work review and full citations.

---

*Project status: in progress (October 2026). Numbers and conclusions reflect the reported experiments and may be updated after protocol audits and final validation.*
