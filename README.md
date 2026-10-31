# An open-source sales follow-up agent that stops at every write and send

An open-source example project for Kortix: a sales follow-up agent for inbound leads, with the rules that decide what it may do without a person written into the repo beside the agent. Kortix is the open-source CompanyOS: your agents, their skills, your company memory, and every connector in one git repo you own, with the agents working on real cloud computers. Everything here is four files you can read in ten minutes.

The email that goes to a lead is the part everyone audits. The HubSpot record the agent writes back is the part that changes the pipeline, and it is the part a default configuration leaves running. Both stop here, on separate rules, because they are not the same risk.

## What is here

- `kortix.yaml`: three connectors with per-action policies, three project-wide rules, the grant and a cron trigger.
- `agents/sales-follow-up.md`: the persona, ending with what it never does.
- `skills/sales-follow-up-playbook/SKILL.md`: the playbook as a skill.
- `docs/hubspot-sweep.md`: one sweep, and one morning.

Together they are the whole project, which is the point: the agent, the playbook and the permissions are text in a repo you own. For the buyer's version of that argument, [what an open-source AI sales agent puts in your repo](https://aisalesagentsguide.com/open-source-ai-sales-agent) walks the layout, the permissions list and the rule before the send.

## The problem: which calls a person has already decided

Working one lead means reading the record, searching the company, writing a note back, drafting a reply, checking a calendar and sending. A person who approves all six stops reading them by the second week. A person who approves none has handed an agent a mailbox and write access to the CRM, and will learn what it did from a customer.

So the decision moves into the file, made in advance, per class of action, while someone is still paying attention.

The read and the write are not the same risk. A bad search result dies with the session. The note written onto the HubSpot record outlives the run that made it, greets whoever opens the account next quarter, and feeds the forecast. Nobody goes back to check it, because a note looks like a fact.

## The manifest, rule by rule

### The HubSpot connector

```yaml
  - slug: hubspot
    name: HubSpot
    provider: composio
    app: hubspot
    policies:
      # Reads run: the agent has to see the lead before it can work it.
      - match: "get_*"
        action: always_run
      - match: "search_*"
        action: always_run
      # The record write is the agent's real output, and it outlives the
      # session that made it. A person decides each one.
      - match: "write_*"
        action: require_approval
      - match: "update_*"
        action: require_approval
      # Deleting a record is never the agent's call.
      - match: "delete_*"
        action: block
```

On a real lead that is four decisions. `get_contact` and `search_contacts` run, so the agent can see who filled in the form. `update_contact` stops, and the research note lands there: what the company does, the page the lead came from, why now, and the one question the reply should ask. `write_*` stops, so a record created for a company the agent found is a decision and not a side effect. `delete_*` is final, and the agent is told the call was refused, not left to retry it.

The playbook keeps a held write worth approving: four fields, one detail per line, and a line for what could not be found.

### Gmail and the calendar

The Gmail connector, from the same file:

```yaml
      # A draft is a file in the mailbox. Creating one runs.
      - match: "draft_*"
        action: always_run
      - match: "get_*"
        action: always_run
      # Sending is the one action a person owns.
      - match: "send_*"
        action: require_approval
```

A draft runs, because a draft reaches nobody. Sending is the one action a person owns outright. The calendar connector has the same shape: `list_*` and `get_*` run so the agent can offer a real slot, `create_*` waits, because booking time on a colleague's calendar is that colleague's decision.

### Project-wide rules

```yaml
policy:
  default_mode: risk

policies:
  # Rules are evaluated in order and the first match wins, so the exception
  # comes first: a message to your own domain goes out without a hold, which
  # is how you run the flow end to end against a test lead.
  - match: "gmail.send_email"
    action: always_run
    conditions:
      - arg: to
        match: "*@your-company.example"
  # Everything else: no agent in this project mails a lead because a model
  # decided to.
  - match: "*.send_*"
    action: require_approval
  - match: "*.delete_*"
    action: block
```

Kortix evaluates these first, then any policy set bound to the calling agent, then the connector's own rules, and no connector rule can loosen a project rule. Within the list the first match wins, which is why the exception comes first: a message to an address at your own domain runs without a hold, while every other send waits, and a connector added tomorrow, under any name, cannot send or delete without a person.

`default_mode: risk` is the net underneath it all: a call that no rule matches still lets a read run and holds a write. Leave it out and the default is `allow_all`. Approval gates you set. Off until you set them.

### The grant

```yaml
skills:
  sales-follow-up-playbook: skills/sales-follow-up-playbook

agents:
  sales-follow-up:
    prompt_file: agents/sales-follow-up.md
    connectors: [hubspot, gmail, google_calendar]
    secrets: none
    skills: [sales-follow-up-playbook]
    kortix_permissions: none
    workspace: { project_repository: true }
```

The manifest carries no version line: the format has no version. Registration works in two steps. The project registers what exists in a top-level `skills:` map, and each agent enables what it uses with its own grants. A skill folder reaches no agent until an agent's `skills:` grant names it, and an omitted `skills`, `connectors`, `secrets` or `kortix_permissions` grant resolves to none. This agent reaches three connectors because three are listed. `secrets: none` is not decoration. The agent holds no provider credential, because the connector does that work outside the boundary.

Three keys are open by default: `tools` is all, `permission` is allow and `codemode` is on. None of them reaches HubSpot, Gmail or a calendar on its own, which is why the rules above, and not the grant, carry the weight here.

## What a person sees

The agent's turn stops at the held call. In a session it waits under Review, in Needs you, and on the approval card in a Slack or Teams thread when the session started there. A script that calls the same connector gets `ConnectorApprovalPendingError` instead, which carries an `approvalUrl` and an `executionId`.

