# DESIGN.md

# Bréf — design & system architecture

**Name is provisional** (Bréf, Old Norse/Icelandic for *letter*). Rename the
directory freely; nothing in this document depends on it.

**Status:** design-locked, implementation not started.
Two developers: desktop shell + visual language (Veridian Zenith), Android +
Windows build (Xtrios09 / Abhijeet Prabhakar).

---

# 1. Locked decisions

These are settled. Reopening one requires a written reason, because each was
argued at length and each closes off a whole class of work.

| Decision | Status | Why |
| --- | --- | --- |
| GTK4 for Linux + Windows desktop | **Locked** | Native widgets, real text shaping, real IME, ~50 ms cold start |
| No libadwaita, ever | **Locked** | It enforces GNOME's HIG. Our visual language is ours, not GNOME's |
| No Flutter, no Dart | **Locked** | Self-drawn engine; text and input never quite match the system |
| No Compose Multiplatform on desktop | **Locked** | Same engine as Flutter (Skia/Skiko), same objection |
| No Qt, no QML | **Locked** | Not wanted, even though Budgie 11 is moving there |
| No WebView / Electron / Tauri UI | **Locked** | "Feels like a web app" is the thing we are avoiding |
| Jetpack Compose on Android | **Locked** | It is the platform's own toolkit, not a cross-platform engine |
| Rust core shared by both shells | **Locked** | One implementation of protocol + crypto, testable with `cargo test` |
| UniFFI is the only FFI boundary | **Locked** | Desktop links the crate directly; only Android crosses a boundary |

## Non-goals for v1

Groups. Multi-device. Pushers/FCM. Voice or video. Media beyond images.
Message search. Themes beyond the two profiles below. Anything with a server
we have to operate.

---

# 2. Architecture

```
crates/core        Rust: identity, ratchet, message log, storage, protocol
crates/core-ffi    UniFFI wrapper (Android only)
desktop/           relm4 + gtk4 (no libadwaita)  -> Linux, Windows
android/           Kotlin + Jetpack Compose      -> Android
```

- `desktop` depends on `crates/core` as a normal path dependency. Same
  process. No bridge, no codegen, no serialisation across a boundary.
- `android` links `libbref_core.so` built by `cargo-ndk`, bound via UniFFI.
- The UI is a **renderer of core state**, not an owner of it. Both shells ask
  the core for a snapshot and render it. No business logic in a shell.

Threading: GTK is not `Send`/`Sync`. Tokio runs on its own thread; results
reach the UI through an `async-channel` consumed by `glib::spawn_future_local`.
Never touch a widget off the main loop.

---

# 3. Token system

Three layers, exactly as in `meshiji`. No widget may use a raw colour, a raw
spacing value, or a raw duration.

**Layer 1 — primitive.** Raw values. Profile-independent where possible.
**Layer 2 — semantic.** Named by role (`surface.raised`, `text.secondary`).
**Layer 3 — component.** Per-widget (`messageRow.hover`, `composer.tray`).

## 3.1 Single source of truth

Tokens are data, emitted to both platforms from one table:

- `desktop/resources/tokens.css` — GTK4 CSS custom properties + classes
- `android/.../DesignTokens.kt` — a Kotlin `object` of `Color`/`Dp` constants

Neither file is edited by hand. Edit the table, regenerate. A hex value that
exists in two places is a bug.

## 3.2 Contrast (measured, WCAG 2.1)

Measured against `#050505`. This is a hard gate, not a preference: a
messenger whose secondary text fails AA is a messenger people misread.

| Token | Value | Ratio | Verdict |
| --- | --- | --- | --- |
| `text.primary` | `#FFFFFF` | 20.3 : 1 | AAA |
| `text.secondary` | `#888888` | 5.75 : 1 | AA |
| `text.muted` (corrected) | `#7A7A7A` | 4.75 : 1 | AA |
| `text.muted` (original) | `#555555` | 2.73 : 1 | **fails AA and AA-large** |
| `text.disabled` | `#555555` | 2.73 : 1 | decorative only, never load-bearing |
| `accent.primary` | `#E50914` on `#050505` | 4.0 : 1 | fine for large text and fills, not body |

`#555555` moves from `text.muted` to `text.disabled` and may only be used for
separators, disabled glyphs and decorative marks. `#7A7A7A` becomes
`text.muted`. (An earlier note suggested `#6E6E6E`; that measures 4.0:1 and
was wrong — `#7A7A7A` is the correct floor.)

## 3.3 Profiles

One widget tree, two stylesheets, one setting. `default` ships; `amber` is
opt-in and is explicitly a taste, not an improvement.

