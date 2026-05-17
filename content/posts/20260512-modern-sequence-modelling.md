+++
date = '2026-05-12T08:21:39-04:00'
draft = false
title = 'Two-Tower, DCN v2, and Transformers: How Modern Retrieval and Ranking Fit Together'
tags = ['recommendation system', 'DCN v2', 'transformer']
+++


If you've spent any time around modern recommendation, search, or ads systems, you've run into three architectures that keep showing up: **two-tower models**, **DCN v2**, and **Transformers**. They're often discussed as if they're alternatives, but in production they're almost always *composed*. Each one solves a different problem, and the interesting design work is in how you fit them together.

This post walks through what each does, where they slot in, and how a typical large-scale retrieval-and-ranking stack actually uses all three.

## The two-tower model: separability for scale

Two-tower (also called dual encoder) is fundamentally a **retrieval** architecture. The setup:

- A **user/query tower** encodes user-side features into a vector `u`.
- An **item tower** encodes item-side features into a vector `v`.
- Score is a simple dot product: `s(u, v) = u · v`.

The whole point is **separability**. Because the two towers never see each other until the final dot product, you can:

1. Precompute every item embedding offline.
2. Store them in an ANN index (FAISS, ScaNN, OpenSearch).
3. At serve time, encode only the user and run a millisecond-latency nearest-neighbor lookup.

That's what makes two-tower scale to catalogs of millions or billions of items.

The price you pay: **no cross-feature interaction between user and item until the very end.** A user feature can't attend to an item feature, because then you couldn't precompute item vectors anymore. Two-tower deliberately gives up expressiveness to buy sub-linear retrieval.

## DCN v2: learning feature crosses

DCN v2 (Deep & Cross Network V2, Google 2020) is the modern successor to Wide & Deep. Where Wide & Deep required humans to hand-pick which feature crosses mattered, DCN v2's **Cross Network** learns them automatically — capturing polynomial feature interactions of bounded degree with a stack of cross layers. A parallel deep MLP handles implicit high-order interactions.

The v2 improvement over v1 is a **full-matrix cross layer** (rather than the rank-1 vector in v1), with an optional **low-rank / mixture-of-experts** decomposition to control parameter cost in production.

DCN v2's sweet spot is **tabular sparse features** — the kind of feature soup typical in ads and recsys: categorical IDs, embeddings, counts, contexts. It's parameter-efficient and tends to outperform plain MLPs when explicit feature crosses matter.

## Transformers: sequences and text

Transformers slot into this world in two distinct ways.

**Sequence encoder within a tower.** User behavior is often a sequence — last N items clicked, queries issued, sessions browsed. A Transformer encodes that sequence into a single user vector. This is the architecture behind BERT4Rec (bidirectional, masked-item prediction), SASRec (causal, GPT-style), Pinterest's PinnerFormer, Meta's HSTU, and recent industrial-scale retrieval models. The item tower is often lighter — items typically have static attributes rather than sequences.

**Both towers as Transformers (dense text retrieval).** When the features themselves are text, both towers are usually pretrained Transformers. Query → BERT/Sentence-BERT → `u`; document → same model → `v`; trained contrastively. This is DPR, E5, BGE, and the embedding models behind most vector-search systems today.

A clever middle-ground worth knowing about is **ColBERT**, which keeps per-token embeddings and scores via `MaxSim` over token pairs — late interaction, more expressive than dot product, still indexable with tricks.

## How they compose in practice

Here's where it gets interesting. Most large production systems use a **two-stage pipeline**:

```
                       ┌─────────────────────────┐
                       │   Retrieval (two-tower) │
                       │   - millions of items   │
                       │   - ANN lookup → top-K  │
                       └────────────┬────────────┘
                                    │
                                    ▼ (top-K, e.g. K=500)
                       ┌─────────────────────────┐
                       │   Ranking (DCN v2)      │
                       │   - full feature crosses│
                       │   - re-score K items    │
                       └─────────────────────────┘
```

- **Stage 1 (retrieval)**: Two-tower fetches the top-K candidates cheaply via ANN. Each tower internally can use a Transformer (for sequences/text) and/or DCN v2 (for tabular crosses) — but the towers stay separable so the index works.
- **Stage 2 (ranking)**: DCN v2 (or another heavy model) re-scores the K candidates with **full cross-feature interactions between user and item** — the thing two-tower structurally couldn't do. Now you only have 500 items, so you can afford the expressiveness.

Within a single tower, Transformer and DCN v2 often coexist:

```
sequence features  → Transformer → pooled vector ┐
                                                  ├─ concat → DCN v2 → u
tabular features   → embeddings  ─────────────────┘
```

Transformer handles the sequence; DCN v2 handles crosses among everything else.

## Transformer vs. DCN v2, side by side

| | Transformer | DCN v2 |
|---|---|---|
| Best at | Sequential/textual features, long context | Tabular sparse features, explicit crosses |
| Inductive bias | Attention over tokens/items | Polynomial feature crosses |
| Typical input | Sequence of IDs or tokens | Concatenated feature embeddings |

They're not competitors. They're complementary tools with different inductive biases, and large systems use both — often in the same tower.

## Practical details that trip people up

A few things worth flagging if you're actually building one of these:

- **Contrastive loss with in-batch negatives** is the standard training objective for two-tower. Sampled softmax and BPR are common too.
- **Asymmetric towers are fine.** The user tower can be a heavy Transformer over long history; the item tower can be a small MLP. They just need to land in the same vector space.
- **Pooling matters more than expected.** CLS, mean, attention-pooling, last-token — these give meaningfully different retrieval quality and are worth tuning.
- **Item embedding refresh cadence.** If items are Transformer-encoded, you need a pipeline to re-encode and re-index on model updates. Usually nightly batch.

## The mental model

The cleanest way to think about this stack:

- **Two-tower** is the structure that makes retrieval scalable. It's a constraint, not a model — "no cross-feature interaction until the dot product."
- **Transformers** are how you encode sequences and text inside a tower (or as both towers, for pure dense retrieval).
- **DCN v2** is how you encode feature crosses — either inside a tower for tabular features, or as the ranker on top of retrieval.

Any architecture that lets user and item features cross early is more accurate but breaks ANN indexability. Two-tower gives that up for speed; DCN v2 in the ranking stage gives it back where you can afford it. Transformers slot in wherever sequence or text structure needs to be encoded.

That's the whole pattern. Once you see it, every "what's the difference between X and Y" question in this space resolves into "which slot does it fill?"

