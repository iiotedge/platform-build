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

- [ ] At least one real microservice migrated from its current local
      relative-path parent to `iiotedge-parent`/`iiotedge-bom`, resolved
      through real network access to GitHub Packages (not just the local
      `.m2` simulation used to verify this repo's own correctness)
