# The built-in rule got there first

*From my [SIEM home lab](https://github.com/exosphere8/siem-home-lab). Two of my web rules
never fired on a real manager. A built-in rule I had never read was taking their events.*

---

## The setup

My Nginx rules hang off Wazuh's built-in web rules: 31100 (every access-log line) and 31101
(4xx responses). I had already learned that 31101 claims most probes, because they get a 404,
so my rules were children of both.

## The symptom

On a real Wazuh 4.14.8 manager:

- A request to `/` with the `sqlmap` user agent ended as **31108**, not my scanner rule 100500.
- Ten probes for sensitive paths never reached my "content discovery" rule. Some of them,
  such as `/server-status`, `/.ssh/id_rsa` and `/.env.bak`, ended as **31516**.

## What was actually happening

Wazuh evaluates a rule's children in order and stops at the first match. Two built-in
children of 31100 matched before mine:

- **31108**, level 0, "Ignored URLs (simple queries)", claims any simple 2xx/3xx request.
  A scanner fetching `/` successfully is exactly that. Level 0 means no alert at all, so the
  scanner disappeared.
- **31516**, "Suspicious URL access", claims `.bak`, `/server-status`, `/.ssh` and a few
  others. That's half of my sensitive-path list.

I had written my rules against the two parents I knew about, and I never listed every child
those parents have. The fix was to make my rules children of 31108 and 31516 as well. My
rules are evaluated after the built-in ones, so my children of those rules then get the
event.

## Why static checks could not see it

My validator checks that a parent rule exists. It cannot know which *sibling* will win for a
given log line. Only running the line through the real ruleset shows that, which is what
`wazuh-logtest` is for, and what I had postponed until "the lab is built".

## What I'd take from this

- **Before you extend a built-in rule, read all its children.** Any one of them can take
  your event first, and a level-0 one does it silently.
- **Don't wait for the lab to run the real engine.** The official Docker image gave me a
  real manager in CI in under a minute. I should have used it before writing 33 rules.
