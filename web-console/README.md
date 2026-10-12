# Web console for the installer (proposal)

Status: **design proposal + mockups** - no implementation in this pull request.

## Problem

Headless installs (ARM single-board machines, rack nodes without a console cable,
one-off lab boxes) currently need one of:

* a keyboard + monitor on the machine,
* a serial console and a TUI you drive through it,
* a pre-built `answer.toml` on a USB stick or a PXE-served answer file.

The unattended path is the right long-term answer, but it asks the operator to
author a TOML file offline before touching the machine - a heavy loop for "install
this one box". There is no middle ground that requires zero local peripherals and
zero pre-prepared media.

## Proposal

Add one installer boot option - **Install Proxmox VE (web console)** - that
boots the native text-mode installer and additionally runs a small HTTP service
in the initrd environment. Any browser on the same LAN opens the installer's own
console page, answers the same questions the text installer asks, and the flow
finishes through the **existing** auto-installer path. No keyboard, no display,
no USB stick, no pre-written TOML.

The console speaks plain HTTP on the machine's DHCP address and is protected by a
one-time pairing code (see below). It is intentionally boring: a static set of
pages plus one small daemon, no embedded UI framework, so it is trivially
auditable and does not balloon the initrd.

## How it maps onto what already exists

The installer core is already UI-agnostic: `Proxmox::UI::Base` defines the
question loop (`prompt`, `message`, `error`, `progress`, `display_html`) and the
repository already ships two backends - `Gtk3.pm` (graphical) and `StdIO.pm`
(text/serial). The web console is **a third backend, `Proxmox::UI::Web`**: the
same question sequence, the same field validations, the same progress stream,
rendered as browser pages. No parallel wizard definitions, no second form
model - one flow, three faces.

The boot option itself boots the **native text-mode installer**, exactly as the
serial-console entry does today; the HTTP front runs alongside it. The target
machine therefore keeps showing the ordinary text-mode screens (so anyone with a
monitor sees a normal installer - with the console URL and pairing code printed
at the bottom), while the browser drives the identical questions.

Downstream, nothing changes either: the configuration produced by the flow is
the same validated document the auto-installer consumes
(`proxmox-auto-installer < /run/automatic-installer-answers`, then
`proxmox-post-hook`), so the web path cannot drift from the file-driven path,
and the generated `answer.toml` can be retained for replay. `unconfigured.sh`
only gains one more branch beside the existing `$proxtui` / `$start_auto_installer`
ones. Failure of the daemon or a timeout at the pairing page drops back to the
existing TUI - the web console never becomes a single point of failure.

## Pairing

The console runs on a shared LAN before it knows who is allowed to drive the
install, so pairing gates every mutating request (reads of the status page may
stay open). Two codes are accepted, because "attach a screen and read the code"
is exactly the scenario headless users do not have:

1. **Session code** - six characters, printed to the console/serial as today.
2. **Machine code** - eight hex characters derived from the machine itself and
   printed by the bootloader **before** the OS environment exists, so it appears
   on the boot menu of a machine that will never show a console. Under UEFI the
   firmware RNG can draw a fresh code per boot; the stable fallback is the head
   of the SMBIOS type-1 UUID (`smbios --type=1 --get-uuid=7 --set=...` in GRUB,
   case-folded). The operator reads it from the boot screen once, from the
   chassis label, or over IPMI - no serial cable required.

A `--webconsole-nopair` (or answer-file) escape hatch covers automated fleets
where pairing is meaningless.

Worth being blunt about: transport is plain HTTP, so the session code travels in
clear text alongside the LAN and the machine code is only as secret as the
machine's label. The mitigations are scoping: the console listens on the DHCP
interface only, accepts no changes after install completion, dies on reboot, and
never carries credentials - the root password is posted once over the same
channel the TUI would have typed it on locally. A TLS-on-first-contact design
(STSN/PAM-style) is a possible follow-up, not a prerequisite.

## Mockups

Concept images are attached to the pull request (not committed, so this branch
stays reviewable as text). Styling follows the Proxmox corporate palette
(black/white with the orange accent) and the boot screen is the native text-mode
look. The pages mirror the TUI question order: keyboard and management address,
root password and SSH keys, target disk, then progress.

## Open questions

* Transport plumbing for `Proxmox::UI::Web`: the TUI blocks in `prompt` waiting
  on stdin; the web backend needs the same call to block on an HTTP answer
  instead. A single-flight question queue (one paired client at a time) keeps
  this simple, but it is the one genuinely new piece of machinery.
* i18n: the TUI ships translations; the console pages would need the same
  catalogs (or start English-only).
* IPv6-only management networks and 802.1X environments are out of scope of v1
  and should be stated loudly in the docs.
* Screenshots here are concept art; they should be replaced by rendered
  prototypes before any real patch series.

## Discussion

Per project convention, the mainline discussion belongs on the
[pve-devel mailing list](https://www.proxmox.com/en/about/company/details/mailing-lists) -
this pull request is a place to anchor the design and mockups, and I am happy to
open the list thread referencing it.
