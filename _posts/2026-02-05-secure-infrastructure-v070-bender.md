---
layout: post
title: "Secure Infrastructure v0.7.0: Bender"
date: 2026-02-05
author: Beyond Identity
---

Project Cobalt's second minor release ships the entire Nexus API Authentication epic: five phases that transform the control plane from unauthenticated HTTP endpoints into a fully attestation-gated, policy-driven system. Every API call now proves caller identity through hardware-bound cryptography, every authorization decision runs through auditable Cedar policies, and every security event flows to your SIEM.

## DPoP Authentication (Phase 1)

All nexus API traffic now requires DPoP (Demonstrating Proof of Possession, RFC 9449) authentication. Each request carries an Ed25519-signed JWT that binds the caller's private key to the specific HTTP method, URI, and timestamp. The server runs a 10-step validation pipeline: parse the proof, verify the typ/alg headers, look up the public key by `kid`, verify the Ed25519 signature, check freshness within a 60-second window, confirm method and URI binding, reject replayed `jti` values, and verify the caller's identity status. Stolen tokens are useless without the signing key. Replayed proofs are caught by the jti cache. There is no fallback to unauthenticated access.

## Challenge-Response Enrollment (Phase 2)

Identity establishment uses a two-phase challenge-response protocol that prevents key substitution attacks. On init, the server issues a 32-byte cryptographic challenge with a 5-minute TTL. On complete, the caller proves private key possession by signing that challenge. Invite codes are SHA-256 hashed at rest, single-use, and time-limited to one hour. DPU enrollment adds DICE chain verification: the server validates that the device's hardware serial matches the claim token before accepting enrollment. The bootstrap flow for the first administrator locks automatically after the first successful enrollment and persists across server restarts.

## Cedar Authorization with Attestation Gating (Phase 3)

Authorization decisions run through AWS Cedar, a policy language that produces auditable, declarative access control. Three roles govern the system: `operator` (scoped to explicit per-resource grants), `tenant:admin` (all resources within their tenant), and `super:admin` (global cross-tenant access). Deny-by-default means unknown actions fail closed.

The attestation gate is the security boundary that distinguishes Cobalt from conventional identity systems. Credential distribution requires verified DPU attestation. If attestation is stale or unavailable, the operation is blocked. If attestation has failed, the operation is blocked permanently with no bypass path. For stale/unavailable states, a `super:admin` can force-bypass with a mandatory reason header that generates a SECURITY_WARNING audit event. The staleness threshold is configurable via `--attestation-stale-after`.

## Lifecycle Management (Phase 4)

Identities now have proper lifecycles with an immediate-effect guarantee: status is checked on every request via a direct database read in DPoP validation step 10. No caching, no propagation delay.

**Operator suspension** is reversible and blocks all KeyMakers owned by the operator instantly. Useful for incident response where you need to lock someone out while investigating without destroying their enrollment.

**KeyMaker and AdminKey revocation** is permanent and irrecoverable. Once revoked, the key is terminal. Self-revocation is prevented.

**DPU decommissioning** blocks authentication and scrubs associated credentials from nexus. The record is retained for audit. A `super:admin` can re-activate a decommissioned DPU, which resets it to pending status and opens a fresh enrollment window.

Last-key protection prevents accidentally revoking the only remaining admin key, which would lock you out of the system entirely.

## RFC 5424 Audit Logging (Phase 5)

Security events now write to the local syslog daemon in RFC 5424 structured data format, parseable by Grafana Loki, Splunk, and any standards-compliant SIEM. Auth success/failure, enrollment, lifecycle actions, and attestation bypasses all generate structured events with `kid`, client IP, request path, and latency. This runs alongside the existing SQLite audit storage (used by `bluectl audit` queries), so you get both local queryability and SIEM integration.

The syslog writer carries zero external dependencies. Socket reconnection uses exponential backoff. A no-secrets regression test validates that private keys, DPoP proofs, and invite codes never appear in log output.

## What This Means

Bender v0.7.0 closes the gap between "working prototype" and "system you can put in front of a security team." The entire authentication and authorization stack is now enforced, auditable, and built on cryptographic proof rather than bearer tokens. If you are evaluating hardware-enforced credential management for GPU infrastructure, this is the release to start with.

Bender is available now via Homebrew (`brew install gobeyondidentity/cobalt/bluectl`), Docker, and Linux packages. Source and full changelog on [GitHub](https://github.com/gobeyondidentity/cobalt).
