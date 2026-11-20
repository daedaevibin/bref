# Bréf

**Private correspondence, delivered without a middleman.**

An end-to-end encrypted messenger that talks device to device: no accounts, no
phone numbers, and **no server that holds your contact list or your message
contents**. Peers find each other by public key, punch through NATs when they
can, and fall back to a stateless relay when they cannot. An always-on desktop
node holds your queue for when your phone is asleep, so nobody has to run a
mailbox service for you.

There is still a relay, and it still sees IP addresses and timing. What it
cannot see is who you are, who you know, or what you said. `DESIGN.md` §8 states
exactly what is and is not protected — read it before quoting a claim from this
page.

> **Status: design phase.** There is no code yet, and that is deliberate. The
> decisions are being settled in `DESIGN.md` before anything is written, because
> half of them are expensive to change later.

**The name is provisional.** Bréf is Old Norse/Icelandic for *letter*, chosen to
match the rest of the family (Voix, Ljod, Galdr, Skuld, Verdandi). It renames in
one commit.

## Platforms

| Platform | Shell | Status |
| --- | --- | --- |
| Linux | GTK4 + relm4 (Rust) | primary target |
| Android | Kotlin + Jetpack Compose | primary target |
| Windows | same GTK4 shell via MSYS2 | undecided for v1 — see §7 |

## Stack, in one table

| Layer | Choice | Why |
| --- | --- | --- |
| Transport | [iroh](https://iroh.computer) | dial by public key, QUIC, hole punching, stateless relay fallback |
| Crypto | `vodozemac` (Olm) | Apache-2.0; `libsignal` is AGPL and incompatible with this licence |
| Storage | SQLite via `rusqlite` | append-only, hash-chained message log |
| Core | one Rust crate | protocol + crypto + storage, tested with `cargo test`, no UI |
| Desktop UI | GTK4, **no libadwaita** | native widgets, real text and IME, instant startup |
| Android UI | Jetpack Compose | the platform's own toolkit |
| Bridge | UniFFI | desktop links the crate directly; only Android crosses a boundary |

## Read this first

**[`DESIGN.md`](DESIGN.md)** is the document under review. It contains the
locked decisions (§1), the architecture (§2), the token system with two visual
profiles and measured contrast (§3), component contracts for the six screens
(§4), motion (§5), anti-patterns (§6), and the open questions (§7).

If you are reviewing: **§7 is where your input changes the design**, and §1 is
where it does not — those were argued and closed on purpose.

## What v1 is not

Groups. Multi-device. Push notifications. Voice, video, or media beyond images.
Search. Any server we have to operate.

## Licence

Open Software License v3.0 — see [`LICENSE`](LICENSE).
