# QA-STP FIPS 140-3 Compatibility
# Software Test Plan

(c) 2026 Altinity Inc. All Rights Reserved.

**Author:** vzakaznikov

**Date:** May 19, 2026

## Table of Contents

* 1 [Introduction](#introduction)
* 2 [Configuration Requirements](#configuration-requirements)
* 3 [Build Verification](#build-verification)
* 4 [GODEBUG Strict Mode Smoke Test](#godebug-strict-mode-smoke-test)
* 5 [FIPS 140-3 Valid TLS Cipher Suites](#fips-140-3-valid-tls-cipher-suites)
* 6 [clickhouse-operator Connections](#clickhouse-operator-connections)
* 7 [metrics-exporter Connections](#metrics-exporter-connections)
* 8 [Self-Test and Integrity Verification](#self-test-and-integrity-verification)
* 9 [CI/CD Image and Policy Verification](#cicd-image-and-policy-verification)
* 10 [(Optional) ACVP Algorithm Validation](#optional-acvp-algorithm-validation)
* 11 [Known FIPS Gaps](#known-fips-gaps)

## Introduction

This test plan covers FIPS 140-3 compatibility testing for the **clickhouse-operator** and
**metrics-exporter** binaries built with Go FIPS support.

The goal is to verify that FIPS-enabled builds of the operator and metrics-exporter:
- Operate correctly under FIPS constraints
- Properly enforce cryptographic restrictions
- Use FIPS-compliant TLS for all inbound and outbound connections

**Boundary:** The operator and metrics-exporter run in the same pod. Internal IPC between them
is localhost HTTP and is not subject to FIPS TLS requirements.

## Configuration Requirements

Plain HTTP/TCP on any external connection is a configuration error for FIPS compliance.
TLS must be enabled for all connections to:

- Kubernetes API
- ClickHouse Server
- ZooKeeper/Keeper
- Prometheus scrape endpoints

## Build Verification

**Objective:** Verify binaries are FIPS builds and linked to Go Cryptographic Module v1.0.0.

**Certificates:**
- [CMVP #5247](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/5247)
- [CAVP A6650](https://csrc.nist.gov/projects/cryptographic-algorithm-validation-program/details?product=19371)

**Build requirement:** `GOFIPS140=v1.0.0` (or `certified`)

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Operator version | Run `clickhouse-operator --version` or check logs | Output includes FIPS indicator |
| Metrics exporter version | Run `metrics-exporter --version` or check logs | Output includes FIPS indicator |
| Build flag | Run `go version -m <binary>` | Shows `GOFIPS140=v1.0.0` |
| FIPS version | Check `crypto/fips140.Version()` | Returns `v1.0.0` |
| FIPS enabled | Check `crypto/fips140.Enabled()` | Returns `true` |

## GODEBUG Strict Mode Smoke Test

**Objective:** Verify the project test suite runs in strict FIPS mode.

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Strict mode smoke test | Run all e2e tests with `GODEBUG=fips140=only` enabled | No panic/crash and no test regressions caused by strict FIPS mode |

## FIPS 140-3 Valid TLS Cipher Suites

The following cipher suites are valid for this test plan and apply to all TLS-enabled
inbound and outbound connections for both clickhouse-operator and metrics-exporter.

### Approved TLS Cipher Suites (Used by Both Clients and Servers)

**TLS 1.2:**

| Cipher Suite | OpenSSL Name |
|--------------|--------------|
| TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 | ECDHE-RSA-AES128-GCM-SHA256 |
| TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 | ECDHE-RSA-AES256-GCM-SHA384 |
| TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 | ECDHE-ECDSA-AES128-GCM-SHA256 |
| TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 | ECDHE-ECDSA-AES256-GCM-SHA384 |
| TLS_DHE_RSA_WITH_AES_128_GCM_SHA256 | DHE-RSA-AES128-GCM-SHA256 |
| TLS_DHE_RSA_WITH_AES_256_GCM_SHA384 | DHE-RSA-AES256-GCM-SHA384 |
| TLS_RSA_WITH_AES_128_GCM_SHA256 | AES128-GCM-SHA256 |
| TLS_RSA_WITH_AES_256_GCM_SHA384 | AES256-GCM-SHA384 |

**TLS 1.3:**

| Cipher Suite | OpenSSL Name |
|--------------|--------------|
| TLS_AES_128_GCM_SHA256 | TLS_AES_128_GCM_SHA256 |
| TLS_AES_256_GCM_SHA384 | TLS_AES_256_GCM_SHA384 |
| TLS_AES_128_CCM_SHA256 | TLS_AES_128_CCM_SHA256 |
| TLS_AES_128_CCM_8_SHA256 | TLS_AES_128_CCM_8_SHA256 |

**Not valid for this test plan (must be rejected):**
- Any TLS cipher suite not explicitly listed in the approved TLS 1.2/TLS 1.3 tables above
- Protocol versions: SSLv2, SSLv3, TLS 1.0, TLS 1.1
- Cipher suites using non-approved/legacy algorithms (for this profile), including:
  - ChaCha20-Poly1305
  - RC4, RC2, DES, 3DES, IDEA, SEED, CAMELLIA, ARIA
  - NULL encryption / NULL authentication
  - Anonymous key exchange (`aNULL`, `eNULL`, `ADH`, `AECDH`)
  - Export/weak suites (`EXP`, `LOW`, `40-bit`, `56-bit`)
  - MD5- or SHA-1-based legacy suites

## clickhouse-operator Connections

**Objective:** Verify all clickhouse-operator inbound and outbound connections use FIPS-compliant TLS.

**Connection Overview:**

| Direction | Target | Protocol | Default Port | TLS Support |
|-----------|--------|----------|--------------|-------------|
| Outbound | Kubernetes API Server | HTTPS | 443 | Yes (client-go), configurable via `security.kubernetes.tls` |
| Outbound | ClickHouse Server | HTTP/HTTPS | 8123/8443 | Yes, configurable via `security.clickhouse.tls` |
| Outbound | ZooKeeper/Keeper | TCP | 2181/2281 | Yes (mTLS), configurable via `security.zookeeper.tls` |
| Outbound | metrics-exporter (IPC) | HTTP | 8888 | No (same pod, localhost) |
| Inbound | Prometheus scrape | HTTP | 9999 | **FIPS Gap** - needs TLS |

**Operator to Kubernetes API**

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Operator FIPS cipher to K8s | Operator connects with FIPS-approved cipher | Connection succeeds |
| Operator non-FIPS cipher to K8s | K8s API only offers non-approved cipher | Operator rejects connection |
| `security.kubernetes.tls.minVersion=1.2` | Enforce TLS 1.2 minimum | TLS 1.1 rejected |
| `security.kubernetes.tls.minVersion=1.3` | Enforce TLS 1.3 minimum | TLS 1.2 rejected |

**Operator to ClickHouse Server**

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Operator FIPS cipher to CH | Operator connects with FIPS-approved cipher | Connection succeeds |
| Operator non-FIPS cipher to CH | Server only offers non-approved cipher | Operator rejects connection |
| `security.clickhouse.tls.minVersion=1.2` | Enforce TLS 1.2 minimum | TLS 1.1 rejected |
| `security.clickhouse.tls.minVersion=1.3` | Enforce TLS 1.3 minimum | TLS 1.2 rejected |

**Operator to ZooKeeper/Keeper**

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Operator FIPS cipher to ZK | Operator connects with FIPS-approved cipher | Connection succeeds |
| Operator non-FIPS cipher to ZK | ZK only offers non-approved cipher | Operator rejects connection |
| `security.zookeeper.tls.minVersion=1.2` | Enforce TLS 1.2 minimum | TLS 1.1 rejected |
| `security.zookeeper.tls.minVersion=1.3` | Enforce TLS 1.3 minimum | TLS 1.2 rejected |

**Operator to metrics-exporter (IPC)**

> Same pod, localhost - HTTP acceptable. Token auth via `security.ipc.mode=Secure`.

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Operator IPC + `security.ipc.mode=Secure` | HTTP with token auth enabled | Works correctly |

**Operator Prometheus Metrics (:9999)**

> **FIPS Gap:** Currently HTTP-only. Requires TLS for FIPS compliance.

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Operator metrics FIPS cipher | Scrape with FIPS-approved cipher | Connection succeeds |
| Operator metrics non-FIPS cipher | Scrape with non-approved cipher | Connection rejected |

## metrics-exporter Connections

**Objective:** Verify all metrics-exporter inbound and outbound connections use FIPS-compliant TLS.

**Connection Overview:**

| Direction | Target | Protocol | Default Port | TLS Support |
|-----------|--------|----------|--------------|-------------|
| Outbound | Kubernetes API Server | HTTPS | 443 | Yes (client-go) |
| Outbound | ClickHouse Server | HTTP/HTTPS | 8123/8443 | Yes, inherits `security.clickhouse.tls` |
| Inbound | Prometheus scrape | HTTP | 8888 `/metrics` | **FIPS Gap** - needs TLS |
| Inbound | Operator IPC | HTTP | 8888 `/chi` | No (same pod, localhost) |

**Exporter to Kubernetes API**

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Exporter FIPS cipher to K8s | Exporter connects with FIPS-approved cipher | Connection succeeds |
| Exporter non-FIPS cipher to K8s | K8s API only offers non-approved cipher | Exporter rejects connection |

**Exporter to ClickHouse Server**

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Exporter FIPS cipher to CH | Exporter queries with FIPS-approved cipher | Connection succeeds |
| Exporter non-FIPS cipher to CH | Server only offers non-approved cipher | Exporter rejects connection |

**Exporter Prometheus Metrics (:8888/metrics)**

> **FIPS Gap:** Currently HTTP-only. Requires TLS for FIPS compliance.

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Exporter metrics FIPS cipher | Scrape with FIPS-approved cipher | Connection succeeds |
| Exporter metrics non-FIPS cipher | Scrape with non-approved cipher | Connection rejected |

**Exporter IPC Endpoint (:8888/chi)**

> Same pod, localhost - HTTP acceptable. Token auth via `security.ipc.mode=Secure`.

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Exporter IPC + `security.ipc.mode=Secure` | HTTP with token auth enabled | Works correctly |

## Self-Test and Integrity Verification

**Objective:** Verify FIPS self-test and integrity checks.

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Corrupted binary | Modify binary bytes and execute | Refuses to run, reports integrity failure |
| CAST failure | Trigger known-answer test failure | Process terminates with CAST error |
| Self-test timing | Self-test runs at process start | Completes before K8s API calls |

## CI/CD Image and Policy Verification

**Objective:** Add CI/CD jobs to validate FIPS image build, image supply-chain checks, and `security.fips.images.policy` enforcement.

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| Operator FIPS image build | Build clickhouse-operator with FIPS tags | Image builds successfully |
| Exporter FIPS image build | Build metrics-exporter with FIPS tags | Image builds successfully |
| Image vulnerability scan | Scan images with Grype | No Critical, High, or Medium vulnerabilities |
| `security.fips.images.policy=Required` | Non-FIPS image in CHI | CHI rejected with FIPSImagePolicyViolation |
| `security.fips.images.policy=Required` | FIPS-tagged image in CHI | CHI reconciles normally |
| `security.fips.images.policy=Permissive` | Any image in CHI | CHI reconciles (default behavior) |

## (Optional) ACVP Algorithm Validation

**Objective:** Reproduce ACVP expected-output checks using the same public-scope config
pattern used in [clickhouse-backup PR #1364](https://github.com/Altinity/clickhouse-backup/pull/1364).

> **Note:** ACVP tests the cryptographic library as compiled into the shipped binary.
> In Go, crypto primitives are statically linked — the bytes ACVP exercises are the exact bytes users run.
> Reference config:
> [`pkg/acvpwrapper/acvp_test_fips140v1.26.public.config.json`](https://github.com/Altinity/clickhouse-backup/blob/master/pkg/acvpwrapper/acvp_test_fips140v1.26.public.config.json)
> (public-API scope; excludes ML-KEM/ML-DSA).

| Test Assertion | Description | Expected Result |
|-----------|-------------|-----------------|
| ACVP wrapper integration | Add `acvp` subcommand to operator/exporter | ACVP subcommand responds |
| ACVP config generation | Run `<binary> acvp getConfig` | Returns supported capabilities |
| ACVP expected-output replay | Run pinned ACVP replay against tracked config | All configured suites match expected output |
| ACVP suite count | Validate configured suite count from tracked config | `38 ACVP tests matched expectations` |

Covered suite families from the tracked config (38 total):
- SHA-2 (6), SHA-3 (4), SHAKE/cSHAKE (4)
- HMAC-SHA-2 (6), HMAC-SHA-3 (4)
- AES-CBC/CTR/GCM and CMAC-AES (4)
- KDA/PBKDF/KDF components (3), DRBG (2)
- ECDSA/EdDSA/RSA (3), TLS 1.2/1.3 (2)

## Known FIPS Gaps

The following connections currently lack TLS and **must be fixed** for FIPS compliance:

| Component | Endpoint | Issue |
|-----------|----------|-------|
| clickhouse-operator | Prometheus scrape inbound (9999) | HTTP-only, no TLS |
| metrics-exporter | Prometheus scrape inbound (8888/metrics) | HTTP-only, no TLS |

**HTTP is not acceptable for FIPS compliance.** All connections must use TLS with FIPS-approved cipher suites.
