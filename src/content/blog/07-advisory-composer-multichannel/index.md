---
title: "One Scan, Four Channels: Advisory Composer as a Pure Client-Side Tool"
summary: "Vulnerability disclosure inside a company is always multi-channel — email to owners, Slack to teams, a PR comment on the dependency bump, a CSV for the audit log. Same data, four formats. Here's how I built a browser-only tool that does all four from a single lockfile upload."
date: "Apr 12 2026"
draft: false
tags:
- React
- TypeScript
- Automation
- OSV
- EPSS
- SBOM
- Security
---

Every vulnerability scan produces the same shape of output and every vulnerability response produces the same shape of communication. On one side: a list of advisories keyed by package, version, severity, and references. On the other side: an email to the owner, a Slack message to the team, a comment on the pull request that bumps the dependency, a row in the incident spreadsheet.

The data in each output is the same. The formatting is completely different. And the time it takes to manually convert between them is real — enough that at my day job I'd watched teammates spend half a day on an L0 incident just re-typing the same advisory four ways for four audiences.

The right response is a tool that takes the scan output once and emits all four formats in a single pass. Advisory Composer is that tool, built as a pure browser app: no backend, no accounts, no server-side state. Paste a lockfile, click scan, walk away with four downloadable outputs.

## The architecture in one sentence

Parse lockfile → batch-query OSV → enrich with EPSS → run four generators → offer downloads.

Everything happens in the browser. No step requires a server. The whole tool is a static React bundle deployed to GitHub Pages.

## Why pure client-side

The original inspiration is an internal tool I'd written in Streamlit that hooked directly into Outlook via `win32com.client` to push per-recipient drafts with custom filtered attachments. That worked because it ran on a Windows workstation inside an enterprise that allowed COM interop. For a public portfolio project, that's not an option — and trying to replicate "send an email directly from the tool" in a browser is a well-known dead end (the browser cannot speak SMTP directly, and proxying through an API requires a server and a sender domain).

So the design inverted: instead of *sending*, the tool *generates downloadable files*. An `.eml` you drag into Outlook. A Slack JSON you paste into Block Kit Builder. A markdown comment you paste into a PR. A CSV you save to disk. The tool never holds credentials, never talks to your email server, never needs a backend.

This has a second benefit I didn't plan for: lockfiles can contain sensitive internal dependency names (private npm scopes, internal Go modules, proprietary Cargo crates). Sending them to a third-party backend is a security conversation in most organisations. Keeping everything in the browser means the tool works with private dependency lists without asking anyone to trust a new service.

## Parsing six lockfile formats without a runtime dependency

I tried `@yarnpkg/lockfile` first. It's tiny, it works, and it adds a dependency for a format that can be parsed with ~20 lines of TypeScript. Same story for the other formats. Every lockfile format Advisory Composer supports is simple enough to hand-parse, and doing so kept the bundle small and avoided a pile of transitive dependencies.

### `package.json` and `package-lock.json`

```typescript
export function parsePackageJson(content: string): ParsedDependency[] {
  const json = JSON.parse(content);
  const deps = { ...json.dependencies, ...json.devDependencies };
  return Object.entries(deps).map(([name, range]) => ({
    name,
    version: cleanVersion(range as string), // strip ^, ~, >=
    ecosystem: "npm",
  }));
}
```

For `package-lock.json` v2 and v3 the `packages` field has exact resolved versions in the keys; v1 uses a nested `dependencies` tree. Both paths are handled, and the tool logs which lockfile version it detected so the user knows.

### `requirements.txt`

Lines starting with `#` are comments. Blank lines are ignored. URL requirements (`package @ https://...`) are skipped with a warning because OSV wants a version, not a URL. Everything else matches a regex of `name[operator]version` where the operator can be `==`, `>=`, `~=`, or absent. If the operator isn't `==` the version is stored anyway, because OSV handles range-based queries fine.

### `go.mod`

