# Renovate Config

Shareable [Renovate config presets](https://docs.renovatebot.com/config-presets/) that Bitwarden repositories extend to inherit their dependency-update policy. The JSON files are the product. There is no application code, no build, and no test suite.

## Merging to `main` is the deploy

Renovate fetches these presets from the **default branch at run time**. There is no version, no publish step, and no staging. A merged change reaches every consuming repository on its next Renovate run, and repos triggered off-schedule pick it up immediately.

Treat every edit as an org-wide config change, and say in the PR body which consumers change and how their next run differs. A reviewer cannot get that from the diff, because four lines of JSON can alter every repo in the org.

## Preset layering

```
default.json          :pinAllExceptPeerDependencies  →  pinned consumers only
    └── extends non-pinned.json                      →  ALL consumers
            └── extends pin-actions.json             →  ALL consumers
```

| File               | Consumers reference it as                      | Reaches                        |
| ------------------ | ---------------------------------------------- | ------------------------------ |
| `default.json`     | `github>bitwarden/renovate-config`             | Pinned consumers only          |
| `non-pinned.json`  | `github>bitwarden/renovate-config:non-pinned`  | Everyone (carries most policy) |
| `pin-actions.json` | `github>bitwarden/renovate-config:pin-actions` | Everyone, via `non-pinned`     |

Repos that must not pin extend `non-pinned` directly. Find consumers by searching the org for `bitwarden/renovate-config`.

## Which file to edit

| The change should apply to…                   | Edit               |
| --------------------------------------------- | ------------------ |
| Everyone, pinned and non-pinned               | `non-pinned.json`  |
| Only consumers of the default (pinned) preset | `default.json`     |
| A BRE-owned datasource or manager carve-out   | `pin-actions.json` |

Pick the narrowest layer that satisfies the request, but note that a narrow file is not the same as a narrow reach. `pin-actions.json` is the smallest file here and still reaches everyone through `non-pinned`.

`pin-actions.json` and `external_dependencies/akuity-versions.json` are owned by `@bitwarden/dept-bre`; everything else defaults to `@bitwarden/dept-architecture`. Touching those two pulls in a different reviewer.

## Renovate version

The org runs the Mend-hosted Renovate GitHub App (PRs come from `app/renovate`), which tracks the latest Renovate release. Write current option spellings and check any key against the [configuration reference](https://docs.renovatebot.com/configuration-options/) before using it.

Renovate auto-migrates renamed options while resolving config, so a deprecated spelling keeps working and validates against `$schema`. Nothing surfaces it in review, and consumers only see it as a "Config Migration Needed" entry on their dependency dashboard. That grace period ends when a later Renovate major drops the migration shim, so write the current spelling and do not copy a key from a neighbouring rule without checking it.

## Conventions

- `$schema` goes first, then `extends`. Every preset carries `"$schema": "https://docs.renovatebot.com/renovate-schema.json"`.
- Order changes behavior. Later `extends` entries win on conflicting keys, and `packageRules` concatenate in array order across the whole resolved chain. Appending is not a neutral edit, so insert at the right index when position matters and say so in the PR.
- `description` is rendered by Renovate in PR bodies and on the dependency dashboard, so it is documentation for the engineer reading the PR rather than a comment for the next editor. State the constraint that forces the rule ("they don't support bracket syntax") instead of restating the keys.
- Preset filenames are the public API: lowercase, hyphenated. Renaming or deleting one breaks consumers immediately, with no deprecation window.
- Reference sibling presets by their full path (`github>bitwarden/renovate-config:<name>`), even from inside this repo. Relative paths do not resolve for consumers.

## Verifying a change

There is no test suite and nothing here executes. `$schema` catches typos in key names but not a deprecated spelling and not a selector that matches nothing. Renovate ships a validator that catches both, so run it on every preset you touch:

```sh
npx --package renovate -- renovate-config-validator --strict default.json non-pinned.json pin-actions.json
```

`--strict` exits non-zero on deprecated options and prints the migrated config, which is the fastest way to see what Renovate actually resolves. Then work the merged config by hand:

1. Which layer did I edit, and who inherits it?
2. Where does my rule land in `packageRules` once every `extends` resolves?
3. Does a later rule override it on a shared key?
4. What does a consuming repo's next run do differently?

For a risky change, point one consuming repo at your branch (`github>bitwarden/renovate-config#<branch>`), inspect the resulting dashboard and PRs, then revert the pointer before merge.

This repo's own `.github/renovate.json` dogfoods `default.json`, but with `enabledManagers: ["github-actions"]`, so it exercises one path only.

## Security-weighted keys

These presets are public and are fetched unauthenticated by every consuming repo, so a preset is a distribution channel to the whole org. Never commit a token, credential, or non-public hostname.

| Key                             | Why it carries weight                                                     |
| ------------------------------- | ------------------------------------------------------------------------- |
| `minimumReleaseAge: "7 days"`   | Org-wide soak window limiting exposure to compromised or yanked releases  |
| `vulnerabilityAlerts`           | Labels security updates and adds `team:team-appsec` as reviewer           |
| `:pinAllExceptPeerDependencies` | Pins exact versions so a resolved dependency cannot change without a diff |
| `allowedVersions`               | Caps a package, and blocks its security updates                           |
| `enabled: false`                | Stops all updates from a datasource, security included                    |
| `ignoreDeps`                    | Excludes packages from updates org-wide (`node`, `npm`)                   |

Do not weaken `minimumReleaseAge` or `vulnerabilityAlerts` without AppSec sign-off. Rules appended after `vulnerabilityAlerts` can override its keys, so check ordering.

Every cap and every `enabled: false` needs a `description` naming what has to be true before it can be lifted, because both block security updates silently and indefinitely. The `chart-releaser-action` cap sat for years with no recorded rationale, which is the outcome to avoid.

Only add a `customDatasources` registry URL for a trusted, org-approved host. Renovate queries it on every run from every consuming repo, and the response drives what version gets proposed.

## `external_dependencies/akuity-versions.json`

Not a preset, and unrelated to everything above. It mirrors an upstream Akuity asset carrying CVE identifiers consumed by other tooling, is refreshed by an automated job, and is owned by `@bitwarden/dept-bre`.

Do not hand-edit it and do not reformat it. It is a single 200KB line, so any formatter rewrites the whole file. Keep it out of preset PRs.

## Gotchas

- A rule has no effect: check whether a later rule overrides it on a shared key, then whether the layer even reaches that consumer, then run the validator to see the config Renovate actually resolved.
- Updates stopped for a package: grep all three presets for the package and its datasource. Usual causes are `allowedVersions`, `enabled: false`, `ignoreDeps`, or `dependencyDashboardApproval` parking it on the dashboard rather than opening a PR. Check the dashboard before concluding anything is broken.
- Read the merged config, not the diff. A consuming repo's Renovate log prints the fully resolved configuration, which is the ground truth.
- `git log -S <key> -- <file>` is the only record of why a cap or disable was added, since JSON cannot carry that history inline.

## References

- [Renovate configuration options](https://docs.renovatebot.com/configuration-options/), the reference to check every key against
- [Config validation](https://docs.renovatebot.com/config-validation/), the `renovate-config-validator` used above
- [Renovate config presets](https://docs.renovatebot.com/config-presets/), how `extends` resolution works
- [String pattern matching](https://docs.renovatebot.com/string-pattern-matching/), regex vs. glob rules for `managerFilePatterns` and `matchFileNames`
- [`bitwarden/ai-plugins`](https://github.com/bitwarden/ai-plugins), where generic cross-repo Claude tooling belongs instead of here
