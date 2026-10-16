# SRUM Parser

Native Rust + egui SRUM viewer with Sigcheck-style signature verification. Reads `SRUDB.dat` itself through the ESE engine built into Windows (esent.dll). No SrumECmd, no CSVs, no other tools, no runtime: one ~4.2 MB exe (the old WPF build was 59 MB; its sources are in `legacy-wpf\`).

Tables: Network Usages, App Timeline Provider, App Resource Use Info.

The published build is `publish\SrumParser.exe`. Close any running copy before replacing it, because Windows locks a running exe.

Run as Administrator to read the live `C:\Windows\System32\sru\SRUDB.dat`. It opens automatically on start (a "Restart as Administrator" button appears if access is denied). Open DB, Ctrl+O, drag-drop (file or folder) or a command-line argument opens another SRUDB.dat.

## Using it

- Toolbar left: search (all columns, case-insensitive), reload, and the filters In Instance (only entries since boot), Show Untrusted (unsigned / tampered / invalid / expired / revoked only) and Show Not Found (files gone from disk only).
- Toolbar right: Network Usage, App Timeline, Resource Use views, Export CSV, Open DB. The space in between shows the row count, signature progress and red warnings (a bare ⚠ when the window is narrow); hover them for details. There is no separate status bar.
- Columns: Timestamp, Name, Path (with the exe's icon), Signature, Generics, then the table's metrics. App Timeline has no Signature or Generics column, and its Copy and CSV export leave out Signature, Signer and Generics. The Show Untrusted and Show Not Found filters still apply there. Entries without a path (services such as CryptSvc, `System`, bare package names) are hidden. Click any header to sort, drag header edges to resize. Hover Signature for the signer and how it was signed; hover a tag for its reason.
- All rows share one background and one text colour. Only the row under the mouse and selected rows (blue, with an accent bar) stand out. The signature verdict is shown as text, not colour; use Show Untrusted to isolate bad ones.
- Click / Ctrl / Shift select rows, arrows / PgUp / PgDn / Home / End move, right-click for Copy, Copy Path, Copy Signer, Open File Location, Filter by This Exe.

| Keys | |
|---|---|
| Ctrl+O · F5 | Open · reload |
| Ctrl+1 / 2 / 3 | Switch table |
| Ctrl+F · Esc | Search · clear search |
| Ctrl+I | In Instance |
| Ctrl+C · Ctrl+A | Copy selected rows (tab-separated) · select all |
| Ctrl+E | Filter by selected exe |
| Ctrl+S | Export current view to CSV |

## Signatures

Same method as Sysinternals Sigcheck (the source of VirusTotal's "Signature info"):

1. SRUM paths (`\Device\HarddiskVolume3\...`) are mapped to drive letters with `QueryDosDevice`. Service groups (`svchost.exe [netsvcs]`) are stripped for the file check.
2. Embedded Authenticode: `WinVerifyTrust` with `WINTRUST_ACTION_GENERIC_VERIFY_V2`.
3. No embedded signature: Windows catalogs via `CryptCATAdminAcquireContext2` (SHA-256, then SHA-1), `CryptCATAdminCalcHashFromFileHandle2`, `CryptCATAdminEnumCatalogFromHash`, then `WinVerifyTrust` against that catalog. Many System32 files are only catalog-signed.
4. Embedded signatures are also checked independently: the PE Authenticode digest (whole file minus checksum, certificate directory entry and certificate table) is recomputed with BCrypt and compared with the digest inside the signature. A mismatch is **Tampered**: the file was patched after signing, or a signature was copied from another file (SigThief). This matters because WinVerifyTrust hands the digest check to the PE SIP registered in `HKLM\SOFTWARE\Microsoft\Cryptography\OID\EncodingType 0\CryptSIPDllVerifyIndirectData\{C689AAB8-8E78-11D0-8C47-00C04FC295EE}`. Malware can point that key at a function that always succeeds (for example `ntdll.dll!DbgUiContinue`), and then Windows, PowerShell and Sigcheck all report forged signatures as valid.
5. That SIP key, `CryptSIPDllGetSignedDataMsg` and the Authenticode final-policy provider are checked on start. If any of them no longer points at `WINTRUST.DLL`, the toolbar shows "Signature checks hijacked"; hover it for the redirected entries.
6. Result: Signed, Tampered, Unsigned, Revoked, Invalid (untrusted or broken chain), Expired, Not Found. The signer comes from the leaf certificate.

Offline: revocation is checked against cached CRLs only (`WTD_CACHE_ONLY_URL_RETRIEVAL`), and nothing is uploaded. Results are cached per path + size + modified time in `%LOCALAPPDATA%\SrumParser\sigcache.tsv`; the file is versioned, so a new checker version discards old verdicts. Files are checked on several threads after the database loads.

## Generics tags

| Tag | Flags |
|---|---|
| NET | Unsigned / tampered / invalid / expired / revoked binary that SRUM saw sending or receiving network data |

## How it reads

- Copies SRUDB.dat (open in the SRUM service while Windows runs) and the SRU*.log / SRU.chk files to a temp folder.
- Locked file (another process holds it without read sharing, a known anti-forensics trick): the file is opened with `FILE_READ_ATTRIBUTES` only, which share modes do not block. `FSCTL_GET_RETRIEVAL_POINTERS` lists its clusters, and those are read straight from the volume (Administrator required). The toolbar then shows "SRUDB.dat locked" and names every process other than the SRUM service host that has the file open (Restart Manager). Changes not yet flushed to disk may be missing from a raw read. Logs let ESE replay changes not yet flushed into the DB. If the logs do not fit, it opens the DB alone.
- Opens the copy read-only with esent.dll (page size read from the DB header) on a worker thread; rows stream into the grid in batches.
- Tables: `{973F5D5C-1D90-4944-BE8E-24B94231A174}` network, `{5C8CF1C7-7257-4F13-B223-970EF5939312}` app timeline, `{D10CA2FE-6FCF-4F6D-848E-B2E99266FA89}` app resource use. `SruDbIdMapTable` maps AppId to the exe string.
- Timestamps: stored UTC, shown in local time, sorted on UTC.
- Temp folder is deleted after each read; leftovers older than 12 h are cleaned on start.
- A panic writes `%TEMP%\SrumParser-crash.txt`.

## Why it is small and fast

- egui on OpenGL (glow): no wgpu, no accesskit. Only Poppins Regular + Medium are embedded (~320 KB, SIL Open Font License); Segoe UI fallbacks and icon glyphs come from `%WINDIR%\Fonts`.
- Win32, esent.dll and wintrust.dll are bound with `raw-dylib` (no `windows` crate). The CRT is linked statically, so no VC++ redistributable is needed.
- Rows are stored column-wise and the grid lays out only visible rows. Search scans one contiguous text buffer per table, and sorting uses precomputed integer keys. Names, paths, signatures and icons are resolved once per distinct exe.
- Release profile: `opt-level = "s"` (smaller than "z" here), fat LTO, 1 codegen unit, `panic = "abort"`, stripped.

## Known issues

- With the default parallel test runner, the test process sometimes dies with `STATUS_ACCESS_VIOLATION` (about half of runs). It only happens when `raw::tests::copies_exclusively_locked_file` and `sig::tests::signatures` run at the same time; each passes alone and the whole suite passes on one thread. The cause is not found yet. It may also affect the app when a locked SRUDB.dat is read while signature checks are running.
