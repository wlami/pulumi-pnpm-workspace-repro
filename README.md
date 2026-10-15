# `pulumi package add` writes a second `pnpm-workspace.yaml` inside a pnpm workspace package

Minimal reproduction for two faults in the Node.js `pnpm` package manager path when a Pulumi
project is a package inside a pnpm workspace.

* Pulumi CLI `v3.268.0`
* pnpm `11.17.0`
* Node.js `v24.12.0`
* macOS (darwin/arm64)

## Layout

```
.
├── pnpm-workspace.yaml        # packages: ["packages/*"]
├── package.json               # private workspace root
└── packages/
    └── infra/
        ├── Pulumi.yaml        # runtime: nodejs, options.packagemanager: pnpm
        ├── package.json
        └── index.ts
```

## Steps

```bash
pnpm install
cd packages/infra
pulumi package add random
```

## Expected

The SDK is generated and linked, its `postinstall` is allowed to run, and no new workspace
configuration file is created. The workspace root is the only place that holds workspace config.

## Actual

`pulumi package add` fails:

```
Successfully generated an SDK for the random package at .../packages/infra/sdks
Installing dependencies...

Detected workspace root at .../pulumi-pnpm-workspace-repro
Scope: all 2 workspace projects

[ERR_PNPM_IGNORED_BUILDS] Ignored build scripts: @pulumi/random@file:packages/infra/sdks/random, protobufjs@7.6.6

Run "pnpm approve-builds" to pick which dependencies should be allowed to run scripts.
error: installing dependencies: exit status 1
```

A second workspace file has appeared, and its key does not match the one pnpm resolves:

```yaml
# packages/infra/pnpm-workspace.yaml   <- written by Pulumi, did not exist before
allowBuilds:
  '@pulumi/random@file:sdks/random': true          # relative to the PROJECT directory
```

```yaml
# pnpm-workspace.yaml (root)           <- what pnpm actually asks about
packages:
  - "packages/*"
allowBuilds:
  '@pulumi/random@file:packages/infra/sdks/random': set this to true or false
  protobufjs: set this to true or false
```

Note that running the repro mutates the root `pnpm-workspace.yaml` too, because pnpm appends the
placeholder entries. `git checkout pnpm-workspace.yaml` to reset between runs.

### Fault 1: the project-level write ignores the workspace root

`Link` → `allowBuildScripts` → `mergeProjectConfig` in `sdk/nodejs/npm/pnpm.go` runs:

```
pnpm config set allowBuilds <json> --location project
```

with `cmd.Dir` set to the Pulumi project directory. On pnpm 11 that setting is stored in
`pnpm-workspace.yaml`, so pnpm creates one in that directory. Nothing checks whether the directory
is already part of a workspace, even though `npm.FindWorkspaceRoot` in `sdk/nodejs/npm/workspaces.go`
— same package, and what `pulumi install` uses to find the root — does exactly that walk-up.

It is recreated on **every** `pulumi package add`, so it cannot simply be deleted once. On a second
run it also copies the merged root configuration into the project file, since `pnpm config get`
reads the effective config from the root and `--location project` then writes the merged result
locally:

```yaml
# packages/infra/pnpm-workspace.yaml after a second `pulumi package add`
allowBuilds:
  '@pulumi/random@file:packages/infra/sdks/random': true
  '@pulumi/random@file:sdks/random': true
  protobufjs: false
```

pnpm does not recommend a nested workspace file
([pnpm discussion #10267](https://github.com/orgs/pnpm/discussions/10267#discussioncomment-15252093)).

### Fault 2: the depPath key is built relative to the wrong directory

`allowBuildScripts` builds the key as `packageName + "@file:" + path`, where `path` is relative to
the project directory (`sdks/random`). pnpm keys local directory dependencies relative to the
**workspace root** (`packages/infra/sdks/random`). The two never match, so the allowlist entry has
no effect and the generated SDK's `postinstall` — which compiles the SDK from TypeScript — is never
allowed to run.

## Workaround

Put the approval in the workspace root file, using the root-relative path, and delete the project
one:

```yaml
# pnpm-workspace.yaml at the workspace root
packages:
  - "packages/*"
allowBuilds:
  '@pulumi/random@file:packages/infra/sdks/random': true
  protobufjs: false
```

```bash
rm packages/infra/pnpm-workspace.yaml
pulumi install    # succeeds, postinstall runs, no second file is created
```

`pulumi install` is unaffected by both faults — it calls `FindWorkspaceRoot` and does not link — so
only `pulumi package add` needs the cleanup, and it needs it every time.

## Adjacent: `protobufjs` is not allowlisted outside `pulumi new`

The same `pulumi package add` fails on pnpm 11 even **without** a workspace, in a project that was
not created by `pulumi new`:

```
.../node_modules/@pulumi/random postinstall$ node ./scripts/postinstall.js
.../node_modules/@pulumi/random postinstall: Done

[ERR_PNPM_IGNORED_BUILDS] Ignored build scripts: protobufjs@7.6.6
error: installing dependencies: exit status 1
```

Here the SDK's own `postinstall` is approved correctly (no workspace, so the relative path matches),
and only `protobufjs` blocks the install. The `allowBuilds: protobufjs: false` entry is written
solely by `Template` in `sdk/nodejs/cmd/pulumi-language-nodejs/main.go`, i.e. by `pulumi new`, so any
pre-existing project hits this. Adding it by hand makes the install pass:

```yaml
allowBuilds:
  '@pulumi/random@file:sdks/random': true
  protobufjs: false
```

That `Template` write has the same missing root check as Fault 1.
