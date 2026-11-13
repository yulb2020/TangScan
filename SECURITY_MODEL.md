# TangScan threat model

Repository: `yulb2020/TangScan` (github.com). No infrastructure repository exists (user confirmed); this is a client-side tool with no deployment. No `SECURITY_MODEL.md` existed; this model is built from the code plus the user's answers.

## User-provided facts
- **Usage:** run **locally by security researchers via the CLI** against targets they are authorized to test (user answer). The hosted-platform/sandbox path (`TSExploit.is_sanbox()`, `APP_NAME` env) is not the deployment being modeled.
- **Attackers:** **every attacker matters — nothing is ruled out** (user answer).
- **Accepted risk / out of scope:** **none — report everything** (user answer), including PoC payload behavior, `verify=False`, and vendored libraries.
- **Infrastructure:** none (user answer).

## System overview and trust boundaries
TangScan is a Python 2 PoC-plugin SDK for the TangScan (wooyun) vulnerability-scanning platform. A researcher runs `python <poc>.py --url/--host ... [--mode verify|exploit] [--debug]`. The PoC sends probes/exploits to a remote target and parses the target's response, then prints a result.

Trust boundaries:
1. **Operator → CLI**: argparse options (`modules/main.py`) converted by `TSConvert` (`modules/exploit.py`). Operator is trusted-ish but options can come from scripts/wrappers.
2. **Scanner ↔ target (network)**: all target responses are attacker-controllable (a target can be a honeypot or compromised). TLS verification is disabled in PoCs (`verify=False`).
3. **PoC code → framework**: PoC plugins are community-contributed Python executed with the operator's privileges.
4. **Vendored dependencies**: `tangscan/thirdparty/` (old `requests`/`urllib3`/`chardet`, `purl`, `colorama`, `six`) handle all HTTP parsing.

## Components
| Name | Code |
|---|---|
| **Framework** | `tangscan/modules/exploit.py` (TSExploit, TSConvert, logging), `tangscan/modules/dotted.py` (DottedDict), `tangscan/modules/main.py` (CLI runner, option parsing, result output) |
| **PoC Plugins** | `tangscan/*.py` (`bash_remote_code_execute.py`, `eyou4_list_userinfo_sql_injection.py`, `mongodb_unauth.py`, `poc_example.py`, `poc_template.py`) |
| **Vendored Libraries** | `tangscan/thirdparty/**` |
| **Docs** | `docs/**` (Sphinx docs, theme JS/HTML) |

## Entry points
- CLI args (`main.py`): `--url`, `--host`, `--port`, `--cmd`, `--mode`, `--debug`, plus PoC-defined options; `json_field` converter calls `json.loads` on input.
- Network responses from targets: regex-parsed in PoCs (`re.findall` on `response.content`), echoed to terminal (`print_*`, `main.py` result printing) — possible terminal escape injection from target-controlled content.
- MongoDB server responses (`mongodb_unauth.py` via `pymongo`).

## Assets
- The researcher's workstation and shell (code execution, terminal integrity).
- Credentials/data extracted from targets (e.g. admin password dumped by eyou4 exploit) shown in output.
- Integrity of scan results (false positives/negatives drive platform payouts per README).
- Authorized-target boundary: the operator should not be tricked into hitting unintended hosts (redirects, SSRF-like behavior).

## Attacker profiles (all in scope per user)
- Malicious or compromised **scan target** returning crafted HTTP/Mongo responses.
- **Network MITM** (TLS verification disabled).
- **Malicious PoC contributor** submitting plugin code.
- Anyone influencing CLI options (wrapper scripts, shared command lines).

## Existing controls
- Results must be JSON-serializable (`main.py` check).
- `--mode verify` default; exploit only on explicit `--mode exploit`.
- Few others: no sandboxing of PoCs in local mode, no output sanitization, no TLS verification, Python 2 only, vendored libs unpatched since ~2014.

## Out of scope / accepted risk
None. The user asked to report everything.

## Open questions
- Is the vendored `thirdparty/` code ever updated, or should findings there point to replacing it with maintained packages?
- Are PoCs ever run unattended (CI/batch) against untrusted target lists?
