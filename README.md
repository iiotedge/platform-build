# IoTMining Build (BOM + Parent)

Central **BOM** and **Parent POM** for all IoTMining services, published to
**GitHub Packages (Maven)** from this repo (`iiotedge/platform-build`).

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
2. `release-parent.yml` fires: aligns both POMs' own `<version>` and the parent's BOM
   import to the tag, installs the BOM locally first (it's a dependency of the parent),
   then deploys BOM then Parent to GitHub Packages, in that order.

## Snapshots

Push to `main` → `snapshot-parent.yml` deploys the version currently checked into each
`pom.xml` as-is. Note this only behaves like a true Maven snapshot if that version
actually ends in `-SNAPSHOT`; as checked in today it's a plain `2025.08.18`, so repeated
pushes to `main` will attempt to redeploy under that same fixed version rather than a
timestamped snapshot. Worth fixing before relying on this path day-to-day.

## Consume in a Service

```xml
<parent>
  <groupId>com.iiotedge.build</groupId>
  <artifactId>iiotedge-parent</artifactId>
  <version>2025.08.18</version>
</parent>

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

A service using `iiotedge-parent` as its actual `<parent>` (no `<relativePath/>`
needed - the parent's own `<repositories>` block already points at this repo) doesn't
need a separate `<repositories>` entry of its own to resolve it. If a service instead
only imports the BOM without using this parent, it still needs that repository
declared itself.

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
