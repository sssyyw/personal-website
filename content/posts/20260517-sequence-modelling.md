+++
date = '2026-05-17T06:51:14-04:00'
draft = true
title = '20260517 Sequence Modelling'
+++

Uber:
https://www.uber.com/ca/en/blog/next-gen-restaurant-recommendation/?uclick_id=a5e267bc-8ea1-499b-b2be-c77705e5f27f
https://www.uber.com/ca/en/blog/transforming-ads-personalization/?uclick_id=a5e267bc-8ea1-499b-b2be-c77705e5f27f

for recommendation system, sequence modeling to generate feature (together with traditional feature) is a common practice. For example, User uses UserContext, a near real-time, cross-line-of-business, event-sourced history of user actions.

the transformer encoder becomes the primary trunk of the model. Traditional non-sequence features are no longer processed in a separate DLRM path; instead, they are transformed and concatenated to target token features before entering the transformer, serving as enriched feature representation for the sequence module.

Pinterest:
https://medium.com/pinterest-engineering/from-clicks-to-conversions-architecting-shopping-conversion-candidate-generation-at-pinterest-04cae5e1455b
https://medium.com/pinterest-engineering/enhancing-ad-relevance-integrating-real-time-context-into-sequential-recommender-models-bc3a2f9b682e
https://medium.com/pinterest-engineering/ads-candidate-generation-using-behavioral-sequence-modeling-f9077ee1325d

Meta:
https://engineering.fb.com/2024/11/19/data-infrastructure/sequence-learning-personalized-ads-recommendations/