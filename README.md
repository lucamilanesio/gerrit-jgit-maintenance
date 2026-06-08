# JGit Servlet-4 Divergence — Maintenance Options

A Gerrit & JGit maintainer's ranked assessment of four approaches to bridging Gerrit `stable-3.13` against JGit `stable-7.4` through the jakarta.servlet / Jetty 12 transition.

*By **David Ostrovsky** — Gerrit & JGit maintainer.*

> **→ [Read the full analysis](https://davido.github.io/gerrit-jgit-maintenance/)**

---

## The Problem

JGit master has moved to **jakarta.servlet 6.1.0** on Jetty 12, because of the alignment with the Eclipse project lifecycle.
Gerrit `stable-3.13` still runs on **javax.servlet 4.0.1**, and is subject to Google's policy on global dependencies updates.

The above situation is caused by the fact that JGit and Gerrit belong to two different organizations
with different policies:

- **JGit** is part of the Eclipse Foundation and the global release cycle.
  Matthias Sohn is the project leader and can approve the dependency updates.
  Google is one of the committers, but not the owner of the project.

- **Gerrit** is a project that belongs to Google and follows the company-wide dependencies system.
  Google has veto on the dependency updates that are used in the *.googlesource.com fork of Gerrit
  and requires a *Library-Compliance* vote from a Googler.

> **NOTE**: This document refers to the Servlet upgrade issue; however, the same problem also exists for many
> other dependencies that are misaligned between JGit and Gerrit, and for future security or maintenance
> dependency bumps.

To support consumers that haven't yet migrated, JGit maintains a special `servlet-4` branch that tracks master but holds back the **jakarta.servlet namespace move** — using **Jetty 12's ee8 compatibility layer** to keep its HTTP/LFS modules (`org.eclipse.jgit.http.server`, `http.test`, `junit.http`, `lfs.server`, `lfs.server.test`) on the `javax.servlet 4` API surface Gerrit needs to compile.

> [!NOTE]
> **Scope is bounded:** three reverts are the entire set of JGit-side changes needed, and no future additions to the fork are expected.

**Why not consume the `servlet-4` branch directly?** It's designed exactly for our servlet stack — but it tracks JGit master, which has migrated to **bzlmod**. Gerrit `stable-3.13` still uses **WORKSPACE-mode** builds. That single divergence — bzlmod vs WORKSPACE — rules out direct consumption.

**Why `stable-7.4` + three reverts instead?** `stable-7.4` predates JGit's bzlmod migration (still on WORKSPACE, matching Gerrit `stable-3.13`), so the bzlmod axis requires zero JGit-side work. The three reverts undo the jakarta.servlet / Jetty 12 bumps that `stable-7.4` inherits from master — restoring the `javax.servlet 4` API surface that the `servlet-4` branch already provides via Jetty 12 ee8.

Once `stable-3.13` ages out and aligns on the modern stack, the fork branch goes away entirely.

---

## Four Options at a Glance

| # | Option | Divergence lives in | Maintained by | Gerrit-tree impact |
|---|---|---|---|---|
| 1 | http_archive + patches in `tools/jgit/` ([change 589501](https://gerrit-review.googlesource.com/c/gerrit/+/589501/2)) | Gerrit's tree | Gerrit team | `tools/jgit/` directory + WORKSPACE `patches=` field |
| 2 | Fork SHA with reverts pre-applied ([change 591581](https://gerrit-review.googlesource.com/c/gerrit/+/591581/1)) | Fork branch on GitHub | Gerrit team (rebases on fork) | Single WORKSPACE SHA pin |
| 3 | Backport bzlmod to stable-3.13 ([change 588622](https://gerrit-review.googlesource.com/c/gerrit/+/588622)) | Gerrit `stable-3.13` branch (bzlmod backport) | Gerrit team | Massive — bzlmod backport on a stable branch |
| 4 | New `servlet-4/stable-7.4` upstream branch | Upstream JGit | JGit team (parallel branch maintenance) | None |

> [!IMPORTANT]
> **Companion proposal for Options 1 & 2:** both approaches remove JGit as a tracked submodule, which regresses Eclipse/IntelliJ source navigation, debugger breakpoints, and `git bisect` across JGit history. [Gerrit change 591441](https://gerrit-review.googlesource.com/c/gerrit/+/591441) (*"Add developer story for build-time-patched JGit workflow"*) closes that gap with a documented IDE workflow against the Bazel-materialized JGit source. Lands alongside whichever of Options 1 or 2 is accepted.

---

## Per-Perspective Views

**🌿 JGit maintainer:** rejects **Option 4** — parallel `servlet-4/stable-7.N` branches per Gerrit stable release scale poorly. Indifferent to 1, 2, 3 (Gerrit's problem).

**⚙️ Gerrit maintainer:** rejects **Option 3** — backporting bzlmod to a stable branch is a high-risk infrastructure change on the line that's supposed to stay conservative. Indifferent-to-favorable on 1, 2, 4.

**🎩 Both hats together** *(my own position — I wear both)*: the real choice is between **Options 1 and 2**. Option 3 is out (wrong cost-benefit on a stable branch). Option 4 is out (unfair externality on the JGit team).

---

## Ranked Preference

| Rank | Option | Verdict |
|---|---|---|
| 🥇 | **#2** Fork SHA with pre-applied reverts | **Recommended** — semantically cleanest, smallest Gerrit-tree footprint, precedent-aligned with `rules_nodejs` on stable-3.14 |
| 🥈 | **#1** Build-time patches in `tools/jgit/` | Defensible fallback if the fork model is rejected on org-policy grounds |
| 🥉 | **#4** New JGit upstream branch | Ideal in principle but unfair to the JGit team (externalises maintenance for a Gerrit-stable-line-mismatch concern) |
| 🚫 | **#3** Backport bzlmod | Wrong cost-benefit on a stable branch; the underlying problem isn't tooling |

### Why #2 wins

- **Semantic cleanness.** Gerrit consumes JGit *as code*. The divergence isn't a Gerrit concern; it's a fork-of-JGit concern. The boundary between projects is honest.
- **Smallest Gerrit-tree footprint.** One WORKSPACE SHA. No `tools/jgit/` directory, no `patches=` declaration, no `patch_tool`, no patch-file maintenance.
- **Established precedent.** The `rules_nodejs` approach on `stable-3.14` already does exactly this pattern and is accepted.
- **Maintenance shape is identical to Option 1.** Three reverts to rebase per JGit SHA bump. Same work, different location — and the location matters.

### Concerns to address before landing #2

- **Fork ownership.** Currently a personal fork. For long-term sustainability the fork should sit under a Gerrit-team org, not an individual's GitHub account.
- **Discoverability.** A short comment in WORKSPACE pointing at the fork branch makes the patches discoverable without digging.
- **End-of-life cleanup.** When `stable-3.13` ages out, the fork should be retired alongside the WORKSPACE pin removal.

---

## TL;DR

**Pick Option 2.** It's structurally cleanest, precedent-aligned, and bounded. Option 1 is a defensible fallback. Option 4 would be the platonic ideal but is unfair to JGit. Option 3 is overkill.

Two mitigations worth landing alongside Option 2:

1. **Move the fork to a team-owned org** (not a personal account).
2. **Add a one-line comment in WORKSPACE** pointing at the fork branch.

Together they turn an ad-hoc personal patch into a documented Gerrit-team-maintained variant — which is what makes Option 2 sustainable across maintainer turnover.

---

> **→ [Read the full analysis with detailed reasoning](https://davido.github.io/gerrit-jgit-maintenance/)**
