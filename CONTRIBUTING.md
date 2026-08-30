# Writing a runbook

House rules for every Markdown file in this repository. A runbook that follows them can be handed to
someone who has never seen the matching video and still works, end to end.

## Repository layout

```text
runbooks/
├── README.md                  index — every runbook and reference document is linked here
├── CONTRIBUTING.md            this file
├── <runbook-name>/
│   ├── README.md              the runbook itself
│   └── screenshots/           optional, only when screens carry what prose cannot
└── reference/                 conventions, design decisions and inventories — not procedures
```

- One runbook per folder, folder name in kebab-case, the document always named `README.md`.
- Folders live flat at the repository root — never nested under a theme folder. **A folder path becomes a
  permanent URL the moment a video description points at it.** Themes are expressed in the index, where
  they cost nothing to change.
- A new runbook is not finished until it is linked from `README.md`.

## Language and typography

- **English, always**, whatever language the work was discussed in.
- **Sentence case for headings**: "Scope and dependencies", not "Scope and Dependencies". Proper nouns and
  product names keep their capitals: "Register NPS in Active Directory".
- Server names, file paths, commands, interfaces, group names and SSIDs go in backticks: `U01PARVMNPS01`,
  `GG-U01-PAR-WiFi-Mobile`, `/etc/hostapd/hostapd.conf`.
- Fenced code blocks carry a language hint (`powershell`, `bash`, `text`).
- Reference another document by relative path so the link survives: `radius-nps-deployment/README.md`,
  `reference/server-naming-convention.md`. Never an absolute `github.com` URL to this repository.
- Names must follow `reference/server-naming-convention.md`. Check it before inventing one.
- **The title is an imperative**: "Create a Broadcom account", not "Creating a Broadcom account".
- **Placeholders go in angle brackets, in PascalCase**: `<CaName>`, `<SamAccountName>`, `<CertificateName>`.
  A reader must never be able to run a command with a placeholder still in it — say so explicitly next to
  any command where substituting the wrong thing would do damage.

## Commands

**Every command block is preceded by the machine it runs on**, in bold, on its own line:

    **On `U01PARVMDOM01`:**

    ```powershell
    Get-ADDomainController -Filter *
    ```

Running the right command on the wrong host is the most expensive way a runbook fails, and it is the one
failure the document itself can prevent. A runbook that only ever touches one machine says so once, in
`Prerequisites`, and may then omit the marker.

Flag a command that cannot be undone — a reboot, a deletion, a `sudo` that rewrites a config — in bold on
the line before it, saying what it destroys.

## What never goes in this repository

This repository is public, and Git keeps history: a secret committed once stays retrievable even after it
is removed in a later commit. There is no clean fix, only key rotation. So, never:

- a real password, RADIUS shared secret, pre-shared key, API key or token — use `<SharedSecret>` and tell
  the reader to generate their own
- a private key, `.pfx`, or an exported certificate containing one
- a real tenant ID, subscription ID or object GUID — use `<tenant>`
- a real person's name, email address, or account identifier

Lab IP addresses, server names, group names and SSIDs are fine: they describe a fictional company on an
isolated lab network, and a reader needs them to follow along.

## Screenshots

Only when a screen carries what prose cannot — a third-party portal that changes its layout, a dialog
with no command-line equivalent. Never as a substitute for writing the step out.

- One `screenshots/` folder inside the runbook's own folder.
- `NN-short-description-in-kebab-case.png`, numbered in the order they are captured.
- PNG. Crop to the part that matters; never paste a full desktop.
- Redact anything the section above forbids, in the image itself.

## Scope of a runbook

**A runbook covers its own subject and nothing else.** It may point at a neighbouring runbook by relative
path when it depends on it, but it does not absorb that neighbour's material. When a topic genuinely spans
several runbooks and needs to be stated once, it gets its own dedicated document rather than being repeated
or bolted onto one of them.

## Section numbering

