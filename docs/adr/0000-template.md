---
status: "{proposed | accepted | rejected | deprecated | superseded by ADR-NNNN}"
date: YYYY-MM-DD
decision-makers: "{who decided}"
consulted: "{optional}"
informed: "{optional}"
---

# {Short title: the problem and the chosen solution}

Template: [MADR 4](https://adr.github.io/madr/). Copy this file to `NNNN-short-title.md`, use the next free number,
and never renumber or delete an ADR. To change a decision, write a new ADR that supersedes the old one and update the
old one's `status`.

## Context and problem statement

{Two to four sentences: the forces at play and the question being decided. Link the design documents and threat ids
(`L-xx`) it touches.}

## Decision drivers

* {driver 1, e.g. a security requirement or threat}
* {driver 2, e.g. per-OS parity}
* {…}

## Considered options

* {option 1}
* {option 2}
* {…}

## Decision outcome

Chosen option: "{option}", because {justification tied to the drivers}.

### Consequences

* Good, because {positive consequence}.
* Bad, because {negative consequence, and how it is mitigated}.

### Confirmation

{How compliance is checked: a test id, a CI check, a spike id with its kill criterion, or a review step.}

## Pros and cons of the options

### {Option 1}

* Good, because {…}
* Bad, because {…}

### {Option 2}

* Good, because {…}
* Bad, because {…}

## More information

{Links to design documents, spikes, public references, and when the decision should be revisited.}
