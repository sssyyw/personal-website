+++
date = '2026-05-17T06:51:14-04:00'
draft = false
title = 'Sequence Modelling'
+++

# Sequence Modeling in Industrial Recommendation & Ads Systems

A synthesis of six engineering blog posts from **Uber**, **Pinterest**, and **Meta** describing how the biggest consumer platforms have moved from hand-crafted aggregate features and DLRMs toward transformer-based sequence modeling for recommendations and ads.

---

## 1. Why sequence modeling? The shared motivation

Every blog post starts from the same diagnosis: classical Deep Learning Recommendation Models (DLRMs) and statistics-based features have hit a ceiling.

The recurring complaints are:

- **Loss of temporal order.** Aggregating "clicks in the last 30 days" into a count or a bag-of-IDs throws away *when* and *in what order* things happened. A user who searched "running shoes" → "marathon training plan" → "energy gels" is in a very different state than someone who looked at the same three items in reverse.
- **Loss of granularity.** Co-occurrence within a single event (e.g. *clicked this product, on this surface, at this hour, after this query*) collapses into separate aggregated features.
- **Reliance on human intuition.** Tens of thousands of hand-engineered features create overlap, compute waste, and a maintenance burden — and still can't surface non-intuitive interactions.
- **Stale features.** Batch-computed signals are 24+ hours behind the user's actual session, useless for cold-start or rapidly-shifting intent.

Meta frames this as a "paradigm shift": move from **human-engineered sparse features → event-based learning directly from sequences of user events**. Uber, Pinterest, and Meta converge on the same answer despite different product surfaces (food, pins, ads).

---

## 2. The common architectural recipe

Across all six posts, you can see a shared template emerge:

**User behavior → event tokens → transformer encoder → user embedding → matched against candidate item/ad.**

The pieces, with the variations each company introduces:

### 2.1 Event representation

Each user action becomes a "token" — analogous to a word in an LLM, but with a much larger and more heterogeneous vocabulary.

- **Uber Eats homefeed:** chronological log of clicks and orders, each with store ID, cuisine, time-of-day.
- **Uber Ads:** store UUID, cuisine type, local hour, local day-of-week, engagement type (click / add-to-cart / order). Categorical IDs go through **multi-hash embeddings** — multiple independent hash functions map high-cardinality IDs into smaller embedding spaces, then combine. This is the key trick for scaling embeddings without keeping a parameter per unique ID.
- **Pinterest:** offsite conversion events (checkout, add-to-cart, signup) for the conversion CG; engagement events for the broader retrieval models.
- **Meta:** generalizes this into **Event-Based Features (EBFs)**, formalized along three dimensions:
  1. *Event stream* (which behavior — ads engaged, pages liked, etc.),
  2. *Sequence length* (how far back to look, tuned per stream),
  3. *Event information* (attributes + timestamp encoding).

Meta is explicit that EBFs *replace* legacy sparse features as the main input — not augment them.

### 2.2 Target-aware attention

A critical design choice nearly every team makes: don't encode the user sequence in isolation. **Append the candidate item (the thing you're scoring) as a query token** so the transformer can compute attention between the candidate and each past event.

- Uber Eats explicitly cites **DIN** and **BST** as inspiration: the target store is appended to the sequence before self-attention.
- Uber Ads uses the candidate ad as the **query** into a target-aware transformer encoder.
- Pinterest uses the same idea via the two-tower setup, with the candidate item or advertiser scored against a sequence-derived user embedding.

The pattern matters because it forces the model to ask "which parts of this user's history are relevant *to this specific candidate*?" rather than collapsing history into one generic vector.

### 2.3 Scaling attention to long sequences

Self-attention is O(N²), which is fine for chat messages but punishing when N is hundreds or thousands of user events. Each company has a different optimization:

- **Uber Ads → Multi-Head Latent Attention (MLA).** Borrowed from the DeepSeek paper. Instead of every event attending to every other event, a fixed set of *L* learnable latent tokens act as bottleneck intermediaries. Attention runs in two stages: tokens → latents (compress), then latents → tokens (broadcast). Drops complexity from O(N²) to O(N×L) where L ≪ N. The latent bottleneck also forces the model to distill dominant signals (cuisines, recency, frequency) into compact representations.
- **Meta → Multi-headed attention pooling.** Self-attention complexity goes from O(N×N) to O(M×N), where M is a tunable number of output embeddings keyed by the ad being ranked.
- **Meta → Jagged tensors + Jagged Flash Attention.** Different users have different sequence lengths. Padding everything to the max is wasteful. Meta built native PyTorch jagged tensor support, kernel-level GPU optimizations, and a Jagged Flash Attention module so they can run Flash Attention over variable-length sequences without padding overhead.

