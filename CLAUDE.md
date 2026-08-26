# CLAUDE.md

## What this repo is

This is Aaron's fork of [python-ring-doorbell](https://github.com/python-ring-doorbell/python-ring-doorbell), the library behind the official Ring integration in Home Assistant (`homeassistant/components/ring/`). Upstream is `git remote upstream`; `origin` is `aaronwolen/python-ring-doorbell`.

The fork exists to fix problems Aaron hits in his own Home Assistant setup and to push those fixes upstream as PRs.

## Why the fork is a permanent fixture (for now)

The upstream maintainer (`@sdb9696`, sole codeowner) has been inactive since 2026-02-28, so PRs sit unmerged. Until that changes, Aaron builds a custom Home Assistant Docker image that installs this fork in place of the pinned PyPI `ring-doorbell` release. Home Assistant core itself pins an older version than upstream master, so the fork is two hops ahead of what stock HA ships.

Every fix therefore has two jobs: land upstream as a clean PR, and ship in production today. Those goals pull in different directions, which is what the branch convention below exists to manage.

## Branch convention

Four roles, and nothing does double duty.

- `upstream/master`: the base for everything. Never commit to it.
- `master`: `upstream/master` plus a single `docs: fork notes` commit carrying this file, and nothing else. Sync with `git rebase upstream/master`, never merge.
- `fix/<slug>`: one branch per upstream PR, cut from `upstream/master` (not from `master`, so the fork notes never leak into a PR). Contains only the fix and its tests. No version bump, no fork-local tooling, no drive-by cleanups.
- `deploy`: generated, never hand-edited, force-pushed. Equals `upstream/master` plus a no-ff merge of every live `fix/*` branch plus one version-stamp commit. This is what the Docker image builds from.

The rule that makes it work: `deploy` is disposable. It is rebuilt from scratch on every refresh rather than merged forward, so it never accumulates drift, never carries a stale copy of a fix branch, and is always sitting on current upstream.

## Never publish upstream without asking

Ask Aaron before anything reaches `python-ring-doorbell/python-ring-doorbell`: opening or reopening a PR, editing a PR title or description, commenting on a PR or issue, or force-pushing a branch that an open PR tracks. That last one is the easy mistake, because the push targets `origin` and updates the upstream PR as a side effect.

The rule holds even when the change is obviously correct, and even when an earlier PR in the same session was approved. Approval to write a fix is not approval to publish it. Preparing the branch, running the checks, and drafting the PR body are all fine unasked. Stop at the point where a stranger would see it.

## Regenerating deploy

```bash
git config rerere.enabled true   # once: replay merge conflict resolutions across rebuilds

git fetch upstream
git checkout -B deploy upstream/master
for ref in $(git for-each-ref --format='%(refname:short)' refs/heads/fix/); do
  git merge --no-ff "$ref" -m "deploy: merge $ref"
done

# Version stamp lives here, never on a fix branch. Format: <HA's pinned base>+aw.<n>
# Read the base from the running image, not from memory or a local core checkout:
#   docker run --rm ghcr.io/aaronwolen/hass-patched:stable python -c \
#     "import json;print(json.load(open('/usr/src/homeassistant/homeassistant/components/ring/manifest.json'))['requirements'])"
sed -i '' 's/^version = .*/version = "0.9.14+aw.1"/' pyproject.toml
git commit -am "deploy: stamp 0.9.14+aw.1"

uv run pytest                    # textually clean merges can still be semantically broken
git push -f origin deploy
```

Keep this script in `aaronwolen/pcserver-docker-stack`, not here. Anything fork-local committed to this repo has to be carried across every upstream rebase.

Retiring a fix: when a PR lands upstream, `git branch -D fix/<slug>` and regenerate. Upstream master already carries it, so `deploy` picks it up for free.

## How this reaches production

The consumer is `aaronwolen/pcserver-docker-stack`: `hass-image/Dockerfile` builds stock HA plus this fork, CI (`.github/workflows/hass-image.yml`) publishes it to `ghcr.io/aaronwolen/hass-patched`, and compose pulls `:stable`. See `docs/hass-image.md` there. Ring tracking issue is #15 in that repo.

That build runs weekly with `pull: true` and `no-cache: true`, so it deliberately re-resolves both the HA base image and the fork branch on every run, and tags each result with a timestamp. Reproducibility lives in the image tag, not the git ref, which is why the Dockerfile can point at the moving `deploy` branch rather than a per-build tag.

## Version stamping and the HA pin

Home Assistant reinstalls its pinned library over the fork at integration setup unless the installed version satisfies the manifest pin: `homeassistant/util/package.py:is_installed` evaluates `req.specifier.contains(installed_version)`, and `homeassistant/requirements.py` pip-installs on failure. That reinstall happens inside the running container, so the image builds green and the fix disappears at runtime.

PEP 440 ignores a candidate's local version segment when the specifier has none, which is the lever:

```text
ring-doorbell==0.9.14   installed=0.9.14+aw.1   -> True
ring-doorbell==0.9.14   installed=0.9.15        -> False
```

