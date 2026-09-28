# Shared CI

Reusable workflows and composite actions shared across `Florian-Noever` projects.

Consumers reference them by tag:

```yaml
uses: Florian-Noever/Florian-Noever/.github/workflows/vscode-extension-ci.yml@v1
uses: Florian-Noever/Florian-Noever/.github/actions/vsix-pack@v1
```

## Layout

| Action | Used by | Purpose |
| --- | --- | --- |
| `resolve-version` | vscode-extension | Validates a release tag and reports the version it carries |
| `setup-dotnet` | vscode-extension | Installs the SDK and restores the NuGet cache |
| `dotnet-publish` | vscode-extension | Publishes a .NET project once per publish profile |
| `vsix-pack` | vscode-extension | Installs, optionally checks, builds and tests an extension, packs it with vsce and checks the VSIX's version and contents |

| Reusable workflow | Purpose |
| --- | --- |
| `vscode-extension-ci.yml` | Optional checks and .NET tests, optional VS Code tests, and a checked preview VSIX |
| `vscode-extension-publish.yml` | VSIX built and tested from the release tag, uploaded and attested, then published to the Visual Studio Marketplace and Open VSX |

## Consuming: a VS Code extension

`.github/workflows/ci.yml`:

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
  ci:
    uses: Florian-Noever/Florian-Noever/.github/workflows/vscode-extension-ci.yml@v1
    with:
      dotnet-project: AL-ActionImage-Viewer.ImageInformationProvider/AL-ActionImage-Viewer.ImageInformationProvider/AL-ActionImage-Viewer.ImageInformationProvider.csproj
      dotnet-publish-profiles: |
        win32
        linux
        darwin
      dotnet-test-project: AL-ActionImage-Viewer.ImageInformationProvider
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
  publish:
    uses: Florian-Noever/Florian-Noever/.github/workflows/vscode-extension-publish.yml@v1
    with:
      dotnet-project: AL-ActionImage-Viewer.ImageInformationProvider/AL-ActionImage-Viewer.ImageInformationProvider/AL-ActionImage-Viewer.ImageInformationProvider.csproj
      dotnet-publish-profiles: |
        win32
        linux
        darwin
      dotnet-test-project: AL-ActionImage-Viewer.ImageInformationProvider
      test-command: npm test
      required-files: |
        bin/win32/AL-ActionImage-Viewer.ImageInformationProvider.exe
        bin/linux/AL-ActionImage-Viewer.ImageInformationProvider
        bin/darwin/AL-ActionImage-Viewer.ImageInformationProvider
      vs-marketplace: true
```

Both files repeat the build inputs, since each workflow builds the extension on its own. This extension depends on one that is not on Open VSX, so it leaves `open-vsx` off; one that goes there as well adds `open-vsx: true` and passes the token:

```yaml
      open-vsx: true
    secrets:
      OPEN_VSX_TOKEN: ${{ secrets.OPEN_VSX_TOKEN }}
```

### Building and testing

The extension is an npm project at the repository root. `vsix-pack` runs `npm ci` and then `vsce package`, which runs the extension's `vscode:prepublish` script, so whatever that script builds needs no input here. The VSIX is written as `artifacts/<name>-<version>.vsix`, and before anything else the action checks that `package-lock.json` carries the same version as `package.json`.

- `dotnet-project` is a .NET project the extension ships, such as a helper it runs. It is published with `-c Release` once per entry of `dotnet-publish-profiles`, before the extension is built. Where the output goes is up to the profiles or the project, for example a target that runs after `Publish` and copies the executable into `bin/<platform>/`, and `.vscodeignore` must not exclude it.
- `dotnet-test-project` runs `dotnet test` on a project, solution or folder.
- `check-command` runs after `npm ci`, typically linting and unit tests that `vscode:prepublish` does not already cover.
- `build-command` runs after `npm ci` for anything else `vscode:prepublish` does not build.
- `test-command` runs the extension's tests, for example with `@vscode/test-electron`, after the build. It runs under `xvfb-run`, since VS Code needs a display even when it is tested.
- `required-files` lists paths or glob patterns, relative to the extension root, that must each match a file in the VSIX, such as the helper for every platform. The run fails if one is missing, and the log always shows what the VSIX contains.
- `pack-dependencies` stays `false` for a bundled extension, which vsce then packs with `--no-dependencies`. Set it for an extension that ships its `node_modules`.

In CI, the `Check` job runs the .NET tests and `check-command`, the `Test` job publishes the helper and runs `test-command`, and the `Package` job packs the extension. Each job only runs when it has something to do, apart from `Package`, which keeps the VSIX as an artifact for `retention-days`, so each push leaves a build to try out.

### Releasing

A release is built from its tag, which must be a plain version such as `v1.2.3`: VS Code extensions carry no prerelease labels in their version. `package.json` and `package-lock.json` must both carry that version, so bump them together with `npm version 1.2.3 --no-git-tag-version`. The run fails before releasing anything otherwise.

The `Pack` job runs everything CI runs, in one job: the .NET tests, the helper published with the release version stamped into it, `check-command`, `build-command`, `test-command`, packing and the content check. The VSIX that passed is uploaded to the release and attested, and unless the release is a pre-release, the same file then goes to every registry that is switched on:

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

The run fails before releasing anything if `vs-marketplace` is on but a caller empties either id.

### Open VSX

Open VSX publishes with a token. Create one under *Settings > Access Tokens* at <https://open-vsx.org> and store it as the `OPEN_VSX_TOKEN` secret. The publisher's namespace has to exist on Open VSX; create it once with `npx ovsx create-namespace <publisher> -p <token>`.

Open VSX refuses an extension whose `extensionDependencies` it cannot resolve, and many extensions depend on one that is only on the Visual Studio Marketplace. So when `open-vsx` is on, the run fails before releasing anything unless the namespace and every dependency are on Open VSX, and also if the token is missing. Leave `open-vsx` off for such an extension.

The Open VSX job runs without the `id-token` permission. ovsx would otherwise try Open VSX's trusted publishing on its own instead of the token.

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

- The workflows reference the actions by their **absolute** `@v1` path. A relative `./` path would resolve against the calling repository, which does not contain these actions. Bump those refs with a new major.
- The vsce and ovsx versions are pinned: `VSCE_VERSION` and `OVSX_VERSION` in `vscode-extension-publish.yml`, and the `vsce-version` default of `vsix-pack`. Dependabot does not see them, so bump them by hand and keep the two vsce pins equal.
- Whoever can move `v1` decides what runs with the publishing identity of every extension that calls `vscode-extension-publish.yml@v1`. Pin the call to a commit SHA if you would rather not have it move implicitly.
