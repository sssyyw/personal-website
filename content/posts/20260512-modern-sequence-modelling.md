+++
date = '2026-05-12T08:21:39-04:00'
draft = true
title = 'Sequence Modelling for recommendation system'
tags = ['recommendation system', 'sequence modelling', 'transformer']
+++

for recommendation system, sequence modeling to generate feature (together with traditional feature) is a common practice. For example, User uses UserContext, a near real-time, cross-line-of-business, event-sourced history of user actions.

the transformer encoder becomes the primary trunk of the model. Traditional non-sequence features are no longer processed in a separate DLRM path; instead, they are transformed and concatenated to target token features before entering the transformer, serving as enriched feature representation for the sequence module.

