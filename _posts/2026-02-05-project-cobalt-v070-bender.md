---
layout: post
title: "Project Cobalt v0.7.0: Bender"
date: 2026-02-05
author: Beyond Identity
---

Bender ships the complete authentication and authorization stack for the Cobalt control plane. Every API call now requires cryptographic proof of caller identity. Authorization decisions run through auditable Cedar policies. Identity lifecycle management gives you instant incident response. And every security event flows to your SIEM.

## Every API Call Proves Identity

All API traffic now requires cryptographic proof of possession. Each request is signed with a hardware-bound Ed25519 key tied to the specific HTTP method, URI, and timestamp. Stolen tokens are useless without the private key. Replayed requests are rejected automatically. There is no fallback to unauthenticated access.

This replaces bearer tokens entirely. Your security team no longer needs to worry about token theft leading to API compromise. The caller proves they hold the key on every request.

## Enrollment Resists Key Substitution

Operator and DPU enrollment now use challenge-response protocols. The server issues a cryptographic challenge; the caller proves key possession by signing it. Invite codes are single-use and time-limited. DPU enrollment adds a hardware verification step: the server validates the device's hardware serial against its claim token before accepting enrollment.

The first administrator bootstrap locks automatically after initial setup and persists across restarts. No configuration drift. No accidental re-enrollment.

## Authorization Gates on Attestation

Authorization runs through Cedar policies with deny-by-default enforcement. Three roles govern access: operators get explicit per-resource grants, tenant admins operate within their tenant boundary, and super admins have cross-tenant access for break-glass scenarios.

The critical differentiator: credential distribution requires verified DPU attestation. If a node's attestation is stale or unavailable, credential operations block. If attestation has failed, operations block permanently with no bypass path. A super admin can force-bypass for stale attestation, but that action generates a security warning in the audit log. There is no silent override.

This is the enforcement layer that separates Cobalt from conventional identity systems. Credentials don't reach infrastructure that can't prove it's in a known-good state.

## Incident Response in Seconds, Not Hours

When something goes wrong, you need to act immediately. Bender gives you three response tools:

**Suspend an operator.** Reversible. Blocks all their keys instantly. Use this when you're investigating and need to lock someone out without destroying their enrollment. Lift the suspension when the investigation clears them.

**Revoke a key.** Permanent. The key is terminal. No reactivation path. Use this when a key is confirmed compromised. Self-revocation is prevented, and last-key protection stops you from accidentally locking yourself out of the system.

**Decommission a DPU.** Blocks authentication and scrubs associated credentials from the control plane. The audit record is retained. If the device is later cleared, a super admin can re-activate it, which resets to pending status and opens a fresh enrollment window.

All three take effect on the next API call. No propagation delay. No cache to expire.

## Audit Logs in Your SIEM

Security events now write to syslog in RFC 5424 structured format. Grafana Loki, Splunk, and any standards-compliant SIEM can ingest them directly. Auth success and failure, enrollment events, lifecycle actions, and attestation bypasses all generate structured events with caller identity, client IP, request path, and latency.

This runs alongside the existing local audit storage used for CLI queries, so you get both operational queryability and SIEM integration. A regression test validates that private keys and credentials never appear in log output.

## What Changed for Your Security Posture

With Bender, the full security stack is in place. Every identity is challenge-response enrolled. Every API call is cryptographically signed. Every authorization decision is policy-driven with attestation gating. Every security event is auditable.

If you're evaluating hardware-enforced credential management for GPU infrastructure, this is the release to start with.

## Get Started

```bash
brew install gobeyondidentity/cobalt/bluectl
```

Also available via Docker and Linux packages. Source and full changelog on [GitHub](https://github.com/gobeyondidentity/cobalt).

---

*Project Cobalt v0.7.0 "Bender" - February 2026*
