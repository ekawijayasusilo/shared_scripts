# shared_script

Shared Fastlane lanes and one-time setup scripts used across my Android, Flutter and KMP template
repos. The logic lives here once; each consuming repo keeps a small stub.

Released by [release-please](https://github.com/googleapis/release-please) (`simple` strategy —
`version.txt` + `CHANGELOG.md`). Consumers pin to a **tag**, so `main` moving never breaks a build.

## Fastlane

### Consumer setup

`Gemfile`:

```ruby
source "https://rubygems.org"
gem "fastlane"
```

One Fastfile per stack, so a consumer imports only what it needs. In your
`fastlane/Fastfile`, pick the matching `path:`:

<!-- x-release-please-start-version -->
```ruby
# Native Android project
import_from_git(
  url: "https://github.com/ekawijayasusilo/shared_scripts.git",
  branch: "v1.1.1",              # a tag name is fine here — git clones it detached
  path: "fastlane/android/Fastfile"
)
```

```ruby
# Flutter project
import_from_git(
  url: "https://github.com/ekawijayasusilo/shared_scripts.git",
  branch: "v1.1.1",
  path: "fastlane/flutter/Fastfile"
)
```
<!-- x-release-please-end -->

Then `bundle install` and `bundle exec fastlane test`.

Each file is self-contained — `import_from_git` checks out only the Fastfile at `path` and a sibling
`actions/` directory, so importing one never drags the other in. A Flutter project that also needs
the native Gradle lanes can import both: the platform blocks (`:android` and `:flutter`) keep the
lane names apart. Only `default_platform` collides — the last import wins, so set it explicitly in
your own Fastfile if you import both.

**Pinning.** `branch:` takes an exact tag and gives a byte-exact pin. `version: "~> 1.1"` is the
alternative — it is a `Gem::Requirement` matched against tag names, so it picks up later patch and
minor tags automatically.

**Caching.** By default every run makes a fresh shallow clone into a temp directory; there is no
persistent cache and nothing goes stale. Passing `cache_path:` opts into a permanent full clone —
avoid it unless you need the speed, because a cached clone with a pinned `version:` range can
silently miss a newly pushed matching tag, and the cache directory is keyed only on the last segment
of the URL.

### `fastlane/android/Fastfile` — `platform :android`

| Lane | Does |
| --- | --- |
| `test` | `testDebugUnitTest` |
| `clean` | `clean` |
| `build_release` | `clean` + `assembleRelease` |
| `bundle_release` | `clean` + `bundleRelease` |

### `fastlane/flutter/Fastfile` — `platform :flutter`

Run as `fastlane <lane>`, or `fastlane flutter <lane>` if you also import the Android file. There are
no per-package lane variants — each lane discovers the
workspace packages from the root `pubspec.yaml` `workspace:` key, uses `--recursive` where the tool
supports it, and loops where it does not.

| Lane | Does | Notes |
| --- | --- | --- |
| `clean_gen` | `flutter clean` per package, one `flutter pub get` at the root, then `build_runner` in each package that declares it | Merged on purpose — always run together |
| `test` | `very_good test --recursive --coverage` | Optional `min_coverage:`. Packages with no `test/` are skipped, not failed |
| `analyze` | `dart format --set-exit-if-changed` (one call, many paths), `flutter analyze` (looped), `bloc lint .`, `dart_code_linter` (looped) | |
| `l10n` | `flutter gen-l10n` | Reads the root `l10n.yaml` |
| `build_apk` | `flutter build apk --release` | Optional `flavor:` |
| `build_aab` | `flutter build appbundle --release` | Optional `flavor:` |
| `build_ipa` | `flutter build ipa --release --no-codesign` | Optional `flavor:`. **Produces `build/ios/archive/Runner.xcarchive`, not an `.ipa`.** macOS only |
| `widgetbook` | `build_runner` then `flutter run -d chrome` in the widgetbook package | Optional `path:`. Blocking — it is a local dev tool |

Examples:

```bash
fastlane clean_gen
fastlane test min_coverage:90
fastlane build_apk flavor:dev
fastlane widgetbook
```

**Signing is out of scope.** `build_apk` / `build_aab` produce artifacts signed with the **debug
keystore** — Android has no unsigned build mode, and the Flutter Gradle template falls back to
`signingConfigs.debug`. These are not release-installable. `build_ipa` produces a genuinely unsigned
archive.

**Prerequisites on the machine running the lanes:**

```bash
dart pub global activate very_good_cli
dart pub global activate bloc_tools        # provides `bloc lint`
```

plus `dart_code_linter` as a `dev_dependency` in each workspace package.

## Setup scripts

Two scripts, split by how often they run.

| Script | Runs | Does |
| --- | --- | --- |
| `scripts/bootstrap-owner.sh` | **Once ever, per GitHub account** | Creates the automation GitHub App, captures its private key, installs it, and writes `RENOVATE_APP_ID` / `RENOVATE_APP_PRIVATE_KEY` to `shared_workflow` and `shared_scripts` |
| `scripts/setup-repo.sh` | **Once per new scaffolded project** | Verifies that App is installed, then Codecov, OpenCode, repo settings and branch protection. Never creates an App, never writes a private key |

Both need an authenticated [`gh`](https://cli.github.com) and bash 4+ (macOS ships bash 3.2 — use
`brew install bash`).

### Running them

Fetch to disk, **read it**, then run. Not `curl | bash` — these handle a GitHub App private key and
API tokens, so the script must be reviewable before it executes.

<!-- x-release-please-start-version -->
```bash
curl -fsSL -o setup-repo.sh \
  https://raw.githubusercontent.com/ekawijayasusilo/shared_scripts/v1.1.1/scripts/setup-repo.sh
less setup-repo.sh
bash setup-repo.sh
```

Tags are mutable — anyone with push access can move `v1.1.1`. Substitute a full commit SHA for the
tag in that URL when you want true immutability.
<!-- x-release-please-end -->

### `setup-repo.sh` is two-phase and resumable

Branch-protection required-check **names do not exist** until the workflows have run at least once,
so the script cannot do everything in one pass.

- **Phase 1** — confirm the target repo, verify the App, Codecov, `OPENCODE_API_KEY`, optional
  `OCR_LLM_URL` / `OCR_LLM_MODEL`, enable Issues, enable auto-merge and Actions write permissions,
  check code scanning. Then it stops and tells you to open a PR.
- **Phase 2** — after the first green run, re-run it. It discovers the actual check names and
  configures branch protection from them.

The phase is detected from live GitHub state, not a local file, so re-running is always safe: it
skips whatever is already done and reports "nothing to do" when everything is configured. Target it
with `--repo OWNER/REPO`, or let it read the current checkout.

It resolves and **confirms** the target repo before the first write — acting on the wrong repo is
costly and silent.
