# Label Studio: server-side request through the task-import URL (CVE-2022-36551)

Case study of a server-side request forgery advisory (CWE-918) in
[HumanSignal/label-studio](https://github.com/HumanSignal/label-studio) (formerly `heartexlabs/label-studio`): the
project task-import API opens a caller-supplied URL with `urllib.request.urlopen` without checking its scheme;
the upstream fix rejects `file://` URLs before the call.

> **Demo only, not for use.** This repository is a case study built from unmodified snapshots of
> [HumanSignal/label-studio](https://github.com/HumanSignal/label-studio) at two historical commits. It is not a fork and is not maintained. Do not install, deploy, or
> depend on it: the `baseline` branch contains code affected by the advisories below.

**This branch: `baseline`** (tag `poe`): snapshot of `HumanSignal/label-studio` @ [`ad29804731`](https://github.com/HumanSignal/label-studio/commit/ad298047312123e2fb781cd6d7b9668ed1b51edc) (2022-08-22), the parent of the upstream fix commit, i.e. the last vulnerable state. The patched snapshot is on [`fix`](https://github.com/ferralon-demo/label-studio-url-import-ssrf/tree/fix).

## Advisory

| | |
|---|---|
| **CVE** | [CVE-2022-36551](https://nvd.nist.gov/vuln/detail/CVE-2022-36551) |
| **GHSA** | [GHSA-pc6f-259w-w3j6](https://github.com/advisories/GHSA-pc6f-259w-w3j6), "Heartex - Label Studio Community Edition vulnerable to SSRF in the Data Import module", CWE-918 |
| **Affected** | `label-studio` < 1.6.0, per the GHSA record |
| **Upstream fix** | [HumanSignal/label-studio#2840](https://github.com/HumanSignal/label-studio/pull/2840), commit [`501142cb81`](https://github.com/HumanSignal/label-studio/commit/501142cb815ac964b0c600c491885b67386870c2), merged 2022-08-23 |

## Ingress and sink at `baseline`

All line numbers are at upstream commit `ad29804731`.

| Step | Location |
|---|---|
| **URL route** | `path('<int:pk>/import', api.ImportAPI.as_view(), name='project-import')`, [`label_studio/data_import/urls.py:17`](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/urls.py#L17), mounted under `api/projects/` at [line 26](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/urls.py#L26) and included from [`label_studio/core/urls.py:58`](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/core/urls.py#L58). No decorator: the view is registered through Django URL configuration. |
| **View** | `class ImportAPI(generics.CreateAPIView)`, [`label_studio/data_import/api.py:161`](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/api.py#L161); `post` ([line 175](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/api.py#L175)) delegates to DRF's `CreateAPIView.post`, which dispatches to `ImportAPI.create` ([line 198](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/api.py#L198)); `create` calls `load_tasks(request, project)` at [line 209](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/api.py#L209) |
| **Input** | `load_tasks`, [`label_studio/data_import/uploader.py:143`](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/uploader.py#L143), reads form field `url` at [line 162](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/uploader.py#L162) |
| **Existing check** | `if settings.SSRF_PROTECTION_ENABLED and url_is_local(url): raise ImportFromLocalIPError`, [lines 175-176](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/uploader.py#L175-L176); `SSRF_PROTECTION_ENABLED` defaults to `False` ([`label_studio/core/settings/base.py:372`](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/core/settings/base.py#L372)). No URL-scheme check precedes the call. |
| **Call** | `tasks_from_url(file_upload_ids, project, request, url)` at [line 177](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/uploader.py#L177); `tasks_from_url` defined at [line 114](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/uploader.py#L114) |
| **Sink** | stdlib `urllib.request.urlopen(url, context=ctx)` at [`uploader.py:123`](https://github.com/HumanSignal/label-studio/blob/ad298047312123e2fb781cd6d7b9668ed1b51edc/label_studio/data_import/uploader.py#L123) |

## What the fix changes

At `501142cb81`, `load_tasks` raises `ValidationError('"url" is not valid')` when
`url.strip().startswith('file://')` ([lines 178-179](https://github.com/HumanSignal/label-studio/blob/501142cb815ac964b0c600c491885b67386870c2/label_studio/data_import/uploader.py#L178-L179)),
before the `tasks_from_url` call (now [line 181](https://github.com/HumanSignal/label-studio/blob/501142cb815ac964b0c600c491885b67386870c2/label_studio/data_import/uploader.py#L181)). The commit also
adds `label_studio/tests/data_import/test_uploader.py`. The call to `urlopen` is still present: the fix adds a
guard and removes no call.

## Provenance

This is not a fork: a fresh repository whose code branches hold upstream source snapshots, with no upstream
history and no upstream remote.

| Branch | Tag | Upstream commit | Date | Snapshot commit | Snapshot tree |
|---|---|---|---|---|---|
| `baseline` | `poe` | [`ad29804731`](https://github.com/HumanSignal/label-studio/commit/ad298047312123e2fb781cd6d7b9668ed1b51edc) | 2022-08-22 | `f75fe894f078f7afa79968fc71d643109eb6cf42` | `ccd778fa8dfb6472b97638a3ede19b9cc26571dc` |
| `fix` | `pone` | [`501142cb81`](https://github.com/HumanSignal/label-studio/commit/501142cb815ac964b0c600c491885b67386870c2) | 2022-08-23 | `2a140fbde0929875e262cf68618318744b8e7f8a` | `d1293b5feb153a5eb4d75b11a563ae9974e8ea7a` |

Each snapshot commit is the upstream tree at that commit with one change: `.github/workflows/` (24 files) is
removed so that upstream CI does not run here. No other file is added, removed, or edited. The `fix` snapshot's
parent is the `baseline` snapshot, so the diff between the two snapshot commits
([compare](https://github.com/ferralon-demo/label-studio-url-import-ssrf/compare/f75fe894f078f7afa79968fc71d643109eb6cf42...2a140fbde0929875e262cf68618318744b8e7f8a)) is exactly the upstream fix commit's diff. On each code branch, a second
commit replaces upstream's `README.md` with a branch-specific version of this file; upstream's README is in the snapshot commit.

To check a snapshot against upstream from a clone of this repository:

```sh
git fetch https://github.com/HumanSignal/label-studio ad298047312123e2fb781cd6d7b9668ed1b51edc
git diff --stat FETCH_HEAD f75fe894f078f7afa79968fc71d643109eb6cf42   # lists only .github/workflows/*
```

## Licence

Label Studio is licensed under the Apache License 2.0. The upstream `LICENSE` and `NOTICE` are kept verbatim at
the root of the `baseline` and `fix` branches, as are the third-party licence files upstream ships
(`licenses/django_license.txt`, `licenses/python_license.txt`, and the `*.LICENSE.txt` files beside the bundled
front-end JavaScript under `label_studio/frontend/dist/`). Changes made for this case study, on 2026-10-07:
`.github/workflows/` removed, and this `README.md` replaces upstream's.