Only the `require` blocks matter for vulnerability scanning. The parser walks the file line by line, tracks whether it's currently inside a `require (...)` block, and emits one dependency per line in the form `<module-path> <version>`. Go modules use `v1.2.3` notation natively, which OSV expects, so no version cleanup is needed.

### `Cargo.toml`

Two dependency syntaxes coexist in Cargo: `package = "1.2.3"` and `package = { version = "1.2.3", features = [...] }`. Rather than reach for a full TOML parser, the Advisory Composer parser walks the `[dependencies]` section and handles both with regex. Dev-dependencies are parsed too.

### CycloneDX SBOM

The modern pattern for dependency manifests at enterprise scale is the Software Bill of Materials, usually in CycloneDX JSON format. The `components` array has every package with a `purl` (Package URL) that encodes the ecosystem. `pkg:npm/lodash@4.17.11` → npm / lodash / 4.17.11. A small `purl` splitter handles the prefixes and strips any `@` version suffix for the version field.

All six parsers dispatch through a single `parseManifest(content, filename)` function that sniffs the format from the filename or — for pasted content without a filename — from the first few characters (`{` with `"name"` key → package.json; `[` or `<?xml` → SBOM variants; `module` keyword → go.mod).

## OSV as the single scanner backend

OSV.dev is the ideal backend for a client-side vulnerability scanner. Its batch API accepts up to 1000 queries per request, returns structured JSON, supports every ecosystem Advisory Composer parses, and is public with no authentication or rate limit for browser use.

A query is a `{package: {name, ecosystem}, version}` object. The scan loops the parsed dependencies, chunks them into groups of 1000, and POSTs each chunk to `/v1/querybatch`:

```typescript
async function scan(deps: ParsedDependency[]): Promise<OSVResult[]> {
  const chunks = chunkArray(deps, 1000);
  const results = await Promise.all(
    chunks.map(chunk =>
      fetch("https://api.osv.dev/v1/querybatch", {
        method: "POST",
        body: JSON.stringify({
          queries: chunk.map(d => ({
            package: { name: d.name, ecosystem: d.ecosystem },
            version: d.version,
          })),
        }),
      }).then(r => r.json())
    )
  );
  return results.flatMap(r => r.results);
}
```

OSV returns an array of results in the same order as the queries. Each result has a `vulns` array (possibly empty), where each vuln has an `id`, a `summary`, a `severity` array (CVSS vectors), `aliases` (usually including a CVE ID), and an `affected` array.

### Severity normalisation

OSV reports severity in two places and they don't always agree. The `severity` array has CVSS vectors (usually CVSS v3.1). The `db_specific` object often has a `severity` string like `"HIGH"` or `"CRITICAL"`. The scanner tries the array first (more structured, more comparable across sources), and falls back to `db_specific` if the array is empty:

```typescript
function normaliseSeverity(vuln: OSVVuln): Severity {
  const cvss = vuln.severity?.find(s => s.type === "CVSS_V3")?.score;
  if (cvss) return fromCvssVector(cvss);
  const dbString = vuln.database_specific?.severity;
  if (dbString) return fromString(dbString);
  return "UNKNOWN";
}
```

This matters because the output generators group by severity, and inconsistent severity strings would produce messy section headers.

## EPSS enrichment

For every CVE alias on every OSV vulnerability, the scanner fetches an EPSS score. FIRST.org exposes this at `https://api.first.org/data/v1/epss?cve_id=CVE-YYYY-XXXX` as a public, no-auth endpoint.

Naively, this is N round trips for N vulnerabilities. In practice the same CVE often shows up multiple times across an OSV result (different advisories for the same CVE), so the enrichment layer caches per CVE:

