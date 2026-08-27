# Phased Upgrade Plan: Java language level & dependency security

Status: Phase 0 and Phase 1 are implemented in `pom.xml` (see "Implemented so far"). Phases 2-4 are
documented here for maintainer review and are **not** implemented.

## Why

The build is pinned to a 2013-era toolchain and a partly abandoned dependency set:

| Item | Current | Problem |
| --- | --- | --- |
| `maven-compiler-plugin` | 3.1 | 2013 release, long past support |
| language level | `1.7` | **rejected by JDK 20+** (`source/target 7` was removed there; JDK 17 still accepts it with an obsolescence warning), and blocks `ObjectInputFilter` and other modern security APIs |
| `xmlbeans-maven-plugin` | `org.codehaus.mojo` 2.3.3 | last release 2008; generates code against XMLBeans **2.4.0** regardless of the runtime `xmlbeans` version |
| `xalan` | 2.7.2 | CVE-2022-34169 (integer truncation in the XSLTC bytecode compiler) fixed in 2.7.3 |
| `xml-security:xmlsec` | 1.3.0 | 2007 artifact under a dead coordinate; superseded by `org.apache.santuario:xmlsec` |
| Bouncy Castle | `*-jdk15on` 1.68 | `jdk15on` line is discontinued; 1.68 predates several BC advisories |
| `org.gnu:gnu-crypto` | 2.0.1 | abandoned (2004), and **not referenced anywhere in `src/main/java`** |
| `xmlbeans`, `xercesImpl`, `httpclient`, `commons-codec`, `jdom` | 3.1.0 / 2.12.2 / 4.5.13 / 1.14 / 2.0.2 | all behind current releases |

Two security-relevant code paths compound this:

* `EbicsClient.loadUser` (`src/main/java/org/kopi/ebics/client/EbicsClient.java` ~246-257) restores
  `Bank`/`Partner`/`User` state with native Java deserialization (`ObjectInputStream.readObject()`),
  with no class allow-list. Anyone who can write into the serialization directory
  (`DefaultSerializationManager`, `<serialization dir>/<name>.cer`) gets a gadget-chain entry point.
* `HttpRequestSender.send` (~75-117) builds an Apache HttpClient 4.5 client with default TLS
  settings; the protocol floor is whatever the running JDK defaults to rather than something the
  client asserts.

> **Version caveat.** The target versions below are the current releases at the time of writing and
> the CVE references are from public advisories; **no scanner has been run against this repository
> yet**. Treat every version number here as a proposal to be confirmed against the Phase 0
> OWASP Dependency-Check report, which is the authoritative CVE baseline for this codebase.

## Repository facts established for this plan

* CI: `.github/workflows/maven.yml` — single job on **JDK 8** (`adopt`), `mvn -B compile package`,
  then `mvn deploy` on `master`. This is the only CI config; there is **no** `.travis.yml`,
  no `mvnw`/`gradlew` wrapper, and no `.mvn/` directory.
* `Dockerfile`: `maven:3-jdk-8` build stage, `openjdk:8-alpine` runtime stage — so the Java level is
  pinned in three places (pom, CI workflow, Dockerfile) and Phase 2 must change all three.
* Codegen: `src/main/xsd` (EBICS H003/H004/H005 + S001 schemas) with `src/main/xsd/config/config.xsdconfig`,
  producing `org.kopi.ebics.schema.*`.
* `org.gnu:gnu-crypto` has zero source references (`grep -ri gnu src/` only matches licence headers).
* JDOM: the code imports `org.jdom2.*`; the declared artifact `org.jdom:jdom:2.0.2` is the old
  coordinate of JDOM2, so moving to `org.jdom:jdom2` is a coordinate change only.
* `xmlsec` usage is narrow: `org.apache.xml.security.Init`, `c14n.Canonicalizer`,
  `utils.IgnoreAllErrorHandler`, `transforms.TransformationException` in `utils/Utils.java`,
  `xml/SignedInfo.java`, `client/EbicsClient.java`.

---

## Phase 0 — Baseline & tooling (risk: none — no production code or dependency changes)

**Objective:** produce an authoritative CVE baseline and a reproducible build baseline *before*
anything changes, so later phases can be judged against it.

Tasks:

1. `pom.xml`: add `org.owasp:dependency-check-maven` inside a `security-scan` profile so ordinary
   builds are unaffected (an unprimed NVD cache download takes many minutes).
   Version choice: **10.0.4** — the last release supporting JDK 8, which is what CI and the
   Dockerfile currently run. Bump to 12.x/13.x in Phase 2 once the JDK is 11+.
