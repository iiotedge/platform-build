# TODO

Known gaps, prioritized. See `CHECKLIST.md` for the full done/open list.

## Resolved (kept here briefly for context, not action items)

- ~~Org mismatch~~ - every `<url>`/`<scm>`/`<distributionManagement>`, the
  local git remote, and the actual Maven groupId/artifactId (`com.iotmining.build`
  -> `com.iiotedge.build`, `iotmining-parent`/`iotmining-bom` ->
  `iiotedge-parent`/`iiotedge-bom`) all now consistently say `iiotedge`.
- ~~Version drift~~ / ~~invalid wildcard GitHub Packages URL~~ / ~~tag-pattern
  mismatch between the workflow and its own README~~ - all fixed.
- ~~xmllint/xmlstarlet silently no-op'd against namespaced Maven POMs~~ - real
  bug, shipped once, caught via an actual failed CI run and fixed with
  `local-name()`/`-N` namespace registration.
- ~~`versions:set` on the parent failed because the BOM wasn't installed under
  its new version yet~~ - real ordering bug, also caught via a real CI
  failure; both workflows now version+install the BOM before touching the
  parent's version at all.
- ~~SpotBugs/JaCoCo coverage gate missing from the parent~~ - added, matching
  auth-service/tenant-management-service's already-adopted thresholds.
- ~~ShedLock not pinned in the BOM~~ - added.
- ~~No LICENSE file despite all 3 POMs declaring Apache 2.0~~ - added.
- ~~jjwt-impl/jjwt-jackson not pinned in the BOM~~ - only jjwt-api was
  pinned (at a stale 0.12.5, while auth-service already used 0.12.6),
  so impl/jackson's own transitive jjwt-api requirement got silently
  overridden - a real dependencyConvergence failure on auth-service's
  first adoption attempt. All three now pinned together at 0.12.6.
- ~~README claimed a consuming service doesn't need its own `<repositories>`
  entry~~ - wrong, disproven with a genuinely empty local `.m2`: the
  not-yet-resolved parent can't bootstrap its own resolution. Corrected,
  and auth-service's own pom now carries the entry it actually needs.

**First real publish confirmed working**: `com.iiotedge.build:iiotedge-parent`
and `:iiotedge-bom` are live on GitHub Packages under the `iiotedge` org.

**First real adoption piloted (auth-service)**: parent + BOM swapped in,
`mvn clean verify` passes standalone (185/185 tests, SpotBugs clean, JaCoCo
gate met). Not yet proven against real GitHub Packages credentials in this
environment - verified via local `.m2` (installed from this repo's own
source) plus a deliberate empty-cache resolution attempt that failed with
exactly `401 Unauthorized` against the correct URL, not a 404 or wrong
coordinate - confirming the wiring is right and only real credentials are
missing here. Still needs a real credentialed run (a dev machine or CI with
`GH_TOKEN` configured) before calling this fully proven.

## Publishing pipeline

- [ ] **`snapshot-parent.yml` has never actually been run** - only the
      tag-triggered `release-parent.yml` path has been confirmed working
      against real GitHub Packages. The branch-push path shares the same
      code patterns that already broke release-parent.yml twice, but hasn't
      been exercised for real yet.
- [ ] **`bump-services.yml`'s hardcoded service list** (`auth`,
      `api-gateway`, `tms`, `pricing-catalog`, `notification`, `alert`,
      `rule-engine`) doesn't match any confirmed real repo name surveyed in
      this org. Fix `core.getInput()` was applied, but this workflow is
      still not trustworthy to actually run until the list is real.
- [x] ~~The 4 `reusable-*.yml` workflows were not called from anywhere~~ -
      `reusable-build.yml` and `reusable-container.yml` were replaced by
      `maven-service.yml`, which the service repositories call.
      `reusable-security.yml` (OWASP + SBOM) and `reusable-sonar.yml` are
      still not called or scheduled anywhere.
- [ ] `reusable-sonar.yml` invokes `sonar:sonar` with no pinned plugin
      version - reproducible builds want a `sonar-maven-plugin` version in
      `iiotedge-parent`'s `pluginManagement`, not "whatever's latest today".
- [ ] Old `com.iotmining.build:iotmining-parent`/`:iotmining-bom` packages
      (published once, before the rename) are still visible/orphaned in
      GitHub Packages under the `iiotedge` org. Renaming doesn't delete
      prior versions - manual cleanup via the GitHub UI/API if wanted.

## Adoption

- [ ] **auth-service adopted `iiotedge-parent`/`iiotedge-bom`, not yet proven
      against real GitHub Packages credentials.** Locally verified only
      (see above). Needs a real run - a dev machine with `~/.m2/settings.xml`
      configured, or CI - before calling this fully done and rolling it out
      to the rest of the platform.
- [ ] tenant-management-service and every other microservice still resolve
      their parent via a local relative path - not touched yet, waiting on
      the auth-service pilot being fully confirmed first.
- [x] One of the two adoption costs documented in `README.md` turned out to
      be real on first contact: `dependencyConvergence` caught the
      jjwt-impl/jjwt-jackson gap above immediately. Spotless/Google-Java-Format
      enforcement is still unexercised - auth-service's existing code style
      hasn't been checked against it (the plugin isn't currently activated
      in auth-service's own POM, only available via the parent's
      `pluginManagement` if a service opts in).

## Security & compliance

- [ ] OWASP dependency-check (`security` profile, `failBuildOnCVSS=7`) and
      CycloneDX SBOM generation (`sbom` profile) are both configured but not
      scheduled or invoked anywhere in CI - same gap auth-service's own
      `TODO.md` notes for itself (needs an NVD API key, fine for CI, not for
      every local build).