```typescript
const epssCache = new Map<string, number | null>();

async function getEpss(cveId: string): Promise<number | null> {
  if (epssCache.has(cveId)) return epssCache.get(cveId)!;
  const res = await fetch(`https://api.first.org/data/v1/epss?cve_id=${cveId}`);
  const json = await res.json();
  const score = json.data?.[0]?.epss ? parseFloat(json.data[0].epss) : null;
  epssCache.set(cveId, score);
  return score;
}
```

And the batch of lookups runs with controlled concurrency — 10 simultaneous requests rather than all at once — to be polite to the API:

```typescript
async function enrichAll(vulns: Vulnerability[]): Promise<Vulnerability[]> {
  const semaphore = new Semaphore(10);
  return Promise.all(
    vulns.map(async v => {
      await semaphore.acquire();
      try {
        if (v.cveId) v.epss = await getEpss(v.cveId);
        return v;
      } finally {
        semaphore.release();
      }
    })
  );
}
```

A 50-vulnerability scan takes a few seconds end-to-end: OSV in one round trip, EPSS in 5 batches of 10 with per-CVE caching cutting duplicate calls.

## The four generators

Same input, four outputs. Each generator is a pure function from `(config, vulnerabilities) → string`.

### Email — `.eml` with CSV attachment

The `.eml` file is RFC 2822 MIME-multipart. No library needed:

```
From: Security Team <security@example.com>
To: Backend Team <owner@example.com>
Subject: [DRAFT - DO NOT SEND] Security Advisory — 12 vulnerabilities
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="bnd"
X-Priority: 1

--bnd
Content-Type: text/html; charset=utf-8
Content-Transfer-Encoding: quoted-printable

<html><body>
  <h2>Security Advisory</h2>
  <table>...</table>
</body></html>

--bnd
Content-Type: text/csv
Content-Disposition: attachment; filename="advisories.csv"
Content-Transfer-Encoding: base64

UGFja2FnZSxWZXJzaW9uLFNldmVyaXR5LEVQU1MsQ1ZFCg...

--bnd--
```

The tricky parts are remembering to base64-encode the attachment, setting the boundaries correctly (they must not appear in the content, so they're generated with a random suffix), and including the blank line between the header and body per spec. The generator tests this by saving a sample output and opening it in Outlook, Apple Mail, and Thunderbird — all three render the HTML body, show the attachment, and let you edit the subject or recipient before sending.

### Slack — Block Kit JSON

Slack's Block Kit is a declarative JSON format. The generator produces a header block, a context line with the scan summary, a divider per severity group, and a section block per vulnerability with fields for package / version / advisory ID / EPSS:

```typescript
{
  blocks: [
    { type: "header", text: { type: "plain_text", text: "Security Advisory" } },
    { type: "context", elements: [{ type: "mrkdwn", text: `*${vulnCount}* vulnerabilities, *${criticalCount}* critical` }] },
    { type: "divider" },
    { type: "header", text: { type: "plain_text", text: "CRITICAL" } },
    ...criticalVulns.map(v => ({
      type: "section",
      fields: [
        { type: "mrkdwn", text: `*Package:*\n${v.package}` },
        { type: "mrkdwn", text: `*Version:*\n${v.version}` },
        { type: "mrkdwn", text: `*Advisory:*\n<${v.url}|${v.id}>` },
        { type: "mrkdwn", text: `*EPSS:*\n${(v.epss * 100).toFixed(1)}%` },
      ],
    })),
    // ...HIGH group, MEDIUM group, etc.
    { type: "context", elements: [{ type: "mrkdwn", text: "Generated by Advisory Composer" }] },
  ],
}
```

Output pastes straight into Slack's [Block Kit Builder](https://app.slack.com/block-kit-builder) for preview, or into any Slack webhook payload.

### PR comment — Markdown

Grouped tables under severity headings, with a collapsible `<details>` section per advisory for the full summary:

```markdown
## Security Advisory

> **12 vulnerabilities found** across 8 packages. **3 critical**, 5 high, 4 medium.

### CRITICAL