### 2.4 Two-tower vs. transformer-trunk

There are two camps:

| Approach | Used by | Tradeoff |
|---|---|---|
| **Two-tower** (user tower + item tower, dot product matching) | Pinterest (all three posts), Uber Eats early hybrid | Cheap retrieval via ANN; no user-item cross features at this stage |
| **Transformer-as-trunk** (everything funnels through the transformer) | Uber Eats GenRec, Meta | More expressive; supports listwise scoring and target-aware cross features |

Uber Eats explicitly describes the migration arc: started with **DLRM/DCNv2 + transformer encoder as parallel paths**, then unified into a **transformer-centered architecture** where traditional features become just more tokens fed into the transformer.

---

## 3. Beyond the basic recipe — company-specific innovations

### 3.1 Uber Eats: from pointwise to listwise (GenRec)

The most ambitious architectural shift in the set. Traditional ranking is **pointwise**: one forward pass per (user, store) pair. Uber Eats moved to **listwise parallelism** — the model takes an array of candidate stores in the same session as input and generates scores for the entire list in a single forward pass.

Why this matters: complexity per store drops to roughly **1/T** of the original (T = number of target stores). Training and serving both get dramatically cheaper, which is what makes scaling the model size feasible.

This is a step toward fully **Generative Recommender (GenRec)** architectures — the model isn't just classifying each (user, item) pair, it's generating a ranked sequence.

### 3.2 Uber Ads: Hetero-MMoE (heterogeneous experts)

Sequence modeling solved the *input* side. The *output* side — multi-task learning across pCTR, pCTO, etc. — got the **Hetero-MMoE** treatment. Traditional MMoE uses MLP experts in a multi-gate mixture; Uber found MLPs alone can't capture rich enough feature interactions. So they mixed expert types:

- **MLP experts** — implicit, deep feature interactions
- **DCN experts** (Deep & Cross Network) — explicit low-to-mid-order feature crosses
- **CIN experts** (Compressed Interaction Network from xDeepFM, in a 2D variant) — explicit high-order vector-wise crosses

Result: +0.93% AUC on pCTR, +0.66% AUC on pCTO. Interestingly inspired by a 2025 paper on Heterogeneous Mixture of Experts for Multi-Task Learning.

### 3.3 Meta: scaling laws for sequence learning

Meta's contribution is less about a single architectural trick and more about treating sequence modeling as a *first-class infrastructure problem*. They redesigned everything — data storage, feature input formats, model architecture, serving optimization — around event sequences. They explicitly draw the analogy to language modeling: EBFs are tokens, but with a vocabulary "many orders of magnitude larger than a natural language."

Their scaling agenda is bold:
- **100× longer sequences** is on the roadmap.
- Investigating **linear attention and state space models** (Mamba-style) for sequences too long for quadratic attention.
- **KV cache optimization** borrowed straight from LLM serving.
- **Multimodal enrichment** of event sequences using customized vector quantization on content embeddings.

The reported impact: 2–4% more conversions on select segments after launch.

### 3.4 Pinterest: handling sparsity, noise, and contextual signals

Pinterest's three posts are essentially one continuous story about ads candidate generation:

1. **Behavioral sequence modeling (Jan 2026).** Transformer-based two-tower model trained on offsite conversion sequences. Built for Next-Advertiser prediction first, then extended to item-level prediction. Sequence length capped at 100 — they tried 1024 and found recall gains diminished after 100, and longer sequences risked introducing stale noise from sparse conversion history.

2. **Shopping conversion CG (Apr 2026).** Conversion data is *sparse, noisy, and delayed* (advertisers report offsite events). Solutions:
   - **Multi-surface model** (Homefeed, Related Pins, Search) to avoid fragmenting the sparse labels.
   - **Dual positive signals**: supplement conversions with clicks, but use a **log-based re-weighting** of clicks based on click duration to suppress noisy short-dwell clicks.
   - **Harder negatives**: ad impressions with no engagement, not just in-batch negatives.
   - **Parallel DCN v2 + MLP** architecture (instead of stacked) — both branches see the original input, eliminating the information bottleneck. +11% recall@1000.
   - **Unified multi-task head** (instead of separate conversion and engagement heads) so serving embeddings benefit from joint optimization.
   - **Advertiser-level loss** as an auxiliary objective because Pin-level conversion data is high-variance.

