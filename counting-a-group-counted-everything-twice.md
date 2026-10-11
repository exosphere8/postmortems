# Counting a group counted everything twice

*A follow-up from my [SIEM home lab](https://github.com/exosphere8/siem-home-lab), and a
correction to [Wazuh only remembers the last rule](wazuh-only-remembers-the-last-rule.md).
The fix I published there was wrong, and a real Wazuh manager showed me how.*

---

## The setup

In the earlier post I fixed an SSH brute-force rule that never counted failed logins as
`root`. The lesson I wrote down was: *count a category, not an identity*. Both failure rules
carried a shared group, and the counting rule counted the group:

```xml
<rule id="100101" level="10" frequency="6" timeframe="120" ignore="60">
  <if_matched_group>ssh_failed_login</if_matched_group>
  <same_source_ip />
```

The web rules got the same treatment: rule 100504 counted the group `web_sensitive_probe`
and fired on the 10th probe for a sensitive path.

Static validation passed. Every reviewer, me included, nodded.

## What the real manager said

I added a CI job that starts the official `wazuh/wazuh-manager:4.14.8` image, deploys the
rules and sends sample events through them. I also made each test fail if a counting rule
fires *before* the last event, not only if it is missing at the end. The trace for six SSH
failures:

```text
100103 100101 100103 100101 100103 100101
```

The "6 failures" rule fired on the **2nd** failure. The "10 probes" rule fired on the
**4th** probe, and it did the same when I sent the events through the manager's real event
queue instead of the test tool. These were not test artifacts.

## What was actually happening

My rules hang off several parents. 100100 matches *any* built-in failed-login rule
(`if_group authentication_failed`), and 100501 is a child of four built-in web rules. Wazuh
builds a tree, so the same rule appears in it several times, once under each parent.

A rule with `if_matched_group` gets linked at load time to every rule *node* in that group,
and each node pushes the event into the counter's list. One failed login went into the list
several times. Six "failures" were really two.

`if_matched_sid` behaves differently: it keeps one list on the rule itself, however many
times the rule appears in the tree. The fix:

```xml
<rule id="100101" ...><if_matched_sid>100100</if_matched_sid> ...   <!-- non-root -->
<rule id="100107" ...><if_matched_sid>100103</if_matched_sid> ...   <!-- root -->
```

The rule that follows a brute force (100102, "login after a brute force") can still use a
group. It only asks "did *any* match happen?", so duplicates don't matter.

## Why I believed the first fix

It came from a correct observation: Wazuh reports one rule per event, so counting one rule
ID misses events that end on a sibling. What I got wrong was the next step. I reasoned about
the documented behaviour of groups and never watched a count happen. The design looked
right on paper and I treated that as evidence.

## What I'd take from this

- **A counting rule needs a test that counts.** "It fired" is not enough. Check *when* it
  fired: on the 6th event, and not before.
- **Publish fixes with the test that proves them.** My earlier post had none. It said
  "six root-only failures must trigger 100101" and left that as a checklist item for later.
- **When a fix is wrong, correct the record where people read it.** The earlier post now
  links here.
