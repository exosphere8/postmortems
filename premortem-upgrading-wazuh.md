# Pre-mortem: upgrading the lab's Wazuh

*From my [SIEM home lab](https://github.com/exosphere8/siem-home-lab). Unlike the other posts,
this one is written **before** the change: the upgrade has not happened yet. It lists what I
expect to go wrong, so I can check each item when it does.*

---

## The setup

The lab's detection content is 33 Wazuh rules in XML, 11 Sigma rules and 3 Suricata
signatures. Until today, the setup guides said "take the current 4.x version". That works
until the day "current" changes, and as of 7 October 2026 it is about to change twice:

- **4.14.9** is at release candidate. It is a routine patch release.
- **5.0.0** is also at release candidate
  ([rc1](https://github.com/wazuh/wazuh/releases/tag/v5.0.0-rc1), 5 October 2026). It is not
  routine at all.

The lab is not deployed yet, so nothing below has broken. That makes this the cheapest
time to think it through.

## The assumption I would have made

"Upgrading the SIEM is an operations task. The rules are content. Upgrade the packages,
redeploy the same rule files, done."

My last post, [Wazuh only remembers the last rule](wazuh-only-remembers-the-last-rule.md), is
the reason I don't trust that. The rules don't stand on their own. They are written against
how a specific Wazuh version decodes events, which built-in rule an event passes through,
and which rule it ends on. Change the version and you can change the rules' behaviour
without editing a single rule file.

## What I expect to go wrong within 4.x

**1. A rule keeps loading but stops matching.** 26 of the 33 rules hang off built-in Wazuh
rules or groups through `if_sid` or `if_group`. `wazuh-analysisd -t` checks that the
configuration loads. It cannot tell me that an event now ends on a new, deeper built-in rule
instead of the one my rule extends, or that a group was renamed. A rule that refers to a
group no event carries anymore is still a valid rule. This is the last post's bug again, with
the upgrade as the trigger instead of my own child rule.

**2. A field changes meaning in a patch release.** This has already happened: 4.14.1 made
`user` an alias of the `dstuser` static field
([#32107](https://github.com/wazuh/wazuh/pull/32107)). Rule 100103, failed SSH login as root,
matches on `<user>`. I don't know yet whether that change affects it. What I do know is
that a passing configuration test would not tell me.

**3. An agent ends up newer than the manager.** The Windows guide said "download the MSI that
matches the manager" and linked the packages list, which always shows the latest version.
The Linux agent installed whatever the 4.x repository offered that day. Install the agents a
week after the server and they can be newer than the manager, which Wazuh does not support.

**4. The indexer heap goes back to its default.** The server VM has 4 GB shared by the
indexer, the manager and the dashboard, which only works because the indexer heap is cut to
1 GB in `jvm.options`. Wazuh's upgrade guide says to back up that file and reapply your
settings by hand. If I miss that, the indexer and manager compete for memory, and the first
symptom will be a slow dashboard, not an error that names the cause.

**5. The upgrade looks like it ran, but didn't.** The packages are held with `apt-mark hold`
so they can't upgrade by accident. Held packages are kept back without an error, so an
upgrade I forget to un-hold for looks successful while still on the old version.

**6. There is no way back.** The lab's Hyper-V script turns automatic checkpoints off (to save
disk space), and downgrading Wazuh packages is not a supported path. Without a manual
checkpoint, a bad upgrade means rebuilding the server.

## What I expect to go wrong moving to 5.x

The [5.x migration guide](https://github.com/wazuh/wazuh-documentation/tree/5.0.0/source/migration-to-5x)
starts with the sentence that matters: 4.x cannot be upgraded in place. 5.x is a new
deployment, and **custom XML rules and decoders cannot be migrated**. 5.x rules are written
in Sigma format, checked against the Wazuh Common Schema, and run by detectors inside the
indexer. So "upgrade" is the wrong word. This is a rewrite plus a data migration.

**7. None of the 33 rules port directly.** I sorted them by how they work:

| Kind | Rules | Problem in 5.x |
|---|---|---|
| Extends a built-in 4.x rule or group | 26 | The 4.x built-in rules they extend do not exist in 5.x |
| Counts events or follows earlier ones (`frequency`, `if_matched_*`) | 4: 100101, 100102, 100201, 100504 | The 5.x rule documentation describes no counting or time window |
| Extends another custom rule | 3 | Their parents are in the two groups above |

The four stateful rules are the important ones: SSH brute force, a successful login after
that brute force, Windows password guessing, and web content discovery. They are the
rules that turn single events into a story.

**8. The Sigma rules look portable, but aren't quite.** I wrote them so the logic would carry
to other SIEMs, and Wazuh 5 now uses Sigma. But they use Sigma's own field names, such as
`EventID` for Windows events, and 5.x checks every field against its own schema and rejects
a rule that names a field it doesn't know. The two Sigma v2 correlation rules (Windows brute
force and password spraying) have no documented 5.x equivalent.

**9. The counting lesson may flip.** In 4.x, one event produces one alert: the deepest rule
wins. The 5.x documentation says each active policy gets its own copy of an event, so one
event can produce several findings. Counting groups, the fix from my last post, could then
count the same login attempt twice. I don't know yet. It's the first thing to test.

**10. Timing changes.** 5.x detectors evaluate events at configured intervals. 100102, the
"successful login after brute force" rule, relies on the order and timing of events within a
two-minute window. I need to measure how much delay intervals add before I trust any
time-window logic.

**11. The tooling around the rules breaks too.** `siemlab correlate` reads 4.x `alerts.json`
lines, while 5.x findings are schema documents in `wazuh-findings-v5-*` indices. `siemlab
validate` enforces numeric ID blocks per project, while 5.x rule IDs are UUIDs and severities
are names. Every project checklist validates with `wazuh-logtest`, and the deploy script
copies files into a rules directory that 5.x no longer has.

**12. Endpoints are half-supported during the move.** 4.x agents can connect to a 5.x manager,
but FIM, SCA, inventory, active response and vulnerability detection are not fully supported
until the agents are upgraded too. Project 4 *is* FIM, so it is blind until both ends are on
5.x.

**13. It doesn't fit on the laptop.** The migration guide says to keep 4.x running until 5.x
is validated. A second server next to the first, plus an endpoint, is roughly 10 GB of the
host's 12 GB, so the parallel run has to be Linux-only.

**14. I'm planning against release-candidate documentation.** Everything in this section comes
from the 5.0.0 documentation branch and the rc1 release notes. Some of it may change before
general availability.

## What I changed today

In [siem-home-lab](https://github.com/exosphere8/siem-home-lab):

- One version, **4.14.8**, recorded in `configs/wazuh-version`. The server guide uses the
  4.14 installer (which installs exactly 4.14.8), and both agent guides install that exact
  version. A test fails if any install guide names a different one.
- The deploy script reads the manager's version. It warns on a 4.x version other than the
  pinned one and refuses anything else, with a clear message on a 5.x manager instead of
  "is this the Wazuh manager?".
- A new [upgrade runbook](https://github.com/exosphere8/siem-home-lab/blob/main/docs/setup-guides/05-upgrading-wazuh.md):
  checkpoint first, un-hold, upgrade in Wazuh's order, check the heap, redeploy, run
  **every** logtest checklist, agents last, rollback by checkpoint. Its 5.x section maps each
  part of the lab to what 5.x does to it.
- The decision is now in the architecture notes: stay on 4.14 until 5.x is generally
  available.

## What I'll check when it happens

- Within 4.x: every `wazuh-logtest` case, with the edge cases first (six root-only SSH
  failures must still trigger 100101), and rule 100103 against the `user`/`dstuser` change.
- For 5.x: whether one event can produce more than one finding, how long a detector interval
  delays a finding, and whether 5.x has any way to express "N events from one source within
  T". The answer to that last one decides whether the stateful rules become 5.x rules or move
  into `siemlab`.

## What I'd take from this

- **An upgrade changes the detection logic, so test it like a change to the rules.** "The
  configuration loads" proves the rules parse, not that they still match.
- **Pin what the rules depend on.** "The current version" means something different every
  month.
- **Read the release notes for the platform's model, not just its features.** "One alert per
  event" and "findings per policy" shape every count I write. They appear in the docs as a
  sentence each.
