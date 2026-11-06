# IOCO tutorials

Practical tutorials for IOCO View 0.1.0, installed from the
[VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=DarrenJMcLeod.ioco-vscode).
Use Windows x64, Git (including Git Bash), and the .NET 8 runtime or later.
The SDK and IOCO source repository are not required. See the Marketplace listing
for validated preview hosts, and Tutorial 1 for the bundled CLI shell function.

These are preview tutorials. The CLI walkthrough completed on 6 Nov 2026 in a fresh
Windows x64 Sandbox (build 26100), using the public sample download, VS Code 1.142.0,
the Marketplace extension and .NET runtime 8.0.33, with no SDK or IOCO source repository.
Editor mapping, serving-file grouping, declaration navigation, automatic refresh and
the seven-URL Pro lookup were reported as successful by the maintainer. Tutorial 3
includes the sample-local Git identity setup discovered during this walkthrough.
This verification does not expand the Marketplace listing's supported-host claims.

1. [From a bug report URL to its code](01-trace-a-url-to-its-code.md) — Community.
2. [Review a shared-service change](02-review-a-shared-service-change.md) — Pro trial.
3. [Make a client handover map](03-client-handover-map.md) — Community, optional Pro step.

## The sample

[Download ioco-tutorial-sample.zip](https://github.com/Tempuskg/ioco-tutorials/releases/download/tutorial-sample-v0.1.0/ioco-tutorial-sample.zip).
This is the prepared tutorial sample, including setup scripts and the Tutorial 2 change;
use this release asset rather than GitHub's automatically generated source archive.
Extract it and follow Tutorial 1. Make fresh copies for Tutorials 2 and 3.

SHA-256: `9506c6e649ece0cabe5cdc00275c436dddc6165762e0df8969aa4f2da9b22d86`

PowerShell verification: `Get-FileHash .\ioco-tutorial-sample.zip -Algorithm SHA256`.
Git Bash verification: `sha256sum ioco-tutorial-sample.zip`.
The result must match the hash above.

The sample archive is MIT-licensed; see [the sample license](SAMPLE-LICENSE.txt),
also included inside the archive. That license applies to the sample archive only.
IOCO is proprietary software under the EULA distributed with the extension.
The sample license does not grant a license to IOCO or to these tutorial articles.

Read [the analysis limits](analysis-limits.md). Review results identify candidates,
and do not prove runtime impact or establish that a change is safe.

For IOCO support, use the Marketplace listing's Q&A tab.