# Railway templates

The CI, release, and update setup shared by my
[Railway](https://railway.com) templates. Each template has its own repo; this
repo holds the workflows they call and the Renovate preset they extend, so a
change here reaches all of them at once.

| Template | Repo | Image |
|---|---|---|
| [SearXNG](https://railway.com/deploy/searxng-api?referralCode=NhCCIt) | [railway-searxng](https://github.com/joeychilson/railway-searxng) | `ghcr.io/joeychilson/railway-searxng` |
| [OpenCode](https://railway.com/deploy/opencode-for-railway?referralCode=NhCCIt) | [railway-opencode](https://github.com/joeychilson/railway-opencode) | `ghcr.io/joeychilson/railway-opencode` |
| [celld](https://railway.com/deploy/celld-for-railway?referralCode=NhCCIt) | [railway-celld](https://github.com/joeychilson/railway-celld) | `ghcr.io/joeychilson/railway-celld` |
| [PostgreSQL with pgvectorscale](https://railway.com/deploy/postgresql-with-pgvectorscale?referralCode=NhCCIt) | [railway-pgvectorscale](https://github.com/joeychilson/railway-pgvectorscale) | `ghcr.io/joeychilson/railway-pgvectorscale` |
| [PostgreSQL with pgvectorscale and pg_textsearch](https://railway.com/deploy/postgresql-with-pgvectorscale-and-pgtext?referralCode=NhCCIt) | [railway-pgvectorscale-textsearch](https://github.com/joeychilson/railway-pgvectorscale-textsearch) | `ghcr.io/joeychilson/railway-pgvectorscale-textsearch` |

## Template repo layout

```text
.github/workflows/ci.yml        calls ci.yml here on every pull request
.github/workflows/release.yml   calls release.yml here on every push to main
Dockerfile                      every version pin has a `# renovate:` comment
test/smoke-test.sh              takes an image name; CI runs it on amd64 and arm64
README.md                       the GitHub README
TEMPLATE.md                     the overview shown on railway.com
renovate.json                   extends default.json from this repo
```

The two workflow files are identical in every repo, except that
railway-pgvectorscale-textsearch also publishes to its old image name.

## Version pins

Every upstream version an image uses is an `ARG` in the Dockerfile with a
comment on the line above it:

```dockerfile
# renovate: datasource=npm packageName=@opencode/cli depName=OpenCode
ARG OPENCODE_VERSION=2.0.28
```

The fields come in this order, and the last three are optional:

| Field | Meaning |
|---|---|
| `datasource` | Where Renovate looks for new versions: `docker`, `npm`, `github-releases`, or `github-tags` |
| `packageName` | The image, package, or `owner/repo` to look up |
| `depName` | The name used in PR titles, release titles, and release notes |
| `depType=tool` | A build tool or base image. It is listed in the release notes but left out of the release title. |
| `versioning` | A Renovate versioning scheme, when the versions aren't semver |
| `extractVersion` | A regex that pulls the version out of a tag, such as `^v(?<version>.+)$` |

An image pinned by digest keeps the digest in the ARG value
(`18.6@sha256:...`). The same comment format works above a `KEY: value` line in
a YAML file.

## Updates

[Renovate](https://docs.renovatebot.com) runs every few hours on each repo,
using [`default.json`](default.json):

- It waits until an npm or GitHub release is 2 days old. Image updates from
  Docker Hub and GHCR don't wait, because Renovate gets no push dates for
  them.
- It merges patch, minor, and digest updates once CI passes.
- It leaves major updates, and minor updates of 0.x versions, open for review.
  The repo's Dependency Dashboard issue lists them.
- It never proposes a PostgreSQL major upgrade, because existing databases
  need a dump and restore.

PR titles name the component and the new version, such as "Bump OpenCode to
2.0.29".

## Releases

Each template has its own semantic version, separate from the upstream
versions it bundles:

- Patch: upstream updates and fixes that need nothing from users.
- Minor: new behavior or settings that existing deployments can ignore.
- Major: a change users must act on, such as a new variable, a removed one, or
  a data migration.

Every push to `main` runs the release workflow. It releases when a file that
goes into the image changed since the last `vX.Y.Z` tag. Changes to top-level
Markdown files, `.github/`, `test/`, `examples/`, `renovate.json`,
`.gitignore`, and `LICENSE` don't release. The version bump is a patch unless a merged PR has the
`release:minor` or `release:major` label. To release by hand, run the Release
workflow from the Actions tab and pick a bump.

A release:

1. Builds the image on amd64 and arm64 runners and runs the smoke test on
   each.
2. Checks that the version is unused, then pushes the image to GHCR as
   `X.Y.Z` and `X.Y`.
3. Creates the `vX.Y.Z` tag and a GitHub release titled
   `vX.Y.Z — Component version, Component version`, with sections for changes,
   versions, and the image.

`X.Y.Z` never moves. `X.Y` always points to the newest patch release in that
line. No release publishes `latest`. Releases run only from `main`, at or after
the latest tag, so a re-run of an old run can't move `X.Y` backwards. Deployments still use old
tags, including `latest`, `main`, and `sha-<commit>` tags from before this
setup, so never delete or move an existing tag.

## Railway

Each template deploys the `X.Y` tag with image auto-updates set to patch, so
deployments pick up patch releases in Railway's nightly update window. After a
minor or major release, change the template's image tag to the new `X.Y` in
the template editor once the release has published, and add upgrade steps to
the release notes.

The service settings (image, variables, healthcheck, auto-updates) can only be
changed in the template editor on railway.com. The overview comes from each
repo's `TEMPLATE.md`:

```text
railway templates update <template-code> --readme-file TEMPLATE.md
```

Every template should have:

- A 1:1 icon on the template and each service. [`icons/`](icons) holds them,
  and the templates load them from
  `https://raw.githubusercontent.com/joeychilson/railway-templates/main/icons/<name>.svg`.
- A description on every variable.
- A healthcheck path when the service has one.
- A volume for any state that must survive a redeploy.

## Repo settings

Every template repo uses the same settings:

- Squash merging only, with the PR title and description as the commit
  message.
- Branches deleted after merge, and auto-merge allowed.
- A `main` ruleset that requires the `ci / lint`, `ci / test (amd64)`, and
  `ci / test (arm64)` checks.
- The [Renovate app](https://github.com/apps/renovate) installed.
- `release:minor` and `release:major` labels.

## License

[MIT](LICENSE)
