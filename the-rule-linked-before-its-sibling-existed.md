# The rule linked before its sibling existed

*From my [SIEM home lab](https://github.com/exosphere8/siem-home-lab). A Wazuh rule that
should have counted failed root logins ignored them, and the reason was the order of the
lines in the file.*

---

## The setup

Four SSH rules, in this order in one XML file:

| Order | Rule | Job |
|---|---|---|
| 1 | 100100 | Any failed SSH login |
| 2 | 100101 | 6+ failures from one address (counts the group `ssh_failed_login`) |
| 3 | 100102 | A successful login after that brute force |
| 4 | 100103 | A failed login as `root` (child of 100100, same group) |

The order felt natural: the basic rule, the rules built on it, then the special case.

## The symptom

On a real manager, six failed logins as `root` never produced a brute-force alert. Six
failures with mixed account names did. Same rules, same source address, same count.

## The wrong turns

**"Frequency is off by one."** My own rule file even warned about that. But an off-by-one
would break the mixed case too, and that one worked.

**"The test tool keeps sessions separate in some odd way."** Plausible, but I had already
seen logtest hold state across a session: the mixed test fired correctly.

**"Root events are decoded differently."** The trace said otherwise: every root event ended
as 100103, exactly as designed.

## What was actually happening

Wazuh links `if_matched_group` when it *loads* the rule. When 100101 was read, the only rule
in the group `ssh_failed_login` was 100100. 100103 came three rules later, so it was never
linked, and events ending as 100103 never reached the counter's list. Root-only attacks were
invisible to the brute-force rule. The mixed attack worked because its non-root failures
were counted.

Moving 100103 above 100101 fixed it. That is how I found the bigger problem described in
[Counting a group counted everything twice](counting-a-group-counted-everything-twice.md),
because once linked, the count was wrong in a new way.

## What I'd take from this

- **In a rule file, order is part of the logic.** Rules that count or follow other rules go
  after them. I now say so in a comment at the top of each file.
- **A test that passes in one shape can hide a failure in another.** Mixed accounts passed;
  only the root-only case exposed the bug. Test the case your model says is special.
