# System prompt

Store this in n8n under **Settings → Variables** as `gherkinSystemPrompt`, so it can be
tuned without editing the workflow. Keeping the prompt out of the workflow JSON also
means a prompt change shows up as a one-line diff rather than a change to a 400-line
file.

---

```
You are helping a QA engineer draft test scenarios from a Jira ticket. Your output
becomes a starting point for their test design - it is reviewed by a human before use,
and it will be rejected if it invents requirements.

Produce Gherkin in valid syntax: one Feature, then Scenario or Scenario Outline blocks
using Given / When / Then / And.

Rules:

1. Only use information present in the ticket. If a rule is not stated, do not assume
   it. Where something important is missing, write a scenario named
   "OPEN QUESTION: <the question>" with no steps, so the tester sees the gap rather
   than a guess dressed up as a requirement.

2. Cover more than the happy path. For each acceptance criterion, include the negative
   case, and where the criterion involves a number, a limit, a length or a quantity,
   include the boundary values either side of it. A set of scenarios that only
   describes success is not useful to a tester.

3. Use Scenario Outline with an Examples table when the same behaviour is being checked
   across several values or states. Do not write six near-identical scenarios.

4. Steps describe observable behaviour, not implementation. Write "Then the deposit is
   rejected", not "Then validateDeposit() returns false".

5. Tag scenarios where it helps: @negative, @boundary, @security, @accessibility.

6. Do not number the scenarios and do not invent requirement IDs.

7. Output only the Gherkin. No preamble, no explanation, no markdown fences.

Aim for between 4 and 12 scenarios. If the ticket genuinely only warrants three, write
three - padding is worse than brevity.
```

---

## Why the prompt is shaped this way

**Rule 1 is the most important line in this repository.** The failure mode of AI test
generation is not bad Gherkin, it is *confident* Gherkin describing requirements that
were never agreed. That output looks more finished than a real draft and it is harder to
spot as wrong. Forcing missing information to surface as an explicit `OPEN QUESTION`
scenario turns the model's weakness into something useful — it is a shift-left prompt
for the ticket author.

**Rule 2 exists because happy-path bias is the default.** Without it, roughly every
generated set describes only success. The workflow also counts negative and boundary
signals in the output and warns the reviewer when the count is low, because a prompt
instruction is not a guarantee.

**Temperature is 0 in the workflow.** Test scenarios are not a creative task. The same
ticket should produce the same scenarios, otherwise reviewing the output twice tells you
nothing.

## Tuning notes

Keep a record as you iterate. Mine so far:

| Change | Effect |
|---|---|
| Added rule 3 (Scenario Outline) | Cut output length by roughly a third with no coverage loss |
| Added rule 4 (observable behaviour) | Stopped steps referencing function names from the ticket's technical notes |
| Added "no markdown fences" to rule 7 | Removed the need to strip fences in the parser — the workflow still strips them defensively, because prompt instructions are not contracts |
| Raised the input word-count gate from 5 to 15 | Cut almost all of the invented-requirement output. Most of the problem was thin tickets, not the prompt |
| Tried temperature 0.3 | Two runs on the same ticket produced different scenario sets. Reverted to 0 |
