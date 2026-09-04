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
- `apt`: apt package names (shell variables like `$ROS_DISTRO` are expanded).
- `pip`: Python requirements (names, optionally with a version constraint).
- `system`: root-level install commands run during image build.
- `user`: user-level install commands run during image build.
- `host_dependencies`: optional host requirements checked before build.

## Prefer `apt:`/`pip:` over commands in `system:`/`user:`

Entries are package names, not commands:

`xplatform/nav2_util.yaml`:

```yaml
name: nav2_util
version: 1.0.0
apt:
  - ros-$ROS_DISTRO-nav2-util
```

`xplatform/tqdm.yaml`:

```yaml
name: tqdm
version: 1.0.0
pip:
  - tqdm
```

Every `pip:` entry across a package's dependencies is collected into one
`pip install`, so pip's resolver sees them together and can find versions that
satisfy all of them. Separate `pip install` runs each solve in isolation: a
later one will uninstall a version an earlier one needs, print an error about
it, and still exit 0 — a green build that fails at runtime.

Apt entries are batched the same way. Apt has no version-skew problem (one
version per package per release), but with `-y` it resolves a genuine conflict
by *removing* the other package and exiting 0 — batching makes that fail loudly
instead. The everyday benefit is one index refresh per package rather than one
per dependency.

Keep a raw `system:`/`user:` command only when the package manager cannot
express the install as a name: a custom pip index, a GPU/CPU branch, a
downloaded `.deb`, or adding a third-party apt repository. Those sit outside the
batched resolve; the `pip check` Airfield runs at the end of every build is what
catches collisions across that seam.

Project-local or package-local manifests are useful during development, but
shared dependencies should be upstreamed with:

```bash
airfield package dependencies check .
airfield package dependencies upstream .
```

Both commands accept `--target-device` and an optional path argument. The path may
point at a package or project directory and defaults to the current directory.