The page shows what the app will receive, with private values hidden. A project manager, or the person who started the session, reads it and chooses Approve or Deny, with an optional note back to the agent. Approve runs the call once, as the agent that asked, and the agent resumes with the result. Kortix checks the project's rules again at that moment, so a Block rule still wins.

An approval covers one exact call: the same connector, action, arguments and account, so a changed argument needs a new one. There is no approve-everything-for-this-session. A caller whose token is not scoped to the project repeats the identical call within 15 minutes of the decision. With a project-scoped token you repeat nothing, and a repeat while the call still waits or runs answers with the same execution id instead of a second approval. That window is a retry allowance for one call, not a standing permit.

A held call that nobody decides expires at `approvals.ttl`, 24 hours by default and settable from 5 minutes to 30 days.

Every action is recorded. Reading, exporting and streaming the audit log depend on the plan.

## The sweep

A cron trigger fires every 15 minutes with `session_mode: fresh`, so each sweep is a new session with no memory of the one before it. One isolated sandbox per session. Each session has its own isolated machine and branch. Several leads in one sweep are worked as independent units, so a thin result on one never blocks the next. The order of operations, the marker a held write leaves behind, and what a morning looks like are in `docs/hubspot-sweep.md`.

## Run it

```bash
kortix init sales-agent-approval-gate
cd sales-agent-approval-gate
```

Replace `kortix.yaml` with the one here and add the other three files, then run `kortix ship`, which validates `kortix.yaml`, commits, offers to set any missing secret, pushes the branch and connects the connectors it declares. [Read the docs](https://kortix.com/docs/cli) for the rest of the loop.

Then connect your own HubSpot, Gmail and Google Calendar accounts, as whichever identity should own the writes. This repo ships no credential, and it has not been run against a live HubSpot account. Connector credentials are brokered server-side and never enter the machine.

One edit is required before the first sweep. Replace `*@your-company.example` with a pattern for a domain you control. The first end-to-end test sends to a mailbox at that domain, so use one you read.

Then check that the test path fires. The exception is listed first, so a message to your own domain goes out without a hold while every other send waits. Create a test lead with an address at your own domain and watch Review: the send leaves, and the note the agent wrote on that same record is still waiting for you. [Read the docs](https://kortix.com/docs/connect/connectors) for the rest of the semantics.

One note on this manifest as it ships. The three connectors use `provider: composio`, and Kortix can move a Pipedream or Composio connector onto its own app of the same slug: `kortix connectors moves` lists what can move, and `kortix connectors move <slug>` moves one. A move finishes either as `moved` or as `sign_in`, when the new app wants the account connected again, so reconnect Gmail afterwards and check that the send rule still matches the action the connector exposes.

If the machine the agent runs on is yours, [running the agent on your own infrastructure](https://aisalesagentsguide.com/self-hosted-ai-sales-agent) covers what changes: one Compose stack on your host, sessions on sandboxes that call back to the instance's public API URL, and an install with no public address that finishes and still starts no session.

The lead follow-up this repo is a cut of is the one [Kortix](https://kortix.com/use-cases/lead-follow-up) runs itself: a HubSpot sweep every 15 minutes, a fresh session each time, the research notes written onto the record, and the draft held in the email channel for a rep to review, edit and send, with `kortix_followup_drafted` set on the contact so the next sweep skips it. This repo keeps the sweep and puts the draft in Gmail instead, with the send on a rule of its own.

## What is deliberately not here

No send authority at any setting on any connector. No bulk write: the persona forbids writing to more than one record in a single action, so a batch becomes separate decisions a person sees one at a time. No vendor dashboard that mirrors the policies somewhere they can drift from the repo.

The agent also will not guess a title from a name, use anything about a person outside their working life, or write a draft when the record contradicts the research. Where to look next: [Kortix on GitHub](https://github.com/kortix-ai/suna).

## Where the same decision is written out for a buyer

The approvals have a written version that does not start from a manifest. One page takes [what a sales agent may do before a person approves it](https://aisalesagentsguide.com/ai-sales-agent-approvals) and runs the same allow, ask and block decisions over a whole lead rather than one connector. Another takes [what it writes back into the CRM](https://aisalesagentsguide.com/ai-sales-agent-crm-write-back) and treats the record as the risky artefact rather than the email, which is the argument this manifest is built on.

If you are still choosing a tool rather than reading one, [the comparison hub](https://aisalesagentsguide.com/) sets the open-source options beside the managed AI SDRs and sorts them by who approves a send.

## When a managed SDR is the better buy

If you want outbound run for you, with the sequences, sending and reply handling that someone else operates, and no configuration to own or review, buy a managed AI SDR such as Artisan, 11x or AiSDR instead. [What an AI SDR is, in the vendors' own words](https://aisalesagentsguide.com/ai-sdr) draws the category as Salesforce and AiSDR define it, research, write, send and book, and lands where this repo does: the demo and the close stay with a person, and someone still decides what reaches a buyer.

On the two differences that settle it, [Kortix compared with AiSDR on price and the send gate](https://aisalesagentsguide.com/kortix-vs-aisdr) sets $1.50 per person contacted beside the credits plan, and each send rule next to the other: a switch inside a vendor dashboard, or a rule in a file you can read. This repo is for the team that wants the CRM write and the send to be decisions it can read in a diff.

When the shortlist is the work, [eight AI SDR tools compared](https://aisalesagentsguide.com/ai-sdr-tools-compared) sorts the tools by that same question, into a switch in a vendor dashboard, a guard set in a script you own, and a person reading the draft.