**Every `##` section is numbered, from 1 to N, with no gaps and no exceptions** — including
`Scope and dependencies`, `Prerequisites` and `References`.

One rule, no judgement call about whether a given section counts as a step. It makes every section
addressable by number in a cross-reference, it keeps long runbooks navigable, and it makes the checklist
below mechanically verifiable: the numbers either run 1 to N without a gap, or they don't.

Sub-sections take the number of their parent (`### 4.1`, `### 4.2`). Step titles should match the
chapters of the matching video, word for word.

## The runbook template

Adapt the following:

    # <Verb> <what it builds>

    <Two or three paragraphs of prose. What this runbook builds, and how it sits against its
    neighbours — cite them by relative path. No "Overview" heading; the reader is already here.>

    ## 1. Architecture

    <An ASCII diagram in a ```text fence: hosts with their names and addresses, the network
    segments, and what connects to what. It answers "what am I looking at" before any prose does.
    See `ad-cs-pki-deployment/README.md` and `hostapd-wifi-access-point/README.md`.>

    ## 2. Scope and dependencies

    <What this runbook does, and explicitly what it does not do. Where the excluded work happens
    instead. This is the section that stops a reader from expecting the wrong outcome.>

    ## 3. Target configuration

    | Item | Value |
    | --- | --- |
    | Server name | `U01PARVMxxx01` |

    ## 4. Prerequisites

    <Exact versions and states, not "a recent Windows Server". What must already be up and verified.>

    ## 5. Initial manual preparation

    <Optional. Work that has to happen before step 1 and is not itself part of the procedure.>

    ## 6. <First action>

    ## 7. <Second action>

    ## 8. Expected final state

    <What a reader must be able to observe if everything worked. Commands and their expected output.>

    ## 9. Update the infrastructure inventory

    <Only when the runbook adds or changes a VM. Points at `reference/vm-inventory.md`.>

    ## 10. Next step — <what follows>

    <Optional. The work this runbook deliberately stops short of, and where it continues.>

    ## 11. References

    - [Title — Microsoft Learn](https://learn.microsoft.com/...)

    ---

    *Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*

### Permitted deviations

- **Phases and checkpoints** as section titles, when a build spans several servers and has genuine points
  of no return. `ad-cs-pki-deployment/README.md` is the reference case. The sections are still numbered
  1 to N like any other — the deviation is what a section *is*, never whether it carries a number.
- **A long click-through with screenshots**, for a third-party portal that cannot be automated or
  usefully described in prose. `create-broadcom-account/README.md` is the reference case. Such a runbook
  has no `Architecture` section — it builds no infrastructure — and says so in one line under
  `Scope and dependencies`.

Everything else — numbering, heading style, front matter, `References`, the footer — applies without
exception, including to a deviating runbook.

## The `reference/` template

Design documents, not procedures. Lighter:

    # UnreadLines — <subject>

    <One paragraph: what this document governs, and who has to follow it.>

    ## 1. <Section>

    ## 2. <Section>

    ## 3. References

    ---

    *Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*

Same numbering rule as a runbook: every `##` section numbered, 1 to N. Other documents cite these by
number (`reference/server-naming-convention.md` §4), so a number that moves breaks them — check with
`grep -rn "§" .` before renumbering an existing reference document.

## Checklist before publishing

- [ ] Headings are in sentence case throughout.
- [ ] Every `##` section is numbered, 1 to N, with no gaps.
- [ ] Every `§N` cross-reference still points at the section it means.
- [ ] `Architecture`, `Scope and dependencies`, `Prerequisites` and `References` are all present.
- [ ] Every command block says which machine it runs on.
- [ ] No secret, private key, real tenant ID or real person's name anywhere — including inside screenshots.
- [ ] Every relative link resolves.
- [ ] Names match `reference/server-naming-convention.md`.
- [ ] `reference/vm-inventory.md` is updated if a VM was added or changed.
- [ ] The footer signature is present, copied exactly.
- [ ] The runbook is linked from `README.md`, in the right theme group and in dependency order.

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
