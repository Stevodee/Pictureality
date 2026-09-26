# Pictureality

A new image format that is navigable and linked internally to more information and perspectives.

Self-hosted 360° panorama tour site. Three.js + Panolens.js viewer, GitHub Pages
hosting, custom `admin.html` for content management. No backend server, no build step.

Live site: https://stevodee.github.io/Pictureality/
Admin page: https://stevodee.github.io/Pictureality/admin.html
Repo: `stevodee/Pictureality` (public)

**Status: frozen proof-of-concept.** UISP (`stevodee/uisp`) is the architectural
successor and gets new platform-level work going forward. Pictureality stays as a
pinned, stable reference until UISP's viewer reaches hotspot-authoring parity, at
which point Gibson House migrates over and this repo is retired rather than
rewritten in place.

---

## 1. Architecture

Two static HTML files, no build step, no server-side code:

- **`index.html`** — original single-tour viewer, reads only `config.json`. Kept as
  dormant fallback.
- **`viewer-multi.html`** — active viewer file; all new feature work targets this
  file. Confirmed via direct diff that its only intentional difference from
  `index.html` is per-tour URL loading (`published/<tour-id>.json` via
  `?tour=<id>`) vs. a single shared `config.json`. All bug fixes and the idle
  ambient drift feature are identical in both files.
- **`admin.html`** — content-management tool, talks directly to the GitHub REST
  API from the browser using a session-only Personal Access Token (never
  persisted — not in the file, not in localStorage).

Data files (in the repo, not in the HTML):

- `panos/` — panorama image files.
- `tours.json` — the full tour library.
- `config.json` — the single tour published under the original single-tour model.
- `published/<tour-id>.json` — per-tour published files under the new
  `viewer-multi.html` model. Presence of the file means the tour is live;
  deletion unpublishes it.

Pinned dependency versions (unchanged, do not update independently):

```
three.js r105
panolens@0.11.0
```

Panolens' upstream repo is confirmed archived (June 2023) — no further upstream
fixes are coming, which is part of why Pictureality is frozen and UISP's Phase 3
renderer work uses Photo Sphere Viewer instead.

---

## 2. Feature status

### Done

- Per-tour unique URLs: each published tour writes to `published/<tour-id>.json`;
  `viewer-multi.html` reads the `?tour=` query param. Confirmed working against
  the Gibson House tour. **Not yet done:** deep linking to a specific panorama
  *within* a tour — the current URL scheme only addresses the tour, not a
  starting panorama inside it.