| Property | `default` | `amber` |
| --- | --- | --- |
| Philosophy | gets out of the way | precision instrument |
| `background.base` | `#141414` | `#050505` |
| `background.sunken` | `#0F0F0F` | `#000000` |
| `surface.raised` | `#1C1C1C` | `#0F0F0F` |
| `surface.overlay` | `#232323` | `#141414` |
| `border.subtle` | `#2A2A2A` | `#222222` |
| `divider` | `#242424` | `#1A1A1A` |
| `accent.primary` | system accent | `#E50914` |
| `accent.danger` | `#C62828` | `#8B1515` |
| Elevation model | 1 px border, no shadow | layered surfaces + blur |
| `radius.base` | 8 px | 16 px |
| `radius.small` | 4 px | 8 px |
| Signature layer | none | `GtkGLArea` particles, blur on 2 surfaces |
| Motion | 120 ms only | 120 / 200 / 320 ms |

`default` uses the system accent and follows `gtk-application-prefer-dark-theme`
plus the freedesktop dark preference. On Budgie, GNOME, KDE and Hyprland it
should look like it belongs without configuration.

`amber` uses blur **only** on `composer.tray` and `sidebar` via
`snapshot.push_blur()`. Never on message rows: blur behind text is the fastest
way to make a readable app unreadable.

---

# 4. Component contracts

Each contract lists states, the tokens each property consumes, and what is
forbidden. "Forbidden" lines are the ones that keep two implementers agreeing.

## 4.1 Conversation list

- **Structure:** `gtk::ListView` over `gio::ListStore`. Virtualised. No
  `ListBox` for message-bearing lists.
- **Row:** 56 px tall, 8 px horizontal padding, avatar 32 px, name
  `text.primary` 14 px medium, preview `text.secondary` 13 px, timestamp
  `text.muted` 12 px right-aligned.
- **States:** rest, hover (`surface.raised`), selected (`accent.primary` at 12%
  over base, 2 px leading accent bar), unread (name semibold, dot 6 px), focus
  (2 px `accent.primary` ring, offset 2 px, never a colour change alone).
- **Forbidden:** per-row shadows; per-row blur; colour as the only unread
  indicator.

## 4.2 Message thread

- **Structure:** `gtk::ListView` + `SignalListItemFactory`. Sticky day
  separators. Reverse-anchored scroll.
- **Bubble:** max width 68% of thread width, radius `radius.base` with a 4 px
  tail on the sender side, padding 10/14, 4 px vertical gap within a group,
  12 px between groups.
- **Own messages:** `accent.primary` at 18% over `surface.raised`; text
  `text.primary`. **Their messages:** `surface.raised`; text `text.primary`.
  Never colour the text of a bubble; contrast must survive both profiles.
- **Delivery state:** clock (queued), single check (sent to peer), double check
  (peer acknowledged), warning glyph (`accent.danger`) for failed. 12 px,
  `text.muted`, bottom-right inside the bubble.
- **Forbidden:** gradients inside bubbles; animated bubbles on arrival beyond
  opacity 120 ms; timestamps inside every bubble (only on group boundaries and
  on hover).

## 4.3 Composer

- `gtk::TextView` wrapped in a `ScrolledWindow`, min 1 line, max 6, then
  scroll. `radius.base`. `surface.raised` (or blurred tray in `amber`).
- Enter sends, Shift+Enter newline — desktop only; Android uses the IME action.
- Send button: 32 px, `accent.primary` fill, disabled at 40% until non-empty.
- **Forbidden:** auto-growing beyond 6 lines; send-on-blur; character counters.

## 4.4 Pairing

The one screen where the design carries security weight.

- Shows **your** identity as a QR code (endpoint id + prekey bundle) and a
  6-word fingerprint in grouped lowercase, e.g. `amber-runner-glass-tallow-nine-oath`.
- Scanning is the phone's job; the desktop displays. Paste-and-verify is the
  fallback for desktop-to-desktop.
- After scan: both sides display the same 6 words. **Confirm is a required
  click on both.** No auto-accept, ever.
- **Forbidden:** dismissing the fingerprint automatically; showing a shortened
  fingerprint; "trust this device" checkboxes; any flow that lets one side
  add a contact without the other side confirming.

## 4.5 Node status

- Lives in the sidebar footer, one line: `direct` / `relayed` / `offline`,
  with a 6 px dot and `text.muted` label.
- Expands to: endpoint id (truncated, copyable), relay in use, messages queued.
- **Forbidden:** modal alerts about connectivity. A messenger that interrupts
  to say it is reconnecting is a messenger that loses trust.

