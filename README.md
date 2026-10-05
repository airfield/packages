# Airfield Packages

This repository contains shared dependency manifests used by Airfield package
builds.

Manifests are grouped by target device:

```text
x86_64/*.yaml
arm64/*.yaml
```

Each manifest name must match the dependency name a package uses, in its
`package.xml` or in the `dependencies:` list of its `airfield.yaml`.

Example package dependency:

```yaml
dependencies:
  - tqdm
```

Matching manifest:

```text
x86_64/tqdm.yaml
```

Most dependencies need no manifest. In a ROS package, a name with no manifest
(whether it comes from `package.xml` or from `airfield.yaml`) is translated
the way rosdep translates it: Airfield reads rosdep's lookup table, so
`nav2_msgs` becomes `ros-<distro>-nav2-msgs`, `eigen` becomes `libeigen3-dev`
and `python3-numpy` stays `python3-numpy`. A name the table does not have is
tried under its conventional apt name: `ros-<distro>-<name>` for a ROS
package name, or the name itself for a Debian-style name.

Add a manifest for what rosdep cannot say:

- the dependency comes from a source build or a downloaded `.deb`, or from
  pip and rosdep has no key for it;
- it installs differently per machine or architecture;
- rosdep does not know the name (`opencv2` standing for `libopencv-dev`), or
  the install should deliberately differ from rosdep's.

A manifest takes precedence over the table. One that only says what the
table already says (`ros-$ROS_DISTRO-<name>`, or `python3-numpy` for
`python3-numpy`) is redundant; such manifests are kept here for Airfield
versions that predate the table, and new ones are not needed.

### Names that mean an Ubuntu package must install that Ubuntu package

A manifest for a `python3-*` name, or for any library ROS's own binaries are
built against, should install the apt package and not a pip one. Every ROS
binary (cv_bridge, the message libraries) was compiled against Ubuntu's
numpy and OpenCV. A pip package of the same module lands in front of
Ubuntu's, and usually brings a newer numpy with it; cv_bridge then crashes or
misreads image types. The `pip check` Airfield runs at the end of a build
cannot see this, because an apt package's compiled modules declare no pip
requirements. If a package really wants pip's build of something, give that
manifest a name of its own (`opencv-python-headless`), so that the standard
name keeps its standard meaning.

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
