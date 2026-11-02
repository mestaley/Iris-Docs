# ISSUE-016: TV control is Plex-only — broaden to full TV control

- Status: open
- Severity: medium
- Area: actuation / tool-registry (media control) + Home Assistant entity coverage
- Discovered: 2026-11-02 (capability review)

## Summary
Iris's TV control currently reaches **Plex only** (play/pause/select content within Plex). The rest of
normal TV control is not wired: power on/off, input/source switching, launching other apps, TV volume,
navigation. So "Iris, turn on the TV" / "switch to <app/input>" / "turn the TV up" don't work.

## Desired
Full-ish TV control via the existing name-based actuation (ADR-005) + `control_media`/`control_device`,
however the TV is represented in Home Assistant:
- Power on/off.
- Input / source selection (HDMI, apps).
- Transport (play/pause/stop/next) beyond Plex.
- Volume / mute.

## Notes / unknowns (to resolve when picked up)
- How is the TV exposed in HA today — a `media_player` (Plex integration), plus a separate entity for
  the TV itself (CEC / Android TV / vendor integration / a harness like a Harmony/IR blaster)? Power and
  input usually live on a *different* entity than the Plex `media_player`.
- Which of the above HA actually exposes determines what's achievable vs what needs an HA-side
  integration first (not an Iris-side fix).
- Entity names need to be in `iris_mappings` so "the TV" resolves (ADR-005).
- Relates to the Music Assistant media-control work (same `control_media`/`select_content` path).

Filed 2026-11-02 (Matt) to track; not scheduled.
