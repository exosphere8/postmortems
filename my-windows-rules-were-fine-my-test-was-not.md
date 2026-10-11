# My Windows rules were fine; my test was not

*From my [SIEM home lab](https://github.com/exosphere8/siem-home-lab). Seven Windows tests
failed on a real Wazuh manager. Not one of the rules was wrong.*

---

## The setup

My CI tests send sample events to a real Wazuh 4.14.8 manager through its logtest socket and
check which rule each event ends on. For Windows I wrote events the way they look in Wazuh
*alerts*: JSON with `win.system.eventID`, `win.eventdata.targetUserName` and so on. I used
`log_format: eventchannel`.

## The symptom

All seven Windows cases failed. A failed logon (event 4625) ended as rule 1002, "Unknown
problem somewhere in the system". The others matched nothing at all.

## The wrong turns

**"My rules are wrong."** That was my first assumption, and I nearly started rewriting
them. But the Windows rules had never even been reached: rule 1002 is a generic syslog rule.

**"Wrong log format name."** I checked the decoders shipped with the ruleset. There is no
XML decoder named `windows_eventchannel` at all, and yet the built-in Windows rules use
`decoded_as windows_eventchannel`.

## What was actually happening

Two things, both in the Wazuh source code rather than the docs:

1. The Event Channel decoder is built into the analysis daemon and only runs for events that
   arrive on the Windows event queue. **Logtest always feeds input through the normal log
   decoders**, so no Windows event can ever reach the Windows rules through logtest.
2. The agent doesn't send the decoded JSON I had copied from alerts. It sends
   `{"Message": ..., "Event": "<Event xmlns=...>...</Event>"}`, the raw event XML, and the
   manager decodes that XML into `win.system` and `win.eventdata`.

So my test runner now sends Windows cases to the manager's analysis queue, in the agent's
format, and reads the alert back from `alerts.json`. All seven Windows cases passed at the
first try. The rules were right all along.

## What I'd take from this

- **When a test fails, ask what it proves before you fix the code.** A test that can't reach
  the code under test says nothing about the code.
- **Copy test inputs from where the data enters, not from where it comes out.** An alert is
  the system's *output*, already decoded. Feeding it back in skipped the very step I needed
  to test.
