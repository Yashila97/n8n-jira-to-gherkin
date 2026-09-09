# Example run

A worked example so the workflow can be understood without installing anything.

**Input ticket** (fictional, from a sportsbook backlog):

> **SPORTS-1234** — Story
> **Summary:** Add deposit limit override request
>
> **Description:** Players on the responsible-gambling deposit limit currently cannot
> request an increase. Add a form on the account page allowing a player to request a new
> daily deposit limit. Requests above £500 go to manual review. Requests at or below
> £500 are applied after a 24-hour cooling-off period. Players who have self-excluded
> cannot use the form at all.
>
> **Acceptance criteria:**
> - Player can submit a requested new daily limit
> - Requests over £500 create a manual review task and are not applied automatically
> - Requests of £500 or less are applied 24 hours after submission
> - Self-excluded players see the form disabled with an explanation
> - A player cannot have more than one open request at a time

**Output posted to the ticket as a comment:**

```gherkin
Feature: Deposit limit override request

  Scenario: Player submits a request at or below the automatic threshold
    Given a player with a daily deposit limit of 100 GBP
    When they request a new daily limit of 400 GBP
    Then the request is accepted
    And the new limit is not applied immediately
    And the new limit is applied 24 hours after submission

  Scenario Outline: Threshold for manual review
    Given a player with an active deposit limit
    When they request a new daily limit of <amount> GBP
    Then the outcome is "<outcome>"

    Examples:
      | amount | outcome                |
      | 499    | automatic after 24h    |
      | 500    | automatic after 24h    |
      | 501    | manual review          |
      | 1000   | manual review          |

  @boundary
  Scenario: A request over the threshold is not applied automatically
    Given a player with a daily deposit limit of 100 GBP
    When they request a new daily limit of 501 GBP
    Then a manual review task is created
    And the daily deposit limit remains 100 GBP after 24 hours

  @negative
  Scenario: A self-excluded player cannot request a limit change
    Given a player who has self-excluded
    When they open the deposit limit section of their account
    Then the request form is disabled
    And an explanation is shown

  @negative
  Scenario: A player cannot open a second concurrent request
    Given a player with an open deposit limit request
    When they submit another request
    Then the second request is rejected
    And the reason given refers to the existing open request

  @boundary
  Scenario: A zero limit request
    Given a player with a daily deposit limit of 100 GBP
    When they request a new daily limit of 0 GBP
    Then the request is accepted
    And the new limit is applied 24 hours after submission

  Scenario: OPEN QUESTION: is a decrease also subject to the 24-hour cooling-off period?
```

## What is good about this output

The `Scenario Outline` on the £500 threshold is the right structure and it picks the
correct boundary values — 499, 500, 501 — rather than three arbitrary amounts. It also
separated "a manual review task is created" from "the limit is unchanged after 24
hours", which are two distinct assertions that a single scenario would have conflated.

The final line is the workflow earning its keep. Nothing in the ticket says whether a
player *reducing* their limit also waits 24 hours, and for a responsible-gambling
feature that is a significant gap — an operator making a player wait 24 hours to
*protect* themselves is a regulatory problem, not a UX one. That question went back to
the product owner and changed the acceptance criteria.

## What a reviewer still has to add

Six years of domain knowledge that is not on the ticket:

- What happens to a request that is pending when the player self-excludes mid-window
- Whether the 24 hours is wall-clock or business hours, and which timezone
- Whether the limit change applies to deposits already in flight
- Jurisdiction differences — the cooling-off requirement is not the same in every market
- Whether the manual review task carries the player's deposit history to the reviewer
- Concurrency: two requests submitted in the same second

None of these are on the ticket, so no amount of prompt tuning would produce them. That
is the honest boundary of this tool, and it is the reason the Jira comment says "a
starting point for test design, not test cases".

## Measured effect

Across the tickets I have run it on, the workflow gets me to a reviewed set of scenarios
in roughly a third of the time, mostly by removing the blank-page problem — editing a
draft is much faster than starting from nothing. It has not once produced a set I could
use unedited, and I would be suspicious of anyone claiming otherwise.