3. **Contextual sequential model (May 2026).** The earlier model encoded users *offline* — no knowledge of what they were currently browsing. On Related Pins, that meant <1% of impressions were attributed to the sequential CG. Fix:
   - Add a **context layer** to the query tower that ingests features from the *subject Pin* the user is currently viewing.
   - Train with **synthetic augmented context** — inject pseudo-context derived from the positive label during training (real online context isn't available offline). High dropout on the context layer prevents over-reliance.
   - **Hybrid inference**: transformer encoder runs offline (daily refresh), context layer + final MLP runs online at request time using the cached offline embedding + real-time context.
   - Result: 3×–10× Recall@K improvement, ~275–300% lift in median candidate relevance, ~0.7% ROAS lift overall (1.4% in top markets).

The Pinterest story is the clearest example of an evolution: sequence model → conversion-specific tuning → real-time context. Each post explicitly builds on the last.

---

## 4. Infrastructure & serving — what the engineering teams actually had to build

This is the part that gets glossed over in academic papers but is the bulk of the work. Themes:

### 4.1 Near-real-time features

The shift from batch to streaming features is universal:

- **Uber's Next Personalization Platform** maintains **UserContext**: a near-real-time, cross-line-of-business, event-sourced history of user actions. Features are computed by pure-Java `FeatureExtractors` invoked by the online Feature Store at inference time. The exact same `FeatureExtractor` code is invoked by an Apache Spark job over historical `UserContext` snapshots to generate training data — eliminating training-serving skew. Data lag went from days to seconds.
- **Pinterest's hybrid inference** splits the user tower between offline (sequence encoder, refreshed daily) and online (context layer + MLP head, computed per request).

### 4.2 Training-serving parity

A recurring engineering theme: ensure the features your model sees at training time *exactly* match what it sees at serving time. Uber does this by running the same Java `FeatureExtractor` code in both paths, plus continuous monitoring via sampled feature logging. Pinterest's hybrid serving design is similarly motivated.

### 4.3 GPU efficiency

- **Uber Eats** migrated from Keras/TensorFlow to PyTorch v2; introduced **multi-hash embeddings** and **BF16 mixed-precision training**. Moved ONNX→TensorRT conversion offline to eliminate 10–60s cold-start delays. Introduced **CPU/GPU disaggregation** — CPU nodes handle feature preprocessing, GPU nodes do only model inference. Double-digit throughput improvement per node.
- **Meta** built **jagged tensor support** end-to-end (PyTorch, custom GPU kernels, Jagged Flash Attention) so variable-length sequences don't waste compute on padding.

### 4.4 Embedding scalability

The catalog is huge — Pinterest has 1B+ items, Uber has millions of merchants and dish variations. Standard embedding tables (one vector per ID) don't scale. Two approaches appear:

- **Multi-hash embeddings** (Uber): multiple hash functions map IDs into small embedding tables, embeddings are combined.
- **Drop high-cardinality features** (Pinterest item model): only keep coarse-grained IDs like advertiser ID, push through a hash embedding layer. Forces the model to learn higher-level affinities rather than memorizing individual items.

---

## 5. Loss functions & training tricks worth noting

- **Sampled softmax with log-Q correction** (Pinterest): standard for two-tower retrieval. Pinterest found they had to **carefully tune the log-Q weights for positives and negatives separately** to avoid the model collapsing onto a few popular advertisers.
- **Diversity metric as a tuning signal** (Pinterest): "fraction of random indices covered by 90% of top retrieved results" — used alongside recall to balance personalization vs. popularity.
- **All-Action Loss vs. Dense-All-Action Loss** (Pinterest): Pinterest found All-Action Loss worked better than the Dense variant from the PinnerFormer paper for offsite conversion data, because event ordering matters more when conversions are sparse.
- **Click duration re-weighting** (Pinterest): log-based weight `w` on clicks based on dwell time, capped by tunable `t_max`, to dampen the noise of accidental short clicks when using clicks as auxiliary positives for conversion prediction.

---

## 6. The common roadmap forward

Striking how aligned the "next steps" sections are across all six posts:

1. **Longer sequences / lifetime histories** — everyone wants to model months or years of behavior, not weeks. Meta talks about 100× longer. Uber talks about "account life sequence learning."
2. **Real-time sequence updates** — Pinterest, Uber Ads, and Meta all want the user representation refreshed *per event*, not per day.
3. **Multimodal enrichment** — text, image, semantic embeddings as part of each event token.
4. **Cross-domain / cross-surface learning** — combining onsite + offsite, organic + ads, search + browse sequences.
5. **Generative ranking** — moving from pointwise scoring to listwise / generative output, with Uber Eats furthest along this path.
6. **Advanced fusion** of context and sequence — Pinterest specifically calls out moving from concatenation to **cross-attention fusion** where context is the query and the encoded sequence is key/value.

---

## 7. TL;DR — the architectural pattern in one paragraph

User actions are treated as tokens in a sequence — like words in a language model — each event carrying ID, category, timestamp, and contextual attributes. A transformer encoder (often with target-aware attention so the candidate item participates as a query) digests this sequence into a user representation. Efficiency tricks like multi-hash embeddings, latent-attention bottlenecks, jagged tensors, and CPU/GPU disaggregation make it production-feasible. Two-tower variants enable cheap ANN retrieval, while transformer-trunk variants enable listwise and generative ranking. Real-time features cut training-serving skew and serve the freshest possible user state at inference time. The destination, across all three companies, is the same: a unified, scalable, intent-aware sequence model that subsumes the patchwork of hand-engineered features that came before.

---

## References

**Uber**

1. Chen, Y., Chen, P., Patel, N., Suresh, S., & Ling, B. (2026, April 16). *Next-Gen Restaurant Recommendation with Generative Modeling and Real-Time Features*. Uber Engineering Blog. https://www.uber.com/ca/en/blog/next-gen-restaurant-recommendation/

2. Estrada, D., & Lin, L. (2026, March 10). *Transforming Ads Personalization with Sequential Modeling and Hetero-MMoE at Uber*. Uber Engineering Blog. https://www.uber.com/ca/en/blog/transforming-ads-personalization/

**Pinterest**

3. Huang, R., Liu, Y., Guo, Z., Mao, A., & Ge, S. (2026, April). *From Clicks to Conversions: Architecting Shopping Conversion Candidate Generation at Pinterest*. Pinterest Engineering Blog. https://medium.com/pinterest-engineering/from-clicks-to-conversions-architecting-shopping-conversion-candidate-generation-at-pinterest-04cae5e1455b

4. Xin, H., Manoharan, L., Jayasurya, K., Guo, Z., & Liviniuk, A. (2026, May 8). *Enhancing Ad Relevance: Integrating Real-Time Context into Sequential Recommender Models*. Pinterest Engineering Blog. https://medium.com/pinterest-engineering/enhancing-ad-relevance-integrating-real-time-context-into-sequential-recommender-models-bc3a2f9b682e

5. Manoharan, L., Jayasurya, K., Guo, Z., Xin, J., & Liviniuk, A. (2026, January 28). *Ads Candidate Generation using Behavioral Sequence Modeling*. Pinterest Engineering Blog. https://medium.com/pinterest-engineering/ads-candidate-generation-using-behavioral-sequence-modeling-f9077ee1325d

**Meta**

6. Reddy, S., Beg, H., Overwijk, A., & O'Byrne, S. (2024, November 19). *Sequence learning: A paradigm shift for personalized ads recommendations*. Engineering at Meta. https://engineering.fb.com/2024/11/19/data-infrastructure/sequence-learning-personalized-ads-recommendations/

**Cited foundational papers**

7. Wang, R., et al. (2021). *DCN V2: Improved Deep & Cross Network and Practical Lessons for Web-scale Learning to Rank Systems*. WWW '21. https://arxiv.org/pdf/2008.13535

8. Hamilton, W. L., et al. (2017). *Inductive Representation Learning on Large Graphs* (GraphSAGE). NIPS. https://arxiv.org/pdf/1706.02216

9. DeepSeek-AI. (2024). *DeepSeek-V2: A Strong, Economical, and Efficient Mixture-of-Experts Language Model* (Multi-Head Latent Attention). https://arxiv.org/pdf/2405.04434

10. *Heterogeneous Mixture of Experts for Multi-Task Learning* (2025). https://arxiv.org/pdf/2505.17925

11. Pancha, N., et al. (2022). *PinnerFormer: Sequence Modeling for User Representation at Pinterest*. https://arxiv.org/abs/2205.04507

12. Zhou, G., et al. (2018). *Deep Interest Network for Click-Through Rate Prediction* (DIN). KDD.

13. Chen, Q., et al. (2019). *Behavior Sequence Transformer for E-commerce Recommendation in Alibaba* (BST).

14. *Jagged Flash Attention*. (2024). ACM. https://dl.acm.org/doi/10.1145/3640457.3688040