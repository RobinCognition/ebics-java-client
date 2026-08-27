# Upgrade Playbook — Security & Language-Level Modernization

## Context

`ebics-java-client` is a Java library and CLI (`org.kopi.ebics.client.EbicsClient`, declared as the
jar `mainClass` in `pom.xml`) that talks to banks over EBICS. The build is currently pinned to a
legacy toolchain and several security-relevant dependencies are years behind:

| Area | Current state | Where |
| --- | --- | --- |
| Java language level | `maven-compiler-plugin` 3.1, `<source>1.7</source>` / `<target>1.7</target>` | `pom.xml` (~lines 150–158) |
| XMLBeans codegen | `xmlbeans-maven-plugin` 2.3.3, `<javaSource>1.8</javaSource>`, schemas in `src/main/xsd` | `pom.xml` (~lines 129–158) |
| Crypto | `bcprov-jdk15on` 1.68, `bcpkix-jdk15on` 1.68, `org.gnu:gnu-crypto` 2.0.1 | `pom.xml` dependencies |
| XML stack | `xmlbeans` 3.1.0, `xalan` 2.7.2, `xmlsec` 1.3.0, `xercesImpl` 2.12.2 | `pom.xml` dependencies |
| Transport / misc | `httpclient` 4.5.13, `commons-codec` 1.14, `commons-logging` 1.2, `jdom` 2.0.2 | `pom.xml` dependencies |
| Deserialization | native Java `ObjectInputStream.readObject()` to restore `Bank`/`Partner`/`User` | `src/main/java/org/kopi/ebics/client/EbicsClient.java` (~lines 246–257) |
| HTTPS transport | Apache HttpClient with default TLS configuration | `src/main/java/org/kopi/ebics/client/HttpRequestSender.java` (~lines 75–117) |

The plan below is deliberately phased so that each step is independently buildable, independently
verifiable, and independently revertable. This document is the plan only — it changes no build or
source files.

## Table of contents

- [Prerequisites](#prerequisites)
- [How to use this playbook](#how-to-use-this-playbook)
- [Phase 0 — Baseline & tooling](#phase-0--baseline--tooling)
- [Phase 1 — Upgrade build plugins](#phase-1--upgrade-build-plugins)
- [Phase 2 — Raise Java language level](#phase-2--raise-java-language-level)
- [Phase 3 — Upgrade security-critical dependencies](#phase-3--upgrade-security-critical-dependencies)
- [Phase 4 — Harden deserialization & transport](#phase-4--harden-deserialization--transport)
- [Notes & caveats](#notes--caveats)

## Prerequisites

- **JDKs available locally.** A JDK 8 to reproduce the current baseline, plus the target JDK for
  Phase 2 (JDK 17 recommended). Note that JDK 20+ refuses `source`/`target` 1.7, so the pre-Phase-2
  build must run on an older JDK.
- **Maven 3.x** with network access to Maven Central. Central occasionally rate-limits (HTTP 429);
  configure a mirror in a `settings.xml` outside the repo if that happens.
- **A bank test/sandbox EBICS profile** (host id, partner id, user id, bank URL) in
  `$HOME/ebics/client/ebics.txt` — see `ebics-template.txt` and `HOWTO.md`. Without it, only
  compile-level and CLI-help-level verification is possible; INI/HIA/HPB and file transfer flows
  cannot be exercised end to end.
- **Optional:** Docker, if the container image (`Dockerfile`, currently `maven:3-jdk-8` +
  `openjdk:8-alpine`) is part of your delivery surface — it must be bumped alongside Phase 2.
- **Write access** to the repo and the ability to open one PR per phase.

## How to use this playbook

1. Work the phases **in order**. Each phase assumes the previous one is merged and green.
2. **One PR per phase** (Phase 3 may reasonably be split into one PR per dependency family). Small
   PRs keep the blast radius of an EBICS protocol regression small.
3. Before starting a phase, re-read its **Objective** so scope creep is obvious; do not fold
   dependency bumps into a plugin-only or language-level-only change.
4. After each phase, run its **Verification** section in full and paste the output into the PR.
   A phase is not done until the build is green and the flows listed still work.
5. If a phase fails verification, revert that phase's commit rather than patching forward — the
   phases are ordered so that each one is a safe stopping point.
6. Treat the **Risk level** as a signal for how much functional (bank-facing) testing is required,
   not just how large the diff is.

Reference build commands used throughout:

```bash
# baseline build (legacy toolchain)
JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64 mvn -B -DskipTests clean package
# tests
JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64 mvn -B test
# CLI smoke test
mvn exec:java -Dexec.mainClass=org.kopi.ebics.client.EbicsClient -Dexec.args="--help"
```

---

## Phase 0 — Baseline & tooling

### Objective

Establish an authoritative, reproducible baseline — build success and a CVE inventory — **before**
any version is changed, so that every later phase can be judged against real data instead of
assumptions.

### Tasks (files/versions to change)

- `pom.xml`: add a dependency vulnerability scanner in the `<build><plugins>` section, e.g. the
  OWASP `dependency-check-maven` plugin (current 9.x/12.x line), configured to emit HTML + JSON
  reports into `target/`. Optionally set `failBuildOnCVSS` high (e.g. 11 = never fail) initially so
  the baseline run is informational only.
- Capture and record in the PR description:
  - `mvn -version` and `java -version` output,
  - a successful `mvn -B -DskipTests clean package`,
  - `mvn dependency:tree` (to expose transitive versions the direct declarations hide),
  - the dependency-check report summary (count of findings by severity, per artifact).
- CI: `.github/workflows/maven.yml` already exists (`Java CI with Maven`, JDK 8 via
  `actions/setup-java@v2`, `mvn -B compile package`, plus an OSSRH deploy step on `master`). Add a
  scan step (`mvn dependency-check:check` or `org.owasp:dependency-check-maven:check`) to that
  workflow, and consider uploading the report as a build artifact. Keep the deploy step untouched.
- Note that no Maven wrapper (`mvnw`) and no `.travis.yml` exist; if a wrapper is desired, adding
  `mvn wrapper:wrapper` output is an optional sub-task here (it pins the Maven version for CI and
  for the `Dockerfile`).
- Record the NVD API key requirement: recent dependency-check versions need an NVD API key for
  reasonable data-feed update times. Store it as a repository secret (e.g. `NVD_API_KEY`) and pass
  it via `-DnvdApiKey=${NVD_API_KEY}` rather than committing it.

### Risk level

**Low.** Additive only — no dependency or language-level change. The main risks are CI runtime
increase (the NVD feed download is slow on a cold cache) and a noisy first report.

### Verification (build + functional flow)

- `mvn -B -DskipTests clean package` succeeds on the baseline JDK 8 toolchain.
- XMLBeans generation still runs: `target/generated-sources/xmlbeans/org/kopi/ebics/schema/`
  contains the generated `org.kopi.ebics.schema.*` classes (~1200+ files).
- `mvn dependency-check:check` completes and produces a report in `target/`.
- CI run on the PR is green and the scan step's report artifact is downloadable.
- Functional: `--help` CLI smoke test still prints the option list.

---

## Phase 1 — Upgrade build plugins

### Objective

Modernize the build plugins only, so that any breakage is attributable to plugin behavior and not
to a language-level or dependency change.

### Tasks (files/versions to change)

- `pom.xml` (~lines 150–158): `maven-compiler-plugin` `3.1` → current `3.13.x`/`3.14.x`.
- `pom.xml` (~lines 129–148): `xmlbeans-maven-plugin` `2.3.3` → a current compatible release.
  Note the coordinate question: the long-standing plugin is `org.codehaus.mojo:xmlbeans-maven-plugin`
  (2.3.3), while newer XMLBeans 3.x/5.x codegen is commonly driven by
  `org.apache.xmlbeans:xmlbeans-maven-plugin`. Decide explicitly which coordinate to move to and
  confirm it is compatible with the declared `org.apache.xmlbeans:xmlbeans` runtime version;
  if the plugin move forces an `xmlbeans` runtime bump, defer that bump to Phase 3 if possible, or
  document the coupling.
- Keep `<source>1.7</source>`/`<target>1.7</target>` and `<javaSource>1.8</javaSource>` **unchanged**
  in this phase.
- Leave `schemaDirectory` (`src/main/xsd`) and `defaultXmlConfigDir` (`src/main/xsd/config`) as is.
- While here, consider (optional, same risk class) bumping other stale build plugins:
  `maven-eclipse-plugin` 2.10, `maven-source-plugin` 3.0.0 — or explicitly defer them.

### Risk level

**Low–medium.** Plugin-only, but XMLBeans codegen is the fragile part: a plugin change can alter
generated class names, the `.xsb` metadata layout, or how the config in `src/main/xsd/config` is
applied. Compilation of `org.kopi.ebics.xml.*` is the canary.

### Verification (build + functional flow)

- Clean build from scratch: `mvn -B -DskipTests clean package` (verify no `target/` reuse).
- Generated sources check: `org.kopi.ebics.schema.*` classes are regenerated under
  `target/generated-sources/xmlbeans/`; compare the file count and a diff of package/class names
  against the Phase 0 baseline. Any missing or renamed generated type is a blocker.
- `mvn -B test` passes.
- Functional: `--help` CLI smoke test; then, with a configured profile, verify document creation
  paths in `org.kopi.ebics.xml` still marshal (e.g. an INI/HIA request build) — at minimum confirm
  the KeyManagement and FileTransfer classes compile against the regenerated schema classes.

---

## Phase 2 — Raise Java language level

### Objective

Move the compiled language level off the end-of-life Java 7 target onto a supported LTS, so later
security work (serialization filters, modern TLS APIs, current Bouncy Castle artifacts) is available.

### Tasks (files/versions to change)

- `pom.xml` (~lines 150–158): replace `<source>1.7</source>`/`<target>1.7</target>` with
  `<release>17</release>` (preferred with a modern compiler plugin) or `<source>17</source>` /
  `<target>17</target>`.
- `pom.xml` (~lines 129–148): align `xmlbeans-maven-plugin` `<javaSource>` with the chosen level
  (note the plugin historically accepts a limited set of values, e.g. `1.5`/`1.8`; if it will not
  accept `17`, leave it at its highest accepted value and document why the two differ).
- Consider adding `maven.compiler.release` / `project.build.sourceEncoding` properties so the level
  is declared in one place.
- Update the toolchain used by consumers of the build:
  - `.github/workflows/maven.yml`: `java-version: '8'` → the chosen LTS (and bump
    `actions/checkout@v2` / `actions/setup-java@v2` to current majors while there).
  - `Dockerfile`: `maven:3-jdk-8` and `openjdk:8-alpine` → matching LTS base images
    (e.g. `maven:3-eclipse-temurin-17` and `eclipse-temurin:17-jre-alpine`).
  - `HOWTO.md`: the "minimum java 1.88" line is stale and should state the new minimum.
- Fix any newly surfaced compile errors/warnings caused by stricter modern `javac` (removed internal
  APIs, `sun.*` usage, illegal reflective access).

#### Decision point — target LTS

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| **Java 11** | Smallest jump from 1.7/8; almost certainly compiles unchanged | Already in extended-support territory; still lacks newer APIs; likely a second migration later | Fallback / intermediate step only |
| **Java 17** | Broad library and BC support; strong encapsulation is manageable; long support horizon | Strong encapsulation of JDK internals may break reflective code; security manager deprecation warnings | **Default choice** |
| **Java 21** | Longest runway | Highest chance of friction with the aged XML stack (XMLBeans/Xalan/xmlsec) before Phase 3 lands | Consider only after Phase 3 |

Default to **Java 17**. If the build fails hard, land Java 11 first as an intermediate commit, then
repeat this phase for 17.

### Risk level

**Medium.** Behavioral changes are plausible even when compilation succeeds: default charset,
stricter reflection, TLS defaults, and XML factory resolution all differ between 8 and 17. The old
XML libraries (Xalan 2.7.2, xmlsec 1.3.0, XMLBeans 3.1.0) are the most likely source of runtime —
not compile-time — surprises, which is why Phase 3 follows immediately.

### Verification (build + functional flow)

- `mvn -B -DskipTests clean package` on the target JDK; then `mvn -B test`.
- Confirm bytecode level, e.g. `javap -verbose -cp target/classes org.kopi.ebics.client.EbicsClient | grep major`
  (61 = Java 17).
- Confirm XMLBeans codegen still produces the full `org.kopi.ebics.schema.*` set.
- CLI entry-point smoke test per `HOWTO.md`:
  - `mvn exec:java -Dexec.mainClass=org.kopi.ebics.client.EbicsClient -Dexec.args="--help"`
  - `... -Dexec.args="--create"` (user/key creation, letters written to
    `./client/users/<userId>/letters/`)
  - `... -Dexec.args="--ini --hia"` and `... -Dexec.args="--hpb"` against a test bank profile
  - `... -Dexec.args="--sta -o sta.txt"` for a download flow
- Docker: `docker build . -t client && docker run client --help` if the image is in scope.

---

## Phase 3 — Upgrade security-critical dependencies

### Objective

Remove known-vulnerable and unmaintained libraries from the runtime classpath, one family at a time,
with a build + flow check after each bump.

### Tasks (files/versions to change)

All changes are in the `<dependencies>` block of `pom.xml`. Recommended order (crypto first, XML
second, plumbing last):

1. **Bouncy Castle** — `org.bouncycastle:bcprov-jdk15on:1.68` and
   `org.bouncycastle:bcpkix-jdk15on:1.68` → the current `bcprov-jdk18on` / `bcpkix-jdk18on`
   artifacts. This is an **artifactId change**, not just a version bump, and the `jdk18on` line
   requires Java 8+. Expect small API adjustments in the certificate/key code
   (`src/main/java/org/kopi/ebics/certificate`, e.g. X.509 builders, `KeyStore` handling, provider
   registration). Verify the `A005` / `X002` / `E002` key generation and the initialization letter
   fingerprints are byte-for-byte unchanged for a fixed input where possible.
2. **XML stack**
   - `xalan:xalan:2.7.2` — CVE-2022-34169 (integer truncation in the XSLT compiler) is the reason
     this is on the list. Preferred remediation: **remove** the dependency and rely on the JDK's
     built-in Xalan-derived `TransformerFactory`; only if a hard dependency exists, move to the
     patched release line. Grep for `javax.xml.transform` / explicit factory selection before
     removing.
   - `org.apache.xmlbeans:xmlbeans:3.1.0` → current 5.x. Must be coordinated with the
     `xmlbeans-maven-plugin` choice from Phase 1 — codegen and runtime versions must match.
   - `xerces:xercesImpl:2.12.2` → evaluate **removal**; the JDK's built-in parser has superseded
     standalone Xerces for most uses, and keeping it on the classpath can shadow the JDK parser and
     change entity/DTD-handling defaults.
   - `xml-security:xmlsec:1.3.0` → migrate to the maintained coordinates
     `org.apache.santuario:xmlsec` (current 3.x/4.x). The old `xml-security:xmlsec` groupId is
     abandoned. Signature generation/verification (C14N and EBICS `AuthSignature`) must be
     re-verified carefully after this.
3. **`org.gnu:gnu-crypto:2.0.1`** — abandoned upstream. A grep of
   `src/main/java/org/kopi/ebics` for `gnu`/`gnu.crypto` found **no source usages**, so this is
   very likely a removable leftover (possibly a transitive/legacy requirement of the old xmlsec).
   Task: confirm with `mvn dependency:tree` that nothing depends on it, remove the declaration, and
   rebuild. If a runtime `NoClassDefFoundError` appears during the crypto flows, map the needed
   primitive onto Bouncy Castle or the JCE instead of restoring the dependency.
4. **Plumbing**
   - `org.apache.httpcomponents:httpclient:4.5.13` → latest 4.5.x (drop-in), or plan a separate
     migration to `org.apache.httpcomponents.client5:httpclient5` (package rename, larger change —
     keep it out of this phase).
   - `commons-codec:commons-codec:1.14` → current 1.1x.
   - `org.jdom:jdom:2.0.2` → note the current coordinates are `org.jdom:jdom2`; move to the
     maintained `jdom2` artifact and adjust imports if the groupId/artifactId change requires it.
   - `commons-logging:commons-logging:1.2` → current, or consider routing through slf4j (out of
     scope here; document only).

After **each** individual bump: commit separately, run the build, and re-run the Phase 0 scanner to
confirm the finding actually disappeared from the report.

### Risk level

**High.** This phase touches the crypto and XML-signature paths that EBICS correctness depends on.
A silently different canonicalization, digest, or padding choice produces requests the bank rejects
(non-zero `ReturnCode`) rather than a build failure. The xmlsec and Bouncy Castle steps are the two
highest-risk items; treat them as their own PRs.

### Verification (build + functional flow)

- Per bump: `mvn -B -DskipTests clean package` and `mvn -B test`, plus `mvn dependency:tree` to
  confirm no duplicate/conflicting versions were dragged in.
- Re-run `mvn dependency-check:check` and diff against the Phase 0 baseline — findings should
  strictly decrease.
- **KeyManagement flow** compiles and runs: `--create`, `--ini`, `--hia`, `--hpb`; the generated
  `A005Letter.txt` / `E002Letter.txt` / `X002Letter.txt` are produced and the HPB bank-key
  fingerprints match the bank's documentation.
- **FileTransfer flow** compiles and runs: a download (`--sta -o sta.txt`) and, where the test
  profile permits, an upload (e.g. a CCT using `pain001-template-ch.xml`), including the segmented
  (>1 MB) path so `org.kopi.ebics.io` segmentation/compression/encryption is exercised.
- Confirm all EBICS responses return `ReturnCode` `000000` where the flow is expected to succeed.

---

## Phase 4 — Harden deserialization & transport

### Objective

Close the two application-level security gaps that dependency upgrades do not address: unrestricted
native Java deserialization of persisted state, and unpinned TLS configuration on the bank
connection.

### Tasks (files/versions to change)

**Deserialization** — `src/main/java/org/kopi/ebics/client/EbicsClient.java` (~lines 246–257) reads
persisted state with `ObjectInputStream.readObject()` for `Bank`, and passes `ObjectInputStream`
into the `Partner`/`User` constructors; the streams come from
`configuration.getSerializationManager().deserialize(...)`.

- Short term (recommended first step): apply a JDK serialization filter. Set an
  `ObjectInputFilter` on each stream (`ObjectInputStream.setObjectInputFilter`, Java 9+; available
  in 8u121+ as `sun.misc.ObjectInputFilter`) allow-listing only `org.kopi.ebics.client.Bank`,
  `org.kopi.ebics.client.Partner`, `org.kopi.ebics.client.User` and the JDK value types they
  legitimately contain, with `maxdepth`/`maxrefs`/`maxbytes` limits and a `!*` catch-all reject.
  The cleanest placement is inside `SerializationManager.deserialize(...)` so every call site is
  covered, rather than only the `EbicsClient` call sites.
- Alternative / additional: a process-wide filter via `-Djdk.serialFilter=...` documented in
  `HOWTO.md` and set in the `Dockerfile` entrypoint — cheap defense in depth, but do not rely on it
  alone since it is caller-controlled.
- Longer term (document as a follow-up, not necessarily this phase): migrate persisted state away
  from native Java serialization to an explicit, schema'd format (e.g. properties/JSON for
  `Bank`/`Partner`/`User` metadata, with keys staying in a `KeyStore`). Requires a migration path
  for existing `$HOME/ebics/client` state directories — a one-shot converter that reads the legacy
  streams under the strict filter and writes the new format.
- Also verify the trust boundary: state files live in the user's home directory, so the realistic
  threat is a tampered or shared state directory. Note this explicitly in the PR so the severity is
  understood rather than assumed.

**Transport** — `src/main/java/org/kopi/ebics/client/HttpRequestSender.java` (~lines 75–117) builds
the client with `HttpClientBuilder.create().setDefaultRequestConfig(...)` and inherits all TLS
defaults; timeouts are set to 300 s and an optional HTTP proxy with basic credentials is configured.

- Set an explicit `SSLConnectionSocketFactory` with `TLSv1.2`/`TLSv1.3` only, a modern cipher
  allow-list, and default hostname verification (do **not** weaken it); wire it via
  `builder.setSSLSocketFactory(...)`.
- Confirm certificate/hostname verification is never disabled anywhere in the code or config, and
  that `bank.url` is required to be `https://`.
- Review the proxy path: credentials come from plaintext `ebics.txt` properties
  (`http.proxy.user`/`http.proxy.password`) — document this and ensure they are never logged.
- Optionally scope the socket/connect timeouts (300 s each) and add a response timeout.

### Risk level

**Medium.** The filter is a deliberate deny-by-default control: too tight an allow-list breaks
loading existing users at runtime (not at compile time), so it must be tested against a real
pre-existing state directory. Pinning TLS can break connectivity to banks with older endpoints —
verify against the actual bank host before merging.

### Verification (build + functional flow)

- `mvn -B -DskipTests clean package` and `mvn -B test`.
- Deserialization: with an **existing** `$HOME/ebics/client` state directory created before the
  change, run a command that loads state (e.g. `--sta`) and confirm `Bank`/`Partner`/`User` load
  successfully (`user.load.success` log message). Then confirm a filter rejection is logged/thrown
  for a state file containing a disallowed class (craft one in a scratch directory).
- Round-trip: `--create` a fresh user, restart the CLI, and confirm the new state loads under the
  filter.
- Transport: run `--hpb` and `--sta` against the bank endpoint and confirm success; capture the
  negotiated protocol/cipher with `-Djavax.net.debug=ssl:handshake` and confirm TLS 1.2 or 1.3.
- Confirm the proxy path still works if used (`http.proxy.host` set).

---

## Notes & caveats

- **Target versions and CVE claims in this document must be validated against the Phase 0 scanner
  output.** No CVE-to-version matching was executed against this repository while writing this
  playbook: the version numbers above were read from `pom.xml`, but "current"/"latest" targets and
  vulnerability attributions (including CVE-2022-34169 for Xalan 2.7.2) are stated from general
  knowledge and are not verified findings for this build. Treat the dependency-check report — not
  this document — as the authoritative list of what must be upgraded and to which version.
- Version numbers move; resolve the actual "current" release at execution time (`mvn
  versions:display-dependency-updates` and `versions:display-plugin-updates` are useful here)
  rather than hard-coding the examples above.
- **Coordinate changes are hidden traps.** `bcprov-jdk15on` → `bcprov-jdk18on`,
  `xml-security:xmlsec` → `org.apache.santuario:xmlsec`, `org.jdom:jdom` → `org.jdom:jdom2`, and
  possibly `org.codehaus.mojo` → `org.apache.xmlbeans` for the XMLBeans plugin are renames, so a
  naive version bump will either fail to resolve or silently leave the old artifact on the
  classpath alongside the new one. Always re-check `mvn dependency:tree` after these.
- **Compile-clean ≠ protocol-correct.** EBICS failures from XML canonicalization, digest, or
  encryption differences appear only as a non-zero `ReturnCode` from the bank. No amount of
  building proves Phase 3 correct; a real (test) bank round trip does.
- **The test surface is thin.** There is no lint/formatter config and test coverage is minimal, so
  verification leans on the CLI flows in `HOWTO.md`. Adding regression tests around
  `org.kopi.ebics.xml` document generation (golden-file comparison of generated request XML) before
  Phase 3 would materially de-risk the whole plan and is worth its own PR.
- **Toolchain consumers must move together.** `.github/workflows/maven.yml` (JDK 8) and the
  `Dockerfile` (`maven:3-jdk-8`, `openjdk:8-alpine`) both pin Java 8; leaving either behind in
  Phase 2 produces a build that passes locally and fails in CI or ships a broken image.
- JDK 20+ rejects `source`/`target` 1.7 outright, so the pre-Phase-2 baseline must be reproduced on
  an older JDK (8 or 17).
- Maven Central rate-limiting (HTTP 429) has been observed from build machines here; use a mirror
  configured outside the repo if downloads start failing mid-phase.
- The `.github/workflows/maven.yml` deploy step publishes on pushes to `master` using OSSRH/GPG
  secrets. Any phase that changes the artifact's Java level or coordinates is also a **release**
  change for downstream consumers — decide on version numbering (a minor vs. major bump) before
  Phase 2 merges.
- This playbook intentionally makes no code or build changes; `pom.xml` and all sources are
  untouched by the commit that adds it.