Stamping `deploy` as `<HA's pinned base>+aw.<n>` therefore satisfies the pin with no patching of HA source at all. Two rules follow:

- Never reuse a real upstream version number. Stamping the fork `0.9.15` means that when upstream actually releases 0.9.15 and HA pins it, the fork silently passes the check while masquerading as a release it is not, missing everything else that went into it. `+aw.<n>` cannot collide.
- Assert the pin at build time, because every drift failure is otherwise silent. Read the pin from the image and check it with HA's own function:

```dockerfile
ARG RING_REF=deploy
RUN set -eux; \
    pip install --no-cache-dir --force-reinstall --no-deps \
      "ring-doorbell @ git+https://github.com/aaronwolen/python-ring-doorbell@${RING_REF}"; \
    python /verify_ring_pin.py
```

```python
# verify_ring_pin.py
import json, sys
from importlib.metadata import version
from homeassistant.util.package import is_installed

manifest = "/usr/src/homeassistant/homeassistant/components/ring/manifest.json"
pin = next(r for r in json.load(open(manifest))["requirements"]
           if r.startswith("ring-doorbell"))
installed = version("ring-doorbell")
if not is_installed(pin):
    sys.exit(f"FAIL: HA pins {pin}, fork installs {installed}. "
             "Re-cut deploy on current upstream and re-stamp the +aw base.")
print(f"OK: HA pins {pin}, fork installs {installed}")
```

This fails the weekly CI build the week HA core bumps its pin, which is exactly when `deploy` needs re-cutting on newer upstream. A bare `sed -i` on the manifest cannot do that: it exits 0 when it matches nothing.

`--no-deps` is correct for holding the rest of HA's dependency set still, but it also means a new runtime dependency added on a fix branch will not be installed. Fix branches should avoid adding dependencies.

## Current state

- `add-indoor-cam-plus`: adds Ring Indoor Cam Plus (`stickup_cam_mini_v3`) support. Open upstream as PR #531 since 2026-06-18. Validated against three real devices. Predates this convention: still carries a 0.9.15 version bump that belongs on `deploy`, and is not yet named `fix/indoor-cam-plus`.
- No other divergence from upstream.
- `hass-image/Dockerfile` currently pins `RING_REF=add-indoor-cam-plus` and rewrites the manifest pin with `sed`. Both go away when `deploy` exists: point `RING_REF` at `deploy` and replace the `sed` with the build-time assertion above.

## Device support: how new cameras break

New Ring hardware fails silently rather than loudly. Devices are grouped by `family` in `ring.py` (`doorbots`, `authorized_doorbots`, `stickup_cams`, `chimes`, `other`), so an unknown camera still becomes a `RingStickUpCam`, but:

- `model` falls through to `"Unknown Stickup Cam"` and logs an error.
- `has_capability()` returns `False` for everything except `HISTORY`, so Home Assistant creates almost no entities for it.

Adding a kind means touching three places, all in the pattern established by the existing indoor cam variants:

- `ring_doorbell/const.py`: a new `*_KINDS` list with the API `kind` string.
- `ring_doorbell/stickup_cam.py`, `model`: a branch returning the marketing name.
- `ring_doorbell/stickup_cam.py`, `has_capability`: add the constant to each capability list the device actually supports (`SIREN`, `MOTION_DETECTION`/`VIDEO`, `LIGHT`, `BATTERY`).

Doorbells follow the same shape in `doorbot.py`. This kind-list design is the reason every new SKU requires a library release, which is the structural weakness worth keeping in mind if a rewrite ever happens.

## Layout

- `ring_doorbell/ring.py`: session, device fetch, device collection.
- `ring_doorbell/doorbot.py`: doorbells, history, recordings, live view.
- `ring_doorbell/stickup_cam.py`: cameras, lights, siren.
- `ring_doorbell/chime.py`, `other.py` (intercoms), `group.py`.
- `ring_doorbell/listen/`: FCM push notification listener, the source of real-time ding/motion events.
- `ring_doorbell/const.py`: device kinds, endpoints, capability enum.
- `tests/`: pytest with `aioresponses` and JSON fixtures in `tests/fixtures/`. Sockets are disabled by `pytest-socket`, so tests must never hit the network.

## Commands

```bash
uv sync                      # install dev environment
uv run pytest                # full test suite
uv run pytest tests/test_ring.py -k indoor
uv run ruff check .          # lint (ruff select = ALL)
uv run ruff format .
uv run mypy ring_doorbell    # strict: disallow_untyped_defs
uv run pre-commit run -a     # what CI effectively runs
```

Target is Python 3.9+, so no `match`, no PEP 604 unions at runtime without `from __future__ import annotations`.

## Conventions

- Match upstream style exactly. Anything that looks like fork-local taste makes the PR harder to merge.
- Do not reformat, refactor, or "improve" unrelated code in a fix branch.
- Keep fixes small and independently rebasable. Two fix branches that touch the same lines will conflict on every `deploy` rebuild, which is what `rerere` is for, but the better answer is to scope them apart.
- Fixture-backed tests only. There is no way to exercise real Ring hardware in CI.
