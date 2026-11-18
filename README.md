# HashBrown

![HashBrown](docs/assets/HashBrown-Banner.png)

**HashBrown 0.3.0 tracks evidence files and their chain of custody.** Import existing disk or memory images into a case, verify their SHA-256 hashes and record custody events. See [offline prerequisites and safe validation](docs/prerequisites.md).

The separate [generic offline evidence registration](docs/offline-evidence.md) prototype hashes selected files and writes a manifest without copying them into a case.

For live acquisition, HashBrown can schedule Windows capture through an existing Velociraptor server and resume downloads. These operations create collection jobs and write evidence. Native physical-disk and RAM capture still require qualification on the intended platform.

Distribution: `hashbrown-ir`. Python package: `hashbrown_ir`. Command: `hashbrown`.
It has no runtime Python dependencies and no shared engine, installer, case store,
or automatic linkage with other incident-response tools. Velociraptor is an
external prerequisite only for remote acquisition. No binary is bundled or downloaded.

## Permissions and execution boundaries

**Offline evidence tracking needs local file access only.** No Microsoft 365,
Google Workspace, GAM, Active Directory, domain-admin, cloud-admin or Velociraptor
rights are required for `python -m hashbrown_ir.evidence register|verify`.
Use an ordinary account that can read the authorized inputs; HashBrown does not
grant access, change ACLs or bypass operating-system restrictions.

| Workflow | Access actually needed | Writes and limits |
| --- | --- | --- |
| Generic `evidence register` | Read the specification and every selected regular file; traverse their parent directories. Create, write, hardlink and remove a temporary publication file in the existing output directory. | Publishes a new manifest outside the evidence root; does not copy or modify source files. Output must not exist. |
| Generic `evidence verify` | Read the manifest and selected files beneath the explicit evidence root; traverse their directories. Supply the independently retained manifest SHA-256. | No case, report, checkpoint, lock or output writes. No target collection or network calls. |
| Offline image import, demo, case/report commands and case `verify` | Read selected source images where applicable; create/read/write the analyst's case directory, lock, metadata, temporary files and image spool. | Imports copy images into local storage; verification can update custody/report data and recover pending metadata. These are not non-mutating case reads. No host administrator or device rights are needed for ordinary readable files. |
| `--preflight` on import/collect/sync | Read local case metadata and inspect explicitly supplied paths and analyst-output filesystem capacity. | No API/collector execution, image/config-content reads, locks or case writes. Passing a plan does not validate credentials, remote permissions, staging capacity or capture readiness. |
| Detached `handoff create|verify` | Read selected evidence and provenance. Creation additionally needs an external writable output directory supporting exclusive hardlink publication; verification needs only input access. | No original-case mutation or image copying. Creation writes a new manifest; verification does not. |

**Live acquisition is implemented, but it is outside the suite's offline-only
workflow and is not report-only or disabled by a global safety switch.** The
separate offline evidence entrypoint exposes only registration and verification.
The original `hashbrown` CLI still accepts `install`, `acquisition collect`
and remote `sync`; absent `--preflight`, those paths can execute real operations.
An explicit disk number is required for physical-disk collection, and simulation
requires an explicitly named existing fixture file. These argument checks are not
an authorization system. Do not run the live commands under an offline-only scope.

For a separately authorized existing deployment, the permission boundaries are:

- The analyst process needs access to the local Velociraptor executable and API
  configuration, plus writable local case/spool storage. Credentials remain
  external to evidence; they are not needed by offline registration.
- The API identity must be authorized in the bound organization to inspect the
  selected client, schedule its collection and read its results/uploads.
  Velociraptor's [collection permission checks](https://docs.velociraptor.app/vql_reference/server/collect_client/)
  depend on collection type and artifact policy; a read-results-only identity is
  not a collection identity.
