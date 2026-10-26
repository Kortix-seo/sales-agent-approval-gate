# The HubSpot sweep, start to finish

A form comes in Tuesday morning. Fifteen minutes later an agent has read the record, researched the company, written a note back, drafted a reply and picked a slot on the calendar. None of it reaches the lead until somebody decides. The sweep runs inside an open-source Kortix project you can read in full.

## What fires the sweep

```yaml
triggers:
  - slug: hubspot-sweep
    name: HubSpot new-lead sweep
    type: cron
    agent: sales-follow-up
    cron: "0 */15 * * * *"
    session_mode: fresh
    prompt: |
      Sweep HubSpot for leads created or updated since the last marker, work
      each one as an independent unit, and stop before every write and send.
```

The cron fires at the top of every fifteen minutes. `session_mode: fresh` shapes the rest: each firing is a new session on its own sandbox, with no transcript and no machine state from the one before it. A fresh session starts from the same base every time. The only thing that carries forward is what the agent wrote into HubSpot, and a held write is a state a person has seen. The prompt is short because the work lives in `agents/sales-follow-up.md` and the `sales-follow-up-playbook` skill.

## The marker is the note

The prompt asks for leads created or updated since the last marker. In this project the marker is the note the agent writes onto the record. Step three of the persona is that write, and the HubSpot connector holds it, so it lands only once a person approves it. An approved note updates the record, and the next sweep sees a lead already worked.

A note nobody approves does not land, so fifteen minutes later the sweep reads the same record, sees nothing new, and works the lead again. A lead that needs a decision keeps arriving in front of you. A denied note does not land either. The agent reads your message, changes the note or the draft, and stops. It does not re-propose the same action with the same arguments, or split a denial into smaller calls. A held note that expires after 24 hours behaves the same way.

The queue in Review is also the list of leads the next sweep will return to. Nothing is dropped without a decision, and nothing is remembered without one.

## One lead, step by step

The persona works a lead in five moves. Read the record for the company, the name, how they arrived and what the form said. Research the company from public sources. Write the note back to the record, which is held. Draft the reply, which is not held because a draft reaches nobody. Then leave both in place and state which approvals it is waiting on.

The playbook fills in what good looks like. The note holds four fields, one line each: what they do, why now with the signal quoted and linked, which offering this looks like, and the open question the reply should ask. The reply runs under 120 words, names the thing they asked about in its first line, asks that one question, and offers a real calendar slot rather than a booking link. With no slot free in the next three working days, it offers nothing and says so.

Three calls in that sequence are held: the HubSpot note, the send, and any calendar event. The delete is blocked outright.

The skill names four leads it will not draft for at all: an existing customer or anyone with an open support thread; a form that mentions security review, procurement, legal or a data processing agreement; a competitor, recruiter or press; and a record whose contents contradict the research. Those records never enter the approval queue, so the queue stays short enough to read.

## The morning

Review, under Needs you. Sends come first. The reply offers a slot the agent found free when the sweep ran, at least two hours ahead of that moment, and a draft that has waited overnight may name a time that has already gone. Check the slot against the calendar before you approve, not after. Bookings come second for that reason, and the record notes last: the note is the account of why the reply says what it says, and your colleagues read it next quarter.

Three decisions are available on each item. Approve runs it once, and an identical repeat of the same call within fifteen minutes runs once more. Deny carries your message back into the session, and the agent reads it and changes its approach. Doing nothing is the third and a real option: the call expires after 24 hours, the card reads Expired, the session does not resume, and the lead comes back.

## What the sweep does not do

It does not send, at any setting, on any connector. Deleting a record is blocked outright. It never writes to more than one record in a single action, so twenty leads are twenty decisions, not one approval covering a spreadsheet. No memory carries across firings. And the daily summary, that paragraph with leads worked, approvals waiting and escalations with their reasons, goes in the session output and stays there.

## The one gap it cannot close

A lead that arrived before you enabled the trigger is the case this flow does not repair. There is no backfill step: whatever the first firing covers is where the work starts, and older records are not guaranteed to appear. Work the backlog by hand once, before you turn the trigger on, so you know which leads the agent has seen and which it has not.