## 4.6 Settings

- `AdwPreferencesWindow` is unavailable (no libadwaita). Use a plain
  `gtk::Window` with `gtk::Box` rows, 1 px `divider` between groups, group
  headings `text.muted` 12 px uppercase 0.04 em tracking.
- Profile switcher lives here, labelled **Appearance: Default / Amber**, with
  one sentence of description each. No preview thumbnails in v1.

---

# 5. Motion

| Token | Duration | Curve | Used for |
| --- | --- | --- | --- |
| `motion.fast` | 120 ms | `ease-out` | hover, press, focus ring, send |
| `motion.standard` | 200 ms | `ease-out` | panel reveal, list insertion |
| `motion.emphasis` | 320 ms | `cubic-bezier(0.2, 0, 0, 1)` | profile switch, pairing confirm |

- `default` profile: `motion.fast` only. Everything else becomes instant.
- Nothing animates on arrival of a message except a 120 ms opacity fade.
- Respect `gtk-enable-animations = false` and GNOME's reduce-motion setting.
  When disabled, all durations become 0 — not "faster".
- **Forbidden:** spring/overshoot curves; animation longer than 320 ms; any
  loop that runs when the window is not focused.

---

# 6. Anti-patterns

- Default Adwaita styling or GNOME header bars leaking in (we are not using
  libadwaita; do not reimplement it by accident).
- Hardcoded hex values, spacing, or durations anywhere outside the token files.
- Blur or particles behind body text.
- Colour as the sole carrier of meaning (unread, error, selection).
- Modal dialogs for background events.
- Auto-playing motion in the `default` profile.
- Custom drawing where a native widget exists: use `GtkTextView`, `GtkListView`,
  `GtkPopover`, `GtkFileDialog` before writing a `snapshot()`.
- Any second source of truth for a token.

---

# 7. Open questions

1. **Node mode:** separate `bref-node` binary, or a flag on `desktop`? Affects
   packaging, autostart and the tray.
2. **Windows in v1:** GTK4/MSYS2 build (consistent, foreign-looking) or defer.
   Owner: Xtrios09.
3. **Avatar policy:** generated geometric marks from the endpoint id (no upload,
   no network) versus user-chosen images. v1 proposal: generated only.
4. **Message log merge:** hybrid logical clock parameters and conflict display
   for concurrent edits on the same conversation. Needs a written decision
   before the first commit to `crates/core`.

---

# 8. Threat model and metadata

Any claim the README makes has to be defensible here. Reviewers will ask
"protects against whom?", and "nobody" is not an answer.

## 8.1 What is protected

| Adversary | Protected | Not protected |
| --- | --- | --- |
| Relay operator | message contents, contact list, message history, account identity (there are no accounts) | IP addresses of both peers, timing and volume of traffic, the fact that two endpoints exchanged packets |
| Anyone who obtains the relay's database | everything — the relay stores nothing | — |
| Network observer (ISP, passive) | message contents, identity of peers beyond IP addresses | the fact that a connection exists at all |
| Someone who steals the phone | message contents, unless the device is unlocked | decrypted history while unlocked |
| Malicious contact | nothing beyond ordinary E2EE | your replies, your existence, your timing |

## 8.2 What is explicitly not claimed

- **No anonymity.** Traffic analysis against a relay, or against an ISP, can
  identify peers. Onion routing is not implemented and is not planned.
- **No protection against a global passive adversary.** Fine-grained timing
  correlation between ingress and egress at one relay is possible in principle.
- **No metadata stripping on the path.** iroh relays see IP pairs. This is the
  price of working NAT traversal; hiding it would require onion routing, which
  we rejected for latency and battery reasons.
- **No push infrastructure**, therefore no third party is told when you receive
  a message. This is why v1 has no FCM/APNs and why Android may need a
  foreground service.
- **No forward secrecy for the message log at rest.** Persisted history is
  encrypted with a device key, not with the ratchet; a stolen unlocked device
  exposes it.

## 8.3 Wording rules for public text

Because the difference between "no server" and "no server that sees your
messages" is the whole argument:

- Say **"no server holds your contact list or message contents"**, never
  "no server" and never "serverless".
- Say **"stateless relay"**, because a stateless relay is a server, and
  claiming otherwise is the mistake ZappierV2 most likely made.
- Never claim anonymity, unlinkability, or "zero metadata".
- Pairing text must always name the 6-word fingerprint; a shortened fingerprint
  is a security bug, not a UI convenience.