| Package | Version | Advisory | EPSS |
|---------|---------|----------|------|
| lodash | 4.17.11 | [GHSA-35jh-r3h4-6jhm](https://github.com/advisories/GHSA-35jh-r3h4-6jhm) | 94.2% |

<details>
<summary>GHSA-35jh-r3h4-6jhm — Command Injection in lodash</summary>

Versions of `lodash` prior to 4.17.12 are vulnerable to command injection...

</details>
```

GitHub renders this cleanly in PR comments with the collapsible details block folded by default. You get a compact top-level view with drill-down on request.

### CSV — flat audit export

One row per advisory, columns: `Package,Version,Ecosystem,AdvisoryID,Severity,EPSS,Summary,URL`. Fields containing commas or quotes are wrapped and escaped per RFC 4180. Opens in Excel, Google Sheets, Numbers, or any CSV-aware tool without further processing.

## The four-step wizard

Step 1: upload or paste. Step 2: scan results. Step 3: configure output fields (owner name, severity threshold, draft-mode toggle, channel multiselect). Step 4: preview + download.

The state lives in a single `useReducer` at App level:

```typescript
type AppState = {
  step: 1 | 2 | 3 | 4;
  dependencies: ParsedDependency[];
  vulnerabilities: Vulnerability[];
  config: OutputConfig;
  outputs: Record<Channel, string>;
};
```

Each step is a separate component that reads the parts of state it needs and dispatches actions to advance. Step 3 regenerates the outputs on every config change, but only for the selected channels, so the live preview updates without redundant work.

The wizard structure also forces a clean recovery path. If OSV is down, Step 2 shows the error and offers retry. If the lockfile is malformed, Step 1 shows the parse error and highlights the problem line. You never get stuck in a half-configured state because there is no half-configured state — each step has a clean enter/exit boundary.

## Why this is a better shape than "send from the tool"

The original day-job inspiration sent mail directly through Outlook. That's faster for the end user in the best case, but it bakes in assumptions about the runtime environment (Windows + Outlook), requires write access to the sender's mailbox (mistakes become noise in sent-items), and makes the tool a security-sensitive artefact (whoever runs it has an authenticated send capability).

Advisory Composer's "generate files, don't send" approach is slower by one human step — the user has to click the `.eml` to open it in their mail client and hit send — but it gains portability (works on any OS with any mail client), reversibility (drafts sit in the user's drafts folder until they decide), and separation of concerns (the tool generates, the user approves and dispatches).

The draft-mode toggle is a nod to the same principle: every generated output has `[DRAFT - DO NOT SEND]` prefixed to the subject by default, and turning that off is an explicit action. The tool treats the user's send button with respect.

## What's next

A couple of things I'd like to add:

**GitHub Advisory GraphQL integration** — OSV has broad coverage but the GitHub Advisory database has richer metadata for GitHub-hosted ecosystems. A user-supplied PAT (optional, client-side only) would unlock deeper advisory information for npm, PyPI, rubygems, and crates.io packages.

**Team-mapping input** — instead of one output per channel, a CSV of "package → owner team" could split the scan into per-owner outputs automatically. This is the piece closest to the original Outlook tool: one upload, N emails, each one pre-addressed to the right owner with only their affected packages.

**Scheduled scans via the user's own GitHub Actions** — Advisory Composer is ephemeral, but the generated outputs and the OSV queries underneath could become a reusable Action that runs on every push to a lockfile and comments on the PR automatically. The generator library would move into an npm package; the UI would stay for interactive use.

## What this demonstrates

Advisory Composer is deliberately boring in its engineering choices. Six regex-based parsers. One batched API. One EPSS fetch loop. Four string-returning generator functions. No backend, no dependencies beyond React itself. The whole app is under 1000 lines of TypeScript.

The interesting part is the *shape* — the decision to generate four outputs from one input in one pass, rather than picking a single channel or forcing the user to configure channels per-scan. That shape is the product. The engineering is whatever it takes to make the shape feel natural.

The live tool is at [tamasczaban.github.io/advisory-composer](https://tamasczaban.github.io/advisory-composer/). The source is [github.com/TamasCzaban/advisory-composer](https://github.com/TamasCzaban/advisory-composer).
