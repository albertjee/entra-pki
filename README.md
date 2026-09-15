# entra-pki

![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-live-brightgreen?logo=github&logoColor=white)
![Format](https://img.shields.io/badge/CRL%20format-DER-blue)
![Transport](https://img.shields.io/badge/transport-HTTPS%20only-success)
![Secrets](https://img.shields.io/badge/secrets-none%20%E2%80%94%20public%20by%20design-blueviolet)

A CRL distribution point for [EntraCBA](https://github.com/albertjee/entra-cba) —
self-hosted, zero-infrastructure revocation for Microsoft Entra
**Certificate-Based Authentication (CBA)** labs, served as static files on GitHub Pages.
No Azure subscription, no App Service, no portal clicks.

## Why this repo exists

The [EntraCBA Field Console](https://github.com/albertjee/entra-cba) issues a self-signed
root CA and leaf certificate per engagement, then configures the tenant's
`certificateBasedAuthConfiguration` to trust that root. Entra fetches the CRL from the
`certificateRevocationListUrl` set in that trust config — **this repository, published
via GitHub Pages, is that URL**.

The console pushes each DER CRL here with `git` and `gh` credentials; Entra fetches it
anonymously over HTTPS. If the URL configured in the tenant ever drifts from what this
repo publishes, sign-in fails with `AADSTS50017` — that mismatch is exactly what this
repo exists to prevent.

## Layout

    cba.crl                     # creation-time placeholder (perClient=false mode; see note)
    Contoso-Labs/cba.crl        # per-client CRL (default perClient mode)
    <CLIENT_NAME>/cba.crl       # one folder per engagement, created on first publish

- The root-level `cba.crl` is a creation-time placeholder and is **not** served to any
  tenant; live tenants always fetch `…/<CLIENT_NAME>/cba.crl`.
- `CLIENT_NAME` is validated as `[A-Za-z0-9_.-]+` by the publisher before it becomes a
  folder or URL segment.
- CRLs are DER-encoded, signed by the per-client root CA, and re-published with an
  incremented `CRLNumber` on every re-run.

## Live endpoints

| Client       | CRL URL                                                                   | Status |
|--------------|---------------------------------------------------------------------------|--------|
| Contoso-Labs | https://albertjee.github.io/entra-pki/Contoso-Labs/cba.crl                 | ![live](https://img.shields.io/badge/status-live-brightgreen) |

## How publishing works

`entra_cba/github_pages.py` in the console does the work:

1. Builds the binary DER CRL locally (`pki.build_crl`, signed by the client's root key).
2. Writes it under `out\<CLIENT_NAME>\`, then pushes it here via `gh`-supplied git
   credentials — argv lists only, no shell, nothing interpolated.
3. `tenant.py` configures the Entra x509 authentication method with
   `https://albertjee.github.io/entra-pki/<CLIENT_NAME>/cba.crl` as the fetch URL.

The console's preflight verifies `gh`, `git`, and git-push credentials before any
publish; its Verify phase re-fetches this URL and reports `crl: PASS/FAIL`.

## Security

This repository holds **public material only** — CRLs and optionally CA certificates.
Nothing here is sensitive by design; client folder names are visible by design.
Private keys and PFX bundles live in the console's DPAPI-protected store, never in git.
GitHub Pages satisfies Entra's two hard requirements for a CRL URL: HTTPS and
anonymous reachability. Note that revocation checking only works while this site stays
published — taking it down breaks revocation for leaves under these CAs.

## Related

- [EntraCBA Field Console](https://github.com/albertjee/entra-cba) — the tool that
  publishes here and drives the full engagement lifecycle.
- Field runbook: `Field Runbook Microsoft Entra Certificate-Based Authentication (CBA)
  Setup.md` in the console repo.