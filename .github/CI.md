# Shared CI

Reusable workflows and composite actions shared across `Florian-Noever` projects. Each workflow covers one kind of project: `dotnet-ci.yml` tests and publishes a .NET project, and the `vscode-extension-*` workflows check, test, pack and publish a VS Code extension. A repository that has both calls both and hands files from one to the other as an artifact.

Workflows only wire jobs together. Everything bigger than a single command, such as packing, uploading to a release or publishing to a registry, lives in a composite action of its own, so it can be reused without the workflow around it.

Consumers reference them by tag:

```yaml
uses: Florian-Noever/Florian-Noever/.github/workflows/vscode-extension-ci.yml@v1
uses: Florian-Noever/Florian-Noever/.github/actions/vsix-pack@v1
```

## Layout

| Action | Used by | Purpose |
| --- | --- | --- |
| `resolve-version` | vscode-extension | Validates a release tag and reports the version it carries |
| `setup-dotnet` | dotnet | Installs the SDK and restores the NuGet cache |
| `dotnet-publish` | dotnet | Publishes a .NET project once per publish profile |
| `vsix-pack` | vscode-extension | Installs, optionally checks, builds and tests an extension, packs it with vsce and checks the VSIX's version and contents |
| `vscode-test` | vscode-extension, `vsix-pack` | Runs an extension's tests under a virtual display |
| `github-release-upload` | vscode-extension | Uploads files to the release that triggered the run and attests them |
| `vs-marketplace-publish` | vscode-extension | Publishes a VSIX to the Visual Studio Marketplace through Microsoft Entra ID |
| `open-vsx-verify` | vscode-extension | Checks the token, the namespace and the dependencies before anything is released |
| `open-vsx-publish` | vscode-extension | Publishes a VSIX to Open VSX with a token |
| `dependabot-merge` | dependabot-automerge | Merges a Dependabot minor or patch update, and leaves any other update for a review |

| Reusable workflow | Purpose |
| --- | --- |
| `dotnet-ci.yml` | Optional tests of a .NET project, and its output published per publish profile and kept as an artifact |
| `vscode-extension-ci.yml` | Optional checks and VS Code tests, and a checked preview VSIX |
| `vscode-extension-publish.yml` | VSIX built and tested from the release tag, uploaded and attested, then published to the Visual Studio Marketplace and Open VSX |
| `dependabot-automerge.yml` | Squash-merges a Dependabot minor or patch update once the other jobs of the calling workflow passed |

## Consuming: a VS Code extension

`.github/workflows/ci.yml`, here for an extension that ships a .NET helper:

```yaml
name: CI

on:
  push:
    branches: ['**']
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  bridge:
    uses: Florian-Noever/Florian-Noever/.github/workflows/dotnet-ci.yml@v1
    with:
      test-project: AL-ActionImage-Viewer.ImageInformationProvider
      publish-project: AL-ActionImage-Viewer.ImageInformationProvider/AL-ActionImage-Viewer.ImageInformationProvider/AL-ActionImage-Viewer.ImageInformationProvider.csproj
      publish-profiles: |
        win32
        linux
        darwin
      artifact-name: bridge
      artifact-path: bin

  extension:
    needs: bridge
    uses: Florian-Noever/Florian-Noever/.github/workflows/vscode-extension-ci.yml@v1
    with:
      artifact-name: bridge
      artifact-path: bin
      test-command: npm test
      required-files: |
        bin/win32/AL-ActionImage-Viewer.ImageInformationProvider.exe
        bin/linux/AL-ActionImage-Viewer.ImageInformationProvider
        bin/darwin/AL-ActionImage-Viewer.ImageInformationProvider
```

`.github/workflows/publish.yml`:

```yaml
name: Publish Release

on:
  release:
    types: [published]

concurrency:
  group: vscode-extension-publish
  cancel-in-progress: false

permissions:
  contents: write
  id-token: write
  attestations: write

jobs:
  bridge:
    uses: Florian-Noever/Florian-Noever/.github/workflows/dotnet-ci.yml@v1
    with:
      test-project: AL-ActionImage-Viewer.ImageInformationProvider
      publish-project: AL-ActionImage-Viewer.ImageInformationProvider/AL-ActionImage-Viewer.ImageInformationProvider/AL-ActionImage-Viewer.ImageInformationProvider.csproj
      publish-profiles: |
        win32
        linux
        darwin
      version: ${{ github.event.release.tag_name }}
      artifact-name: bridge
      artifact-path: bin

  publish:
    needs: bridge
    uses: Florian-Noever/Florian-Noever/.github/workflows/vscode-extension-publish.yml@v1
    with:
      artifact-name: bridge
      artifact-path: bin
      test-command: npm test
      required-files: |
        bin/win32/AL-ActionImage-Viewer.ImageInformationProvider.exe
        bin/linux/AL-ActionImage-Viewer.ImageInformationProvider
        bin/darwin/AL-ActionImage-Viewer.ImageInformationProvider
      vs-marketplace: true
```

An extension without a helper leaves out the `bridge` job and the `artifact-*` inputs. Both files repeat the build inputs, since each workflow builds the extension on its own. This extension depends on one that is not on Open VSX, so it leaves `open-vsx` off; one that goes there as well adds `open-vsx: true` and passes the token:

```yaml
      open-vsx: true
    secrets:
      OPEN_VSX_TOKEN: ${{ secrets.OPEN_VSX_TOKEN }}
```

### Building and testing

The extension is an npm project at the repository root. `vsix-pack` runs `npm ci` and then `vsce package`, which runs the extension's `vscode:prepublish` script, so whatever that script builds needs no input here. The VSIX is written as `artifacts/<name>-<version>.vsix`, and before anything else the action checks that `package-lock.json` carries the same version as `package.json`.

- `artifact-name` names an artifact of an earlier job of the same run, such as a helper built by `dotnet-ci.yml`, that is downloaded into `artifact-path` before anything is built or tested.
- `check-command` runs after `npm ci`, typically linting and unit tests that `vscode:prepublish` does not already cover.
- `build-command` runs after `npm ci` for anything else `vscode:prepublish` does not build.
- `test-command` runs the extension's tests, for example with `@vscode/test-electron`, after the build. It runs under `xvfb-run`, since VS Code needs a display even when it is tested.
- `required-files` lists paths or glob patterns, relative to the extension root, that must each match a file in the VSIX, such as the helper for every platform. The run fails if one is missing, and the log always shows what the VSIX contains.
- `pack-dependencies` stays `false` for a bundled extension, which vsce then packs with `--no-dependencies`. Set it for an extension that ships its `node_modules`.

In CI, the `Check` job runs `check-command`, the `Test` job runs `test-command`, and the `Package` job packs the extension. Each job only runs when it has something to do, apart from `Package`, which keeps the VSIX as an artifact for `retention-days`, so each push leaves a build to try out.

### Shipping a .NET helper

`dotnet-ci.yml` builds the helper, independently of what uses it:

- `test-project` runs `dotnet test` on a project, solution or folder.
- `publish-project` is published with `-c Release` once per entry of `publish-profiles`. Where the output goes is up to the profiles or the project, for example a target that runs after `Publish` and copies the executable into `bin/<platform>/`.
- `version`, such as the release tag, is stamped into the published assemblies; a leading `v` is dropped.
- `artifact-name` keeps the directory `artifact-path` as an artifact for `retention-days`, one day by default, which is long enough for the later jobs of the same run.

The extension's job needs the helper's job, and the `vscode-extension-*` workflows download the artifact into the same `artifact-path`, so the extension is tested and packed with the binaries that job built; `.vscodeignore` must not exclude them. A failing .NET test therefore stops the extension's jobs as well. Artifacts do not keep the executable bit, so the extension has to set it on its helper before running it on Linux and macOS.

### Releasing

A release is built from its tag, which must be a plain version such as `v1.2.3`: VS Code extensions carry no prerelease labels in their version. `package.json` and `package-lock.json` must both carry that version, so bump them together with `npm version 1.2.3 --no-git-tag-version`. The run fails before releasing anything otherwise.

The `Pack` job runs everything the extension's CI runs, in one job: `check-command`, `build-command`, `test-command`, packing and the content check, with the helper published from the same tag. The VSIX that passed is uploaded to the release and attested, and unless the release is a pre-release, the same file then goes to every registry that is switched on:

- `vs-marketplace: true` publishes it to the **Visual Studio Marketplace**;
- `open-vsx: true` publishes it to **Open VSX**.

Both are off unless set. A pre-release only gets the VSIX on the GitHub release. Unticking *pre-release* afterwards does not publish it, since GitHub reports that as `released` rather than `published`; release a new version instead.

The registries skip a version they already have, so a failed registry job can be retried with **Re-run failed jobs**. Do not use *Re-run all jobs*: it rebuilds the VSIX, which replaces the file on the release while the registries keep the first build. Do not turn on immutable releases either, since the VSIX is attached after the release is published.

As for the other workflows, the secrets and the permissions block belong to the calling repository. A reusable workflow never sees the secrets of the repository that stores it, and it cannot grant itself more permissions than its caller has.

### Visual Studio Marketplace

Azure DevOps retires global personal access tokens on 2026-12-01, and a Marketplace token had to be one. The workflow therefore publishes through Microsoft Entra ID: the job exchanges its GitHub OIDC token for the identity of an Entra app registration through `azure/login`, and `vsce publish --azure-credential` uses that identity. No Marketplace secret is stored anywhere.

One app registration, *Marketplace publishing* in the [Microsoft Entra admin center](https://entra.microsoft.com), publishes every extension of the account. Its client and tenant ids are the defaults of `azure-client-id` and `azure-tenant-id`, so callers pass neither. It trusts GitHub through a single flexible federated credential under *Certificates & secrets > Federated credentials*: scenario *Other issuer*, issuer `https://token.actions.githubusercontent.com`, audience `api://AzureADTokenExchange` and this claims matching expression:

```text
claims['sub'] matches 'repo:Florian-Noever*:environment:vs-marketplace' and claims['repository_owner_id'] eq '242208742' and claims['job_workflow_ref'] matches 'Florian-Noever/Florian-Noever/.github/workflows/vscode-extension-publish.yml@*'
```

It accepts the Marketplace job of this workflow in any repository of the account, and nothing else. The owner is matched by its numeric id, which survives renames, and `repo:Florian-Noever*` covers both the name-based subject of repositories created before 2026-07-15 and the immutable `repo:Florian-Noever@242208742/<repository>@<id>` subject of newer ones. So a new extension needs no setup in Entra or in its repository's settings. The job runs in the environment `vs-marketplace`, which GitHub creates on the first run; limiting its deployments to tags matching `v*` is optional hardening.

Flexible federated credentials are still a preview. Should they stop working, give each repository a classic credential instead (scenario *GitHub Actions deploying Azure resources*, entity type *Environment*, environment `vs-marketplace`) and check its subject: the portal proposes the immutable format, which only matches repositories created after 2026-07-15 or opted in to it. `gh api /repos/Florian-Noever/<repository>/actions/oidc/customization/sub` shows which format a repository uses.

The app has to be a member of every publisher it publishes for. The first time it publishes for a publisher, the Marketplace refuses it and the failed job leaves a notice with the app's Azure DevOps profile id. Add that id under *Members* of the publisher at <https://marketplace.visualstudio.com/manage/publishers> with the *Contributor* role, then re-run the failed jobs.

### Open VSX

Open VSX publishes with a token. Create one under *Settings > Access Tokens* at <https://open-vsx.org> and store it as the `OPEN_VSX_TOKEN` secret. The publisher's namespace has to exist on Open VSX; create it once with `npx ovsx create-namespace <publisher> -p <token>`.

Open VSX refuses an extension whose `extensionDependencies` it cannot resolve, and many extensions depend on one that is only on the Visual Studio Marketplace. So when `open-vsx` is on, `open-vsx-verify` fails the run before anything is released unless the namespace and every dependency are on Open VSX, and also if the token is missing. Leave `open-vsx` off for such an extension.

The Open VSX job runs without the `id-token` permission. ovsx would otherwise try Open VSX's trusted publishing on its own instead of the token.

## Consuming: Dependabot auto-merge

`dependabot-automerge.yml` merges a Dependabot pull request once the repository's own checks passed. Add it as the last job of the repository's CI, after every other job:

```yaml
  automerge:
    needs: [bridge, extension]
    permissions:
      contents: write
      pull-requests: write
    uses: Florian-Noever/Florian-Noever/.github/workflows/dependabot-automerge.yml@v1
```

Nothing is merged unless every job in `needs` succeeded. The merge only happens on `pull_request` runs that Dependabot started for its own pull request; on every other run the job is skipped, and a pull request that someone else pushed commits to is left for a review. Dependabot's runs get a read-only token, and the `permissions` of the calling job are what let this one merge. The repository has to allow squash merging.

Only minor and patch updates are merged, judged by the update type `dependabot/fetch-metadata` reports; for a group it is the largest change in the group. Major updates, and updates whose type is unknown, stay open with a notice, since a green CI does not prove that nothing the tests miss broke. When several pull requests finish together, GitHub refuses a merge while it still settles the base branch another one just changed, so the merge is tried up to five times, ten seconds apart. A pull request that conflicts after all is rebased by Dependabot, which runs the checks and this job again.

Dependabot itself is set up in the repository's `.github/dependabot.yml`. For NuGet, point `directories` at the folders that directly hold the solution or project files: Dependabot does not search subfolders for them.

## Why Marketplace publishing can be a reusable workflow

nuget.org's trusted publishing requires the repository in the OIDC `job_workflow_ref` claim to match the `repository` claim, which fails once the publishing job lives in another repository. A Microsoft Entra federated credential only checks the claims it names, and in a reusable workflow the subject names the calling repository and its environment while `job_workflow_ref` names the shared workflow:

```text
sub              = repo:Florian-Noever/al-actionimage-viewer:environment:vs-marketplace
job_workflow_ref = Florian-Noever/Florian-Noever/.github/workflows/vscode-extension-publish.yml@refs/tags/v1
```

so the Marketplace job can stay in the shared workflow, and the credential above even requires it to.

vsce 4 can also publish through the Marketplace's own trusted publishing (`vsce publish --oidc`), without Entra ID. The Marketplace cannot be configured for it yet. Once it can, check which claims it matches before switching, for the same reason as nuget.org.

## Versioning

Consumers track a moving major tag. Tags are annotated:

```bash
git tag -a v1.1.0 -m "Short summary of the change"
git tag -fa v1 -m "Latest v1.x of the shared CI workflows and composite actions" v1.1.0
git push origin v1.1.0 && git push -f origin v1
```

Things to remember:

- The workflows, and actions that use other actions such as `vsix-pack`, reference them by their **absolute** `@v1` path. A relative `./` path would resolve against the calling repository, which does not contain these actions. Bump those refs with a new major.
- The vsce and ovsx versions are pinned as input defaults: `vsce-version` of `vsix-pack` and `vs-marketplace-publish`, and `ovsx-version` of `open-vsx-publish`. Dependabot does not see them, so bump them by hand and keep the two vsce pins equal.
- Whoever can move `v1` decides what runs with the publishing identity of every extension that calls `vscode-extension-publish.yml@v1`. Pin the call to a commit SHA if you would rather not have it move implicitly.