- Camera FOV raised to 75° (from Panolens' default 60°).
- **Whip-pan bug fixed and committed** (`a23476b`, Sept 2). The originally
  suspected cause — `Math.atan2` reconstruction not matching Panolens' internal
  `calculateCameraDirectionDelta` convention — was wrong. The actual cause: an
  X-axis double-negation. `tweenControlCenter` already negates X internally
  (mesh-mirroring convention), and `relevel()` was passing raw camera-direction
  X straight through, so it tweened toward a horizontally mirrored target
  heading. Fix negates X before passing it in. Also added a fallback heading for
  near-zenith/nadir looks so the camera doesn't visibly stick tilted.
- OrbitControls momentum (`momentumDampingFactor`) zeroed to stop it fighting
  custom tweens.
- `pointerup` rebound to `window` instead of the container (was silently
  dropping releases that landed on UI chrome) — fixed in both `index.html` and
  `admin.html`.
- `Infospot.hide()` was permanently killing raycasting on markers across
  panorama switches — fixed by pairing with `.show()` on re-entry.
- Idle ambient drift implemented for `viewer-multi.html`: after a drag, the
  camera slowly continues panning in the last-dragged direction once the
  relevel tween finishes. Uses a 2D XZ cross product for sign detection and
  reuses `tweenControlCenter` (same X-negation convention as the relevel fix).
  Fully wired and sequenced after `autoRelevel` via delay math
  (`RELEVEL_DELAY_MS + RELEVEL_TWEEN_MS`) — done, no merge step pending.

### In progress / partially done

- Merged relevel+drift module for `viewer-multi.html` — still separate pieces.
- Info-marker click-vs-relevel interaction — flagged as a concern, not yet
  addressed.

### Superseded — will not be built here

- **Admin thumbnail generation.** Originally: admin loads full-resolution
  panoramas just to render small previews — slow. Decision: this will not get
  its own infrastructure (no resize proxy, no canvas-generated thumbnail files).
  Gibson House's panoramas are being migrated onto the same ImageKit
  `thumb`/`mobile`/`full` tiering used by UISP, ahead of the eventual PSV/VR
  cutover; `admin.html`'s previews will point at the `thumb` tier once that
  migration lands, solving admin load-speed as a side effect of work already
  planned rather than as its own build.

### Deferred indefinitely — superseded by UISP if revisited

These were on the original backlog. The repo being frozen means none of them
get built here; noted so the scope isn't silently lost, not as active plans.

- North marker / heading orientation: confirmed via exiftool that
  `GPano:PoseHeadingDegrees` is absent from all Gibson House panoramas.
  Sensor Logger magnetometer correlation is the planned path forward, to be
  validated with a real-world test shoot against `geo-correlate.html`.
- Directional hover-thumbnail markers — blocked on arrival-heading logic
  validation in real use.
- Text title / floating text field within a scene.
- 2D (non-360) photo support within a tour.
- Spatial audio.
- Video hotspots + timeline. (UISP's `SCENE_SPEC.md` now defines a related but
  more general resumable video-detour pattern — see that repo if this comes up
  again.)
- Image-prep track (not site code): cinemagraph process, nadir/navi picture
  creation, reading embedded photo metadata (EXIF/GPS/capture date).
- Merging the admin page's background into a live, always-on interactive canvas
  — explicitly paused pending the VR-oriented UI think-through (see Section 4).
- Full cutover from `index.html`/`admin.html` to the `-multi` variants.
- VR/immersive mode — deliberately sequenced last since it changes the entire
  interaction model, not just visuals; this is the actual trigger for retiring
  Pictureality in favor of UISP's PSV-based viewer.

### Delivered utilities

- `tour-node-map.html` — prototype showing the Gibson House tour graph, flags
  missing return links. Follow-up on what to do with those flags is TBD.

---

## 3. Credentials and git — housekeeping

- Two GitHub Personal Access Tokens were exposed in chat during a past
  troubleshooting session and should be treated as burned/revoked.
- Credential issues around this were resolved by disabling the `osxkeychain`
  helper and embedding tokens directly in the remote URL instead.

---

## 4. Design decisions worth preserving

- **Config split** (tour library vs. published tour) exists specifically so
  multiple tours can be built/edited without risk to what's currently live.
- **Token security model**: the admin page never persists the GitHub token
  anywhere, because the repo is public. Pasted fresh each session, held only in
  a JS variable.
- **Fullscreen hotspot editor**: was originally a small embedded panel; changed
  to fullscreen specifically to eliminate container-sizing bugs (Panolens
  doesn't handle small/oddly-shaped viewport containers gracefully) and to give
  a bigger, more accurate click target.
- **Marker icons** are generated at runtime via canvas, not hosted image files —
  avoids asset management for a handful of small colored glyphs.
- **Additive-only prototyping**: `viewer-multi.html` was built as a full copy of
  `index.html` with additive-only changes, leaving the original untouched until
  a deliberate cutover decision. Confirmed via direct diff that this discipline
  held.

**Open design question (paused):** could the admin page's background become a
live, always-on interactive panorama canvas (marker placement directly in the
background, no modal), rather than the current "click Edit Hotspots to open a
fullscreen modal" flow? Paused pending the VR-oriented UI design work —
pointer-based desktop interaction and VR interaction (controller ray/gaze) are
different enough that redesigning the admin canvas now risks redoing it once VR
support is actually tackled. Given the repo is frozen, this is unlikely to be
picked up here at all — it's the kind of question UISP's admin/authoring layer
should answer instead.
