# Production Readiness Checklist

What's actually verified vs. still open. See `TODO.md` for the reasoning
behind each unchecked item. Mirrors auth-service/tenant-management-service's
checklist structure.

## Package identity & publishing

- [x] groupId/artifactId consistently `com.iiotedge.build:iiotedge-parent`
      / `:iiotedge-bom` everywhere (POMs, workflows, README, directory names)
- [x] Repo/org references consistent (`iiotedge/platform-build`) across
      `<url>`/`<scm>`/`<distributionManagement>` and the local git remote
- [x] Version drift between the parent's BOM import and the BOM's own
      version resolved
- [x] Release workflow's tag trigger actually matches what its own README
      tells people to run
- [x] **A real tag-triggered release has actually succeeded** -
      `com.iiotedge.build:iiotedge-parent`/`:iiotedge-bom` are live on
      GitHub Packages
- [ ] Snapshot (branch-push) publish path exercised for real, not just
      simulated locally
- [ ] `bump-services.yml`'s downstream service list verified against real
      repo names

## Quality gates

- [x] JaCoCo coverage gate (>=85% instruction / >=65% branch) in the
      parent's `pluginManagement`, matching auth-service/TMS's own
      already-proven thresholds
- [x] SpotBugs + FindSecBugs (Medium threshold, Max effort) likewise
      centralized in the parent
- [x] Every consuming service still supplies its own `spotbugs-exclude.xml`
      (findings are necessarily service-specific) - only the plugin
      version/base config is centralized
- [ ] `sonar-maven-plugin` version pinned (currently resolves whatever's
      latest at build time)
- [ ] OWASP dependency-check / CycloneDX SBOM profiles actually scheduled
      somewhere, not just configured

## Correctness (found via real CI failures, not just review)

- [x] `xmllint`/`xmlstarlet` namespace bug (silently no-op'd against
      namespaced Maven POMs) - fixed and verified with a completely clean
      local `.m2` cache, not one that happened to already have the answer
      cached
- [x] Version-alignment ordering bug (parent's own `versions:set` needs the
      BOM already installed under its new version) - fixed the same way
- [x] `reusable-container.yml`'s broken Tag & Push step (referenced a Docker
      tag Paketo never actually built) - removed, replaced with an explicit
      build-time image name override
- [x] `reusable-security.yml` missing `actions/setup-java` entirely - fixed
- [x] BOM's `dependencyConvergence` gap on jjwt (only `jjwt-api` pinned,
      `jjwt-impl`/`jjwt-jackson` not, at a version mismatched with what
      auth-service already used) - found on the first real adoption
      attempt, fixed by pinning all three together
- [x] README's claim that a consuming service doesn't need its own
      `<repositories>` entry - empirically wrong, disproven with a
      genuinely empty local `.m2`, corrected

## Documentation & repo hygiene

- [x] README explains why this exists, how to consume it, and the two real
      adoption costs (Spotless formatting, strict dependency convergence)
      up front rather than as a surprise
- [x] `LICENSE` added (Apache 2.0, matching what all 3 POMs already
      declared in their `<licenses>` metadata - the file was just missing)
- [x] `.gitignore` added (target/, IDE files, OS files, local Maven
      settings)
- [x] `TODO.md`/`CHECKLIST.md` added, matching the platform's established
      documentation convention

## Adoption (the actual point of this repo)

- [x] auth-service migrated from its local relative-path parent to
      `iiotedge-parent`/`iiotedge-bom` - `mvn clean verify` passes
      standalone (185/185 tests, SpotBugs clean, JaCoCo gate met)
- [ ] That migration confirmed against **real** GitHub Packages credentials,
      not just a local `.m2` proxy + a confirmatory 401 (right URL, missing
      auth) - needs an actual dev machine or CI run with `GH_TOKEN` set
- [ ] The rest of the microservices migrated - waiting on the above
