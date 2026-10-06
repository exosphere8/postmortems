# Wazuh only remembers the last rule

*A post-mortem from my [SIEM home lab](https://github.com/exosphere8/siem-home-lab), in
which a brute-force rule was written so that it could never count attacks on the account
attackers try most.*

---

## The setup

Project 2 of the lab detects SSH password guessing. The rules chain from signal to story:

| Rule | Level | Job |
|---|---|---|
| 100100 | 5 | Re-label every failed sshd login with an ATT&CK mapping |
| 100103 | 8 | A failed login as `root`, as a child of 100100 |
| 100101 | 10 | 6+ failures from one source within 2 minutes |
| 100102 | 13 | A successful login from that source afterwards: the guessing worked |

100101 is a frequency rule, and I pointed it at the building block:

```xml
<rule id="100101" level="10" frequency="6" timeframe="120" ignore="60">
  <if_matched_sid>100100</if_matched_sid>
  <same_source_ip />
```

100103 hangs off 100100 with `<if_sid>100100</if_sid>` and `<user>^root$</user>`.

## The assumption

I read "100103 is a child of 100100" the way I would read a class hierarchy: a failed root
login *is a* failed login, so anything that counts 100100 counts it too.

That is how the rule tree is drawn, and it is how the rules match. An event does go through
100100 on its way to 100103. It's just not how Wazuh *remembers* events.

## What was actually happening

Wazuh produces one alert per event: the deepest rule that matched. A failed root login
walks the tree (built-in sshd rule, then 100100, then 100103) and is reported as **100103**.
The frequency rule looks back through events by the rule they finally ended as. Root
failures never end as 100100, so 100100's history never contains them.

Follow that through for an attacker who only tries `root`, which is the most common SSH
brute force there is:

- every attempt raises a level-8 "failed login as root";
- 100101 never reaches six, because it is counting a list that stays empty;
- 100102, the alert that says *the guessing worked*, depends on 100101 and can never fire.

The more specific I made the detection for the most dangerous account, the less the
aggregate saw of it.

This was caught in review, before the lab is deployed, so it has not been observed on a
live manager yet. The Project 2 checklist now contains the case that proves it either way:
six root-only failures in `wazuh-logtest` must trigger 100101.

## The fix

Count a category, not an identity. Both rules now carry a shared group, and the aggregate
counts the group:

```xml
<group>authentication_failed,ssh_failed_login,</group>   <!-- on 100100 and 100103 -->

<rule id="100101" level="10" frequency="6" timeframe="120" ignore="60">
  <if_matched_group>ssh_failed_login</if_matched_group>
  <same_source_ip />
```

Whichever rule an event ends as, it still counts as a failed login.

## The same bug, twice more

Once I knew the pattern, it showed up in other projects too.

**Web scanners hid their own content discovery.** 100501 flags requests for sensitive paths
(`/.env`, `/.git/`), and 100504 fires when one address makes ten of them in a minute.
100500 flags scanner user agents. A scanner running with its default user agent and probing
`/.env` ends as 100500, not 100501, so 100504 never fired for exactly the tools that do
content discovery. The fix is a child rule for "scanner probing a sensitive path", with
100501 and that child sharing a `web_sensitive_probe` group that 100504 counts.

**A generic Suricata rule hid the specific ones.** 100600 raises any severity-1 Suricata
alert to level 12. The lab's own signatures (SSH connection burst, scanner user agent) have
severity-1 classtypes, so 100600 claimed them first and their specific rules, which carry
the ATT&CK mapping, were never reached. Now the specific rules are children of 100600 as
well, so they refine it instead of competing with it.

## Two other things I had wrong about the environment

**Real-time file monitoring does not work on single files.** I configured `realtime="yes"`
on `/etc/passwd`, `/etc/shadow` and `/etc/sudoers`. Wazuh's real-time mode uses inotify on
*directories*, so those entries would quietly fall back to the scheduled scan, which runs
every 12 hours in this lab. The single files now use `whodata` (auditd), which works on files
and also records which user and process made the change. That answers the first question of
any investigation for free.

**NAT is not isolation.** The network guide told the reader to verify that lab VMs *cannot*
reach the home LAN. Hyper-V NAT only blocks the other direction: nothing on the LAN can
open a connection into the lab, but the VMs can reach the LAN through the host. The guide
now says exactly that, and every validation step targets lab addresses only.

## What I'd take from this

- **When a system reports one answer per event, every count is a count of answers.** If a
  more specific rule can win, count a group that both rules share, never a single rule ID.
- **Write the validation case for the edge that breaks your model, not the one that
  confirms it.** My original logtest step repeated a non-root failure six times, which is
  the one case where the bug cannot appear.
- **"It's configured" is not "it's monitored".** Both the inotify fallback and the NAT
  direction were cases of a setting being accepted without doing what I meant.