- Artifact installation modifies the server's artifact repository and needs the
  applicable [artifact-writer permission](https://docs.velociraptor.app/vql_reference/server/artifact_set/).
  Binary retrieval through this implementation's
  [`fs_sparse` accessor requires `SERVER_ADMIN`](https://docs.velociraptor.app/vql_reference/accessors/fs_sparse/)
  in the bound organization. The existing separate download credential option
  separates this powerful access from collection metadata access.
- The enrolled client must be able to read an existing source file for file
  collection. Physical disk reads instead need the endpoint's administrative
  [direct-device access](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea#physical-disks-and-volumes).
  RAM capture requires the elevated Windows x64 client and compatible driver
  described in the packaged artifact; Velociraptor's
  [WinPmem function requires `MACHINE_STATE`](https://docs.velociraptor.app/vql_reference/windows/winpmem/).
  It writes a staged image in the client's temporary directory and uploads it.
  Client temporary storage, server upload storage and analyst spool storage must
  each be writable by their respective service/process identity and have capacity.

Those live prerequisites are documented boundaries, not instructions to broaden
host access or change driver/security policy. Actual native disk/RAM acceptance
remains outstanding; successful fixture transfers or hashes do not qualify it.
See [acquisition limits](docs/acquisition.md) and
[offline evidence usage](docs/offline-evidence.md).

## Install and run the inert demo

Python 3.12 or newer:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install .
.\.venv\Scripts\hashbrown.exe --version
.\.venv\Scripts\hashbrown.exe --root .\demo-cases demo
```

On Linux/macOS, use `.venv/bin/python` and `.venv/bin/hashbrown` instead. The same
CLI is available with `python -m hashbrown_ir` from the installed interpreter.

The demo prints JSON with a new `HB-...` case ID, case path, **two verified images**,
no acquisition gaps, and the path to `report.html`. Open that report in a browser.
Each demo run creates a new case. It generates a 1,474,560-byte FAT12 disk image and
a 3,149,824-byte memory-transfer fixture. The latter is **not a Windows RAM dump**
and cannot validate memory analysis. It reads no host disks or physical RAM,
loads no driver, executes no malware, and contacts no server.

## Import an existing image

```powershell
hashbrown --root D:\HashBrownCases case create --label INCIDENT --operator 'Analyst J'
# Use the case_id returned above.
hashbrown --root D:\HashBrownCases case bind HB-2026-REPLACE `
  --client-id C.0123456789abcdef --hostname host.example.test --org root
hashbrown --root D:\HashBrownCases acquisition import HB-2026-REPLACE `
  --client-id C.0123456789abcdef --kind disk --file D:\Evidence\disk.raw `
  --operator 'Analyst J' --sha256 ORIGINAL_SHA256
hashbrown --root D:\HashBrownCases verify HB-2026-REPLACE
```

Replace the sample case ID, client ID, hostname, source path and hash. Bind the
source endpoint's actual Velociraptor ID/FQDN/organization; bindings are immutable.
Existing files can be imported offline after binding. Omit `--sha256` if no
independent acquisition hash is available; HashBrown still hashes the original
and its copy, but that does not establish the original's earlier history.

Images are copied without conversion to `images/ACQ-.../image.bin`. The source is
preserved. `--kind memory` accepts an existing memory image. `--format raw|vhdx|vmdk|other`
is a label for one file, not a format validator or a segmented-image merger.
HashBrown does not collect VMDK dependencies. Use `--simulation` only for synthetic
fixture files. The default maximum image size is 256 GiB; set `--max-bytes` when needed.

## Read-only preflight

Add `--preflight` to the same `acquisition import`, `collect`, or `sync` command
before running it. The plan checks parameters, case/binding metadata, explicitly
named local file metadata, output collisions and advisory free space on the
**analyst's case output filesystem**. It never submits a collection, executes
Velociraptor, loads a driver, opens image/credential contents, or writes evidence,
reports, checkpoints or case locks.

```powershell
hashbrown --root D:\HashBrownCases acquisition import HB-2026-REPLACE `
  --client-id C.0123456789abcdef --kind disk --file D:\Evidence\disk.raw `
  --operator 'Analyst J' --preflight
# Use --plan-format text for a readable plan instead of JSON.
```

The JSON includes `status`, individual `checks`, a `plan` and explicit `unknowns`.
Preflight exits `0` for locally passing checks, `2` for warnings/unknown prerequisites,
and `1` for blocking failures. Passing checks do not certify capture readiness or
completeness. Usage errors still follow argparse's normal exit code `2`.

Remote size is unknown unless a saved upload descriptor or optional
`--declared-bytes N` supplies a planning estimate. A declaration is unverified and
does not constrain a later capture. `--max-bytes` remains the actual byte budget.
Client/server capacity, credential validity, roles, drivers and device extent
remain unknown; preflight makes no remote API requests. Local paths must identify
regular files/directories without observed linked/reparse ancestors. Use local
storage; non-Windows mount backends cannot be certified by these checks.

For saved local imports, `sync ... --preflight --file ORIGINAL` explicitly enables
source identity and source/output same-file checks, including hardlinks and Windows
short-path aliases. It cannot redirect the saved import. Without `--file`, those
source checks remain unknown. New acquisition output IDs are allocated only during
execution, so preflight checks their parent directory, not an as-yet-unallocated
filename. See the [preflight details](docs/acquisition.md#read-only-preflight).

## Remote capture and recovery

See the [acquisition guide](docs/acquisition.md) for artifact installation,
credentials, explicit disk selection, memory staging, interrupted transfers,
partial captures and the optional client/server fixture exercise.

```powershell
hashbrown install --velociraptor C:\IR\velociraptor.exe --api-config C:\IR\api.yaml --org root
hashbrown --root D:\HashBrownCases acquisition collect HB-2026-REPLACE `
  --client-id C.0123456789abcdef --kind memory --operator 'Analyst J' `
  --velociraptor C:\IR\velociraptor.exe --api-config C:\IR\api.yaml
hashbrown --root D:\HashBrownCases acquisition status HB-2026-REPLACE
hashbrown --root D:\HashBrownCases acquisition sync HB-2026-REPLACE --job ACQ-REPLACE `
  --velociraptor C:\IR\velociraptor.exe --api-config C:\IR\api.yaml `
  --download-api-config C:\IR\export-api.yaml
```

The download API identity needs `SERVER_ADMIN` in the bound organization because
binary reads use `fs_sparse`. The optional separate download credential is used
only for binary retrieval. Credential contents are never stored in the case.

## Notes, reports and verification

```powershell
hashbrown --root D:\HashBrownCases acquisition note HB-2026-REPLACE --job ACQ-REPLACE `
  --operator 'Analyst J' --text 'Original acquisition time and method recorded in laboratory notebook.' `
  --reference 'Notebook 4, page 12'
hashbrown --root D:\HashBrownCases report HB-2026-REPLACE
hashbrown --root D:\HashBrownCases acquisition verify HB-2026-REPLACE
```

References are freeform analyst statements. They do not link to another tool's
case, resolve events, or prove that the referenced evidence occurs in an image.
The offline report escapes notes and has no scripts or remote assets. Keep the
case directory together for its relative evidence links.

Full verification checks original metadata, image bytes, partial checkpoints and
the local custody chain. Report generation checks presence, size and modification
time and shows the last full verification time; it does not reread gigabytes.
A valid received-byte hash can coexist with a **partial capture**. The custody
chain is locally tamper-evident, not signed or immutable; a party able to rewrite
the entire case can rewrite its chain. Protect the directory and retain an
independently controlled copy or hash anchor.

Results are JSON on stdout; operational errors go to stderr. Exit code `0` means
success, `1` an operational/integrity error, and `2` a pending/partial acquisition
or reported gaps. Argument usage errors also exit `2`. `acquisition status`
returns `0` after successful inspection; inspect each job's `status` and `integrity`.
The global `--root` option goes **before** the command. Its default is `./hashbrown-cases`.

## Portable evidence handoff

Export explicitly selected committed images' records, then verify manually relocated
images with the detached manifest and HashBrown alone. The recipient does not need
the original case, server, configuration or credentials.

```powershell
hashbrown --root D:\HashBrownCases handoff create HB-2026-REPLACE `
  --job ACQ-REPLACE --output D:\PrivateHandoff\incident.json
# Copy selected images yourself, retaining the paths listed in the summary.
hashbrown handoff verify --manifest E:\Received\incident.json `
  --evidence-root E:\Received --manifest-sha256 INDEPENDENT_MANIFEST_FILE_SHA256
```

The producer freshly hashes selected saved images, checks original metadata/custody,
and writes one new manifest outside the source store without copying images. The
recipient streams fresh hashes and checks the exported provenance. Both leave the
case/images unchanged. Use `--format text` for readable output. Partial captures
remain partial (exit `2`), even when bytes match; integrity failures exit `1`.

The manifest is unsigned. Only selected custody event bodies are exported and
verified; omitted bodies have unverified hash links only. Exchange the full manifest
file hash through an independent channel. No result qualifies native capture or
image readability. See [handoff instructions, schema and limits](docs/handoff.md),
including canonical local paths and the output filesystem's hardlink requirement.

## Validation and limits

```powershell
python -m unittest discover -s tests -v
```

The package tests cover preserved hashes, interruption/resume, wrong endpoint/path,
changed images, tampering, partial captures, custody, case isolation, metadata,
artifact export, read-only preflight and the inert CLI demo. Read the [validation record](docs/validation.md)
for precise boundaries. **Actual physical-device imaging and kernel-driver RAM
capture have not been exercised.** The artifacts implement these operations,
but the designated Windows VM acceptance step remains outstanding. Large-image
throughput and multi-terabyte operation have not been benchmarked.

HashBrown provides acquisition and preservation. It does not supply filesystem
analysis, Volatility analysis, write blocking, forensic conclusions, multiuser
authorization, independent custody signatures or automatic incident correlation.

The [A2 designated-VM acceptance procedure](docs/a2-lab-acceptance.md) and repository-only
`scripts/a2_lab.py` prepare bounded transfer checkpoints and reviewable runtime,
source, byte-integrity and reader-output records. The helper never submits a
collection or automatically qualifies native capture/readability. The repository
suite now has 90 tests, including 13 A2 helper cases and 25 detached handoff cases. Actual
physical acceptance requires an explicitly designated VM and remains outstanding.

MIT license. Selected earlier MIT-licensed acquisition code was adapted; see
[third-party notices](THIRD_PARTY_NOTICES.md). HashBrown is independently installable
and maintains its own source, artifacts, tests and storage format.