2. `.github/workflows/dependency-check.yml`: scheduled (weekly) + `workflow_dispatch` scan that
   uploads the report as an artifact. Deliberately **not** wired into the PR build: an NVD API key is
   required for usable download rates, and a missing/expired key must not turn PR CI red.
   Requires repo secret `NVD_API_KEY` (free from https://nvd.nist.gov/developers/request-an-api-key).
3. Record the baseline locally:
   ```bash
   mvn -version                       # record Maven + JDK in the PR/issue
   mvn -B -DskipTests clean package    # must succeed on JDK 8
   # -Dgpg.skip=true is required: maven-gpg-plugin binds `sign` to the verify phase for every
   # build, so verify fails before dependency-check runs unless a signing key is present.
   mvn -Psecurity-scan -DskipTests -Dgpg.skip=true -Dnvd.api.key=$NVD_API_KEY verify
   ```
4. Commit the resulting report summary (or attach it to the tracking issue) as *the* baseline list of
   CVEs. Every later phase is verified by re-running the scan and diffing against it.

**Verification:** `mvn -B -DskipTests clean package` still succeeds on JDK 8 and produces
`target/ebics-cli-<version>.jar` plus `target/lib/`; the scan produces
`target/dependency-check-report.html` / `.json`.

Baseline recorded for this change (Ubuntu 22.04 box, not CI):

* `Apache Maven 3.6.3`; `mvn -B -DskipTests clean package` **succeeds on both JDK 8**
  (`1.8.0_502`) **and JDK 17** (`17.0.13`), generating 1230 XMLBeans classes under
  `target/generated-sources/xmlbeans/org/kopi/ebics/schema/{h000,h003,h004,h005,s001,s002,xmldsig}`.
  JDK 17's javac still accepts `-source 7` (with an obsolescence warning); it is **JDK 20** that
  removed source/target 7, so the language level — not a hard build break yet — becomes one on the
  next JDK upgrade. Existing deprecation/unchecked and Javadoc warnings are the only noise.
* Caveat on reproducing this: Maven Central rate-limited (HTTP 429) plain resolution from this box,
  so the baseline was taken through a Central mirror. A 429 during plugin resolution is an
  infrastructure symptom, not a project defect.
* The OWASP scan itself has **not** been run yet: `dependency-check` needs an `NVD_API_KEY` to
  download the NVD feed at a usable rate, which the maintainer must add (see Phase 0 task 2).
  Producing that first report is the immediate next step and it, not this table, is authoritative.

## Phase 1 — Upgrade build plugins only (risk: low)

**Objective:** modernize the plugins with the language level held constant, so any breakage is
attributable to the plugins alone.

Tasks:

1. `maven-compiler-plugin` **3.1 → 3.14.0**. Keep `<source>1.7</source>`/`<target>1.7</target>`
   in this phase. (JDK 8's javac still accepts 7 with an "obsolete" warning.)
2. `xmlbeans-maven-plugin`: **there is no newer release** — `org.codehaus.mojo:xmlbeans-maven-plugin`
   2.3.3 (2008) is the latest ever published, and XMLBeans 5.x replaced it with a plugin shipped
   *inside* the `org.apache.xmlbeans:xmlbeans` artifact (goal `xmlbeans:compile`). Because that
   migration changes configuration keys (`schemaDirectory` → `sourceDir`, `defaultXmlConfigDir` →
   `xmlConfigs`), adds a `repackage`d metadata package, and regenerates all schema classes, it is
   **deferred to Phase 3**, where it is coupled to the `xmlbeans` runtime bump that actually
   requires it.
   What is done here instead, at near-zero risk: pin the plugin's own codegen dependency to the
   runtime version (`org.apache.xmlbeans:xmlbeans:3.1.0` as a `<plugin><dependencies>` entry) so
   generated sources and runtime library stop being two different XMLBeans majors (the plugin
   otherwise silently drags in xmlbeans 2.4.0 — see XMLBEANS-593).

**Verification:** clean build on JDK 8; confirm `target/generated-sources/xmlbeans/org/kopi/ebics/schema/`
is regenerated from `src/main/xsd` and that the jar manifest still has
`Main-Class: org.kopi.ebics.client.EbicsClient`.

## Phase 2 — Raise the Java language level (risk: medium)

**Objective:** get onto a supported LTS so the project is buildable by current JDKs and can use
modern security APIs.

**Decision point — 11 vs 17 vs 21.** Default recommendation: **17**.

| Target | For | Against |
| --- | --- | --- |
| 11 | smallest jump; every current dependency supports it; keeps `xmlsec` 3.x in reach | already past its most active phase; another migration later; misses records/sealed types |
| **17 (recommended)** | LTS with long support; required by `xmlsec` 4.x; `ObjectInputFilter` (Java 9+) available for Phase 4; supported by all target dependency versions | strong encapsulation (JEP 403) can break reflective access — a real risk for XMLBeans/Xalan/Xerces-era libraries; needs the Docker base image and CI JDK changed too |
| 21 | newest LTS | least ecosystem mileage for these legacy XML libraries; no capability this project needs that 17 lacks |

Tasks:

1. `pom.xml`: replace `<source>/<target>` with `<maven.compiler.release>17</maven.compiler.release>`
   (`release` is stricter and correct: it also validates against the JDK 17 API surface).
2. Drop the now-meaningless `<javaSource>1.8</javaSource>` from the XMLBeans plugin, or set it to
   `1.8` explicitly if generated code must stay 8-compatible for downstream consumers of the
   published `ebics-cli` artifact. **Consumer impact:** this library is published to Maven Central,
   so raising the bytecode level is a breaking change for any consumer still on Java 8 — worth a
   minor/major version bump and a release note.
3. `.github/workflows/maven.yml`: `java-version: '8'` → `'17'`, and `distribution: adopt` →
   `temurin` (`adopt` is deprecated in `actions/setup-java`). Also bump `actions/checkout@v2` →
   `@v4` and `actions/setup-java@v2` → `@v4` (v2 runs on the removed Node 12/16 runtimes).
4. `Dockerfile`: `maven:3-jdk-8` → `maven:3-eclipse-temurin-17`, `openjdk:8-alpine` →
   `eclipse-temurin:17-jre-alpine`.
5. Bump `dependency-check-maven` to the current 12.x/13.x line (JDK 11+) now that the JDK allows it.

**Verification:** `mvn -B clean package` on JDK 17; then the CLI smoke tests from `HOWTO.md` against
a scratch config directory — `--create` (config bootstrap), `--ini`/`--hia` (key generation and
serialization write path), and `--letters`. A signed-request path (`--sta`) needs bank credentials, so
if no test bank is available, state that explicitly in the PR rather than implying it was exercised.
Watch specifically for `InaccessibleObjectException`/`IllegalAccessError` from XMLBeans, Xalan or
Xerces reflection under JEP 403; if any appear, the fallback is targeted `--add-opens` in the
`maven-surefire-plugin`/jar manifest, or Java 11 as an intermediate step.

## Phase 3 — Upgrade security-critical dependencies (risk: medium-high)

**Objective:** eliminate the known-vulnerable and abandoned crypto/XML stack. Do this as **one
commit per dependency** so a regression bisects cleanly.

Order matters — crypto and XML first, then transport, then utilities:

1. **Bouncy Castle** `bcprov-jdk15on`/`bcpkix-jdk15on` 1.68 → `bcprov-jdk18on`/`bcpkix-jdk18on`
   (1.80 at time of writing). The artifactId changes (`jdk15on` → `jdk18on`, Java 8+ baseline) and
   both must move in lockstep — mixing `jdk15on` and `jdk18on` on one classpath yields duplicate
   `org.bouncycastle` packages. API surfaces to re-check: `certificate/X509Generator.java`
   (`SubjectPublicKeyInfo`, `ASN1InputStream`, `X509v3CertificateBuilder`) and
   `certificate/KeyStoreManager.java` (`PEMParser`, `JcaPEMKeyConverter`). Deprecated `ASN1InputStream`
   read patterns are the most likely compile fixes.
2. **`xalan` 2.7.2 → 2.7.3** (CVE-2022-34169). Better still: check whether Xalan is needed at all —
   the JDK bundles a Xalan fork as its `TransformerFactory`, and this codebase does no `javax.xml.transform`
   work outside the xmlsec canonicalization path. Removing it is the strongest fix; 2.7.3 is the
   safe fallback.
3. **`xml-security:xmlsec` 1.3.0 → `org.apache.santuario:xmlsec`** (4.0.x on Java 17, 3.0.x on Java 11).
   This is a groupId change plus a 17-year version jump; the four imported types still exist, but
   `Init.init()` semantics and canonicalization algorithm registration have changed. Verify the
   canonicalized `SignedInfo` bytes are byte-identical before/after — an EBICS signature that
   canonicalizes differently is accepted by no bank. Capture a canonicalization fixture test first.
4. **`xmlbeans` 3.1.0 → 5.3.0**, together with migrating codegen to the XMLBeans 5.x built-in Maven
   plugin (`org.apache.xmlbeans:xmlbeans` with `<extensions>true</extensions>`, goal `compile`),
   because the codehaus 2.3.3 plugin generates 2.4.0-era sources that do not compile against 5.x
   (references to the removed `org.apache.xmlbeans.xml.stream` package — XMLBEANS-593). Map
   `schemaDirectory` → `sourceDir`, `defaultXmlConfigDir` → `xmlConfigs`, and set `repackage` so the
   metadata package stays stable. Expect all `org.kopi.ebics.schema.*` classes to be regenerated.
5. **`xercesImpl` 2.12.2** — already the latest release; the real question is whether it is needed.
   The JDK's built-in parser has been sufficient since Java 7 and the code has no `org.apache.xerces`
   imports. Prefer removal, verified by a full build plus an XML round-trip through `DefaultEbicsRootElement`.
6. **`org.gnu:gnu-crypto` 2.0.1 → removed.** Confirmed unused: no `gnu.*` import anywhere in
   `src/main/java`. Delete the dependency and rebuild; nothing to migrate. (If a downstream consumer
   relies on it leaking onto the classpath transitively, that is their dependency to declare.)
7. **`httpclient` 4.5.13 → 4.5.14** (drop-in). A move to HttpClient 5.x (`httpclient5` 5.6.x) is a
   separate, larger change — new package names and a rewritten builder API in `HttpRequestSender` —
   and should be its own follow-up, not part of this phase.
8. **`commons-codec` 1.14 → current (1.19.0+)**, **`jdom` → `org.jdom:jdom2:2.0.6.1`** (coordinate
   change only; the code already imports `org.jdom2.*`), **`commons-logging` 1.2** (still current),
   `commons-cli` 1.4 → current, and the JUnit 5 / Mockito test dependencies to current versions.

**Verification per step:** clean build, then a compile-and-run check of the two functional flows that
touch this stack — KeyManagement (`--ini`, `--hia`, `--hpb`: `xml/`, `certificate/`, `xmlsec`) and
FileTransfer (upload/download: `xml/`, `xmlbeans` schema classes, `httpclient`). Re-run the Phase 0
scan and diff the CVE list against the baseline. Any change to canonicalization or signature output
must be validated against a real bank test endpoint before release.

## Phase 4 — Harden deserialization & transport (risk: low-medium)

**Objective:** close the two code-level exposures that a version bump alone does not fix.

1. **Deserialization allow-list.** With Java 9+ available (Phase 2), install a
   `java.io.ObjectInputFilter` on every stream created by
   `DefaultSerializationManager.deserialize` (`src/main/java/org/kopi/ebics/session/DefaultSerializationManager.java:73`)
   — the single choke point for all three `readObject()` call sites in
   `EbicsClient.loadUser` (~246-257). Allow-list exactly what the persisted graph needs:
   `org.kopi.ebics.client.Bank`, `Partner`, `User`, `java.security.cert.X509Certificate` and the
   `java.lang.*`/`java.util.*`/`java.net.URL` leaves those objects hold; reject everything else, and
   cap `maxdepth`/`maxrefs`. Verify with a unit test that a serialized allowed graph round-trips and
   a stream containing a disallowed class throws `InvalidClassException`.
   *Longer term:* replace native serialization for persisted state entirely — the stored data is a
   URL, a few IDs and X.509 certificates, all of which serialize cleanly as JSON/properties plus a
   PEM/JKS keystore. That removes the attack class instead of filtering it, at the cost of a
   migration path for existing `*.cer` files.
2. **TLS floor in transport.** `HttpRequestSender.send` (~75-117) never touches TLS configuration.
   Set an explicit `SSLConnectionSocketFactory` on the `HttpClientBuilder` restricted to
   `TLSv1.2`/`TLSv1.3` with default hostname verification, so a JDK whose defaults still permit
   TLS 1.0/1.1 cannot silently negotiate down. Also consider surfacing a truststore option: EBICS
   banks are a fixed, small set, and pinning to a bank-supplied CA is realistic here.
   Verify against a TLS-1.2-only and a TLS-1.3 endpoint (e.g. `https://tls-v1-1.badssl.com:1011`
   must fail, a normal bank URL must succeed).

---

## Implemented so far in this change

* **Phase 0**: `security-scan` profile with `dependency-check-maven` 10.0.4 (report-only, JDK 8
  compatible); `.github/workflows/dependency-check.yml` (weekly + manual, artifact upload).
* **Phase 1**: `maven-compiler-plugin` 3.1 → 3.14.0 (language level deliberately unchanged at 1.7);
  `xmlbeans-maven-plugin` codegen pinned to `xmlbeans` 3.1.0, matching the runtime dependency.

Everything from Phase 2 onwards is left for maintainer review.
