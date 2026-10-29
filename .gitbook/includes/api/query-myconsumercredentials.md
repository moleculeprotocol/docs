---
title: query-myconsumercredentials
---

Personal API credentials the caller has issued and not revoked, newest first. Requires the caller's session and fails with `UNAUTHENTICATED` without one. Expired credentials no longer count against the issuance limit and stay listed until revoked or until the caller issues another.

Returns [`[ConsumerCredential!]`](../../../api-reference/types.md#consumercredential).

This operation takes no arguments.
