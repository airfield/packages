# Airfield Packages

This repository contains shared dependency manifests used by Airfield package
builds.

Manifests are grouped by target device:

```text
x86_64/*.yaml
arm64/*.yaml
```

Each manifest name must match the dependency name used in a package
`airfield.yaml`.

Example package dependency:

```yaml
dependencies:
  - tqdm
```

Matching manifest:

```text
x86_64/tqdm.yaml
```

Manifest fields:

- `name`: dependency identifier.
- `version`: manifest version.
- `ros_versions`: optional compatible ROS distributions.
- `system`: root-level install commands run during image build.
- `user`: user-level install commands run during image build.
- `host_dependencies`: optional host requirements checked before build.

Project-local or package-local manifests are useful during development, but
shared dependencies should be upstreamed with:

```bash
airfield package dependencies check .
airfield package dependencies upstream .
```

Both commands accept `--target-device` and an optional path argument. The path may
point at a package or project directory and defaults to the current directory.
