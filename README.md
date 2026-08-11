# IoTMining Build (BOM + Parent)

Central **BOM** and **Parent POM** for all IoTMining services, published to
**GitHub Packages (Maven)** from this repo (`iiotedge/platform-build`).

`com.iiotedge.build:iiotedge-parent`/`:iiotedge-bom` are live - the
tag-triggered release path has been run for real and confirmed publishing
successfully under the `iiotedge` org. License: Apache 2.0 (see `LICENSE`).

## Why this exists

Without this, a service's `pom.xml` resolves its parent via a **local relative path**
(`../pom.xml`), which only works if you've checked out the entire monorepo with the
exact right directory layout - a developer can't `git clone` just one service and
build it. Publishing the shared parent/BOM as versioned artifacts to a registry fixes
that: any service resolves them by GAV coordinates over the network, the same way it
already resolves `spring-boot-starter-web` from Maven Central. No sibling checkouts
required.

**Parent and BOM are deliberately two separate artifacts, not one.** A `pom.xml` can
only have a single `<parent>`, but can `import` any number of BOMs via
`<dependencyManagement>`. Splitting them means a service that *can't* use
`iiotedge-parent` as its actual parent (e.g. one already parented directly off
`spring-boot-starter-parent`, or a very different stack) can still `import` the BOM
alone for consistent dependency versions across the platform.

- **`iiotedge-parent`** - build governance: plugin versions and their default
  config (compiler, surefire/failsafe, JaCoCo coverage gate, SpotBugs + FindSecBugs,
  OWASP dependency-check, CycloneDX SBOM, Spotless/Google-Java-Format, Paketo
  container images), plus `ci`/`it`/`sbom`/`security` profiles. Itself parented off
  `spring-boot-starter-parent`, so a service using it gets Boot's own sensible
  defaults too, in one hop.
- **`iiotedge-bom`** - dependency *version* alignment only. Imports the Spring
  Boot/Cloud/Jackson/Micrometer/Testcontainers BOMs and pins ~25 individual libraries
  (Postgres, Kafka, Flyway, JJWT, ShedLock, BouncyCastle, the test stack, etc) so
  every service that imports it gets the same, deliberately-chosen version of each,
  without needing to declare `<version>` itself.

## Modules
- `iiotedge-bom/` → `com.iiotedge.build:iiotedge-bom:2025.08.18`
- `iiotedge-parent/` → `com.iiotedge.build:iiotedge-parent:2025.08.18`

## How to Release

1. Push a tag matching the workflow's trigger: `git tag 2025.08.18 && git push --tags`
   (a `v`-prefixed form like `v2025.08.18` also works - the workflow strips the `v`
   before using it as the actual Maven version, so both forms publish identically).
2. `release-parent.yml` fires: versions and installs the BOM locally first (the
   parent's own version can't be set until the BOM it imports is resolvable under its
   new version - ordering that mattered enough to break this workflow twice before it
   was right), then aligns the parent's version and BOM import, then deploys BOM then
   Parent to GitHub Packages.

**This path has been run for real and confirmed working** - the packages currently
live on GitHub Packages came from an actual tagged release, not just local simulation.

## Snapshots

Push to `main` → `snapshot-parent.yml` forces a real `-SNAPSHOT`-suffixed version
(the same way the release path forces a tag-matched version - the checked-in
`pom.xml` version is a plain release number, not `-SNAPSHOT`, so this step is
what makes repeated pushes actually behave like Maven snapshots instead of trying
to redeploy the same fixed version) before installing and deploying, in the same
BOM-before-parent order as the release path.

**Unlike the release path above, this one hasn't been exercised against real
GitHub Packages yet** - only verified via local simulation with a clean `.m2`
cache. Given the release path broke twice in ways only a real (or truly
clean-cache) run surfaced, treat this as unconfirmed until it's actually been run.

## Consume in a Service

```xml
<parent>
  <groupId>com.iiotedge.build</groupId>
  <artifactId>iiotedge-parent</artifactId>
  <version>2025.08.18</version>
  <relativePath/>
</parent>

<!-- Required even though the parent's own <repositories> block already
     points here - confirmed with a genuinely empty local .m2 cache that
     Maven only checks Central otherwise and never attempts GitHub Packages
     at all. The not-yet-resolved parent can't bootstrap its own
     resolution; a child needs its own entry to find the parent in the
     first place. Only the parent's *dependencies* (like the BOM it
     imports) benefit from its repository list, once the parent itself has
     already been fetched. -->
<repositories>
  <repository>
    <id>github</id>
    <url>https://maven.pkg.github.com/iiotedge/platform-build</url>
  </repository>
</repositories>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.iiotedge.build</groupId>
      <artifactId>iiotedge-bom</artifactId>
      <version>2025.08.18</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

The explicit empty `<relativePath/>` matters too - without it, Maven's default
behavior is to check `../pom.xml` *before* falling back to the repository, which
silently resolves to whatever happens to sit one directory up in a monorepo
checkout instead of the intended published parent.

If a consuming service's own `spring-boot-maven-plugin` execution already
existed before adopting this parent, give it an explicit `<id>repackage</id>`.
Once the parent chain goes through `spring-boot-starter-parent` for real (this
parent's own parent), Boot's pluginManagement binds a default execution under
that same id - without matching it, the goal runs twice.

**Two real costs of adopting this parent, worth knowing before you do:**
- `spotless-maven-plugin` is configured with `apply`+`check` bound, enforcing Google
  Java Format - existing code not already in that style will fail the build (or get
  reformatted) the first time it picks this up.
- The enforcer's `dependencyConvergence` rule is strict - any two dependencies
  pulling in different transitive versions of the same artifact fails the build. Real
  services with an existing, organically-grown dependency tree may hit this on first
  adoption and need to add explicit version overrides to converge.

Neither is a reason not to adopt it - just don't be surprised.

### Dev machine setup (for local `mvn install`/`mvn deploy` against this repo)

`~/.m2/settings.xml`:
```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">
  <servers>
    <server>
      <id>github</id>
      <username>${env.GH_ACTOR}</username>
      <password>${env.GH_TOKEN}</password>
    </server>
  </servers>
</settings>
```

```bash
export GH_ACTOR=<your-gh-username>
export GH_TOKEN=<PAT with read:packages, write:packages if publishing>
```

## GitHub Repo Setting Required

Settings → Actions → General → Workflow permissions → **Read and write permissions**
(needed for the default `GITHUB_TOKEN` to publish packages; `GH_PACKAGES_TOKEN` is a
PAT fallback if org policy blocks that).

## Quality gates `iiotedge-parent` centralizes

A service that activates these plugins (adding a bare `<plugin>` entry with no
version/config of its own, inheriting everything from this parent) gets:

| Gate | Threshold |
|---|---|
| JaCoCo instruction coverage | >=85% |
| JaCoCo branch coverage | >=65% |
| SpotBugs + FindSecBugs | Medium+ severity, Max effort |
| Enforcer | Java 21+, Maven 3.9+, no duplicate POM dependency versions, strict dependency convergence |

Matches the thresholds auth-service and tenant-management-service already
enforce locally in their own POMs - adopting this parent is meant to be a
lateral move for them, not a downgrade. Each consuming service still supplies
its own `spotbugs-exclude.xml` (findings are necessarily service-specific);
only the plugin version and base config are centralized here.

`ci`/`it`/`sbom`/`security` profiles are also available (OWASP dependency-check,
CycloneDX SBOM, Spotless/Google-Java-Format) - see `TODO.md` for which of these
are configured but not yet actually scheduled anywhere.
