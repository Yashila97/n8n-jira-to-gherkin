# n8n — Jira ticket to draft Gherkin scenarios

An n8n workflow that reads a Jira ticket, sends the parts of it that inform test design
to an AI model, validates the response, and posts draft Gherkin scenarios back to the
ticket as a clearly-labelled comment for a tester to review.

Built to cut the blank-page problem out of test design. It does not write test cases —
it produces a draft to argue with, which is a different and much more achievable thing.

## The workflow

```
Manual trigger
      ▼
Set issue key ──▶ Get Jira issue ──▶ Extract & check ticket
                                              ▼
                                   Enough detail to work with?
                                      │                  │
                                     yes                 no
                                      ▼                  ▼
                            Generate Gherkin      Request more detail
                             (Anthropic API)              ▼
                                      ▼            Ask author for detail
                          Parse & validate output    (Jira comment)
                                      ▼
                            Format Jira comment
                                      ▼
                              Post draft to Jira
```

Import [`workflow/jira-to-gherkin.json`](workflow/jira-to-gherkin.json) into n8n.
Eleven nodes, no custom nodes required.

## Setup

1. **Jira credential** — Jira Software Cloud API in n8n, then re-select it on the three
   Jira nodes. The credential ids in the export are placeholders.
2. **Anthropic credential** — a Header Auth credential with name `x-api-key` and your
   key as the value. Re-select it on the *Generate Gherkin* node.
3. **The system prompt** — Settings → Variables → add `gherkinSystemPrompt`, using the
   text in [`prompts/system-prompt.md`](prompts/system-prompt.md).
4. **Your acceptance-criteria field id** — the *Extract and check ticket* node has
   `CUSTOM_FIELD_ACCEPTANCE_CRITERIA` set to `customfield_10100`. Find yours at
   `/rest/api/3/field` on your Jira instance and change it. If your board keeps
   acceptance criteria in the description, leave it — the node handles both.
5. **Test on one ticket first.** Keep the manual trigger while you tune the prompt. An
   unattended workflow posting draft scenarios onto every new story is a fast way to be
   asked to turn it off.

Node parameter names shift slightly between n8n versions. If a node shows a warning
after import, open it and re-pick the operation — the logic is all in the Code nodes,
which are version-stable.

## The four decisions worth explaining

**A quality gate on the input, not just the output.** The *Enough detail to work with?*
branch counts the words across the description and acceptance criteria and stops if
there are fewer than fifteen. This is the single change that most improved the output.
A one-line ticket does not produce bad scenarios — it produces confident invented
requirements, which are worse, because they look finished. The false branch asks the
author for detail instead, which turns the workflow into a shift-left nudge rather than
a plausible-nonsense generator.

**Only the fields that inform test design are sent.** Summary, description, acceptance
criteria, type, components, labels. Not the reporter, not the comment thread, not
internal links. Partly tokens, mostly that sending a whole Jira issue object to a third
party by default is a habit worth not having.

**Structural validation before a human is asked to look.** The *Parse and validate*
node checks the output is valid Gherkin, contains Given/When/Then, has no `TODO`
placeholders, and invents no requirement IDs. It also counts negative and boundary
signals in the text and flags the comment when the set looks happy-path heavy — which
is the default failure mode of generated test design, and a prompt instruction is not a
guarantee.

**The comment says what it is.** "Draft Gherkin scenarios (AI-generated, unreviewed) —
a starting point for test design, not test cases." Framing is not decoration here. A
comment that reads like finished work gets pasted into a test plan by someone in a
hurry, and that outcome is the one thing this workflow must not cause.

## What it actually does for me

It removes the blank page. Editing a draft set of scenarios is far quicker than writing
from nothing, and the `OPEN QUESTION` scenarios have twice sent a genuine requirement
gap back to a product owner before development started.

**What it does not do:** produce anything usable unedited. The
[worked example](examples/example-output.md) lists the six things a reviewer still had
to add on one ticket — concurrency, jurisdiction differences, what happens to a pending
request when a player self-excludes. None of those were on the ticket, so no prompt
would have produced them. That is the honest limit of the tool.

A tool that saves you two thirds of the time on test design is worth building. A tool
you claim replaces test design will fail in front of you the first time someone asks
about a scenario it invented.

## Contents

| File | What it is |
|---|---|
| [`workflow/jira-to-gherkin.json`](workflow/jira-to-gherkin.json) | The importable workflow |
| [`prompts/system-prompt.md`](prompts/system-prompt.md) | The prompt, why each rule is there, and a tuning log |
| [`examples/example-output.md`](examples/example-output.md) | A worked run: ticket in, scenarios out, and what the reviewer still had to add |

## Author

Naga Yashila Araveti — QA Engineer. AI-assisted test design on regulated payment and
gaming platforms. [LinkedIn](https://www.linkedin.com/in/naga-araveti)
