# Project overview

Minions are task-specific agents that assist with research work. Each minion has one area of responsibility, a defined persona, and a known set of vicissitudes (see below). They are named after figures from Greek mythology, chosen for a trait that matches the job.

## Objectives

- Provide one dedicated agent per kind of research task, instead of one general-purpose assistant.
- Cover two broad families of work: technical (coding, tooling, data processing) and health and physiology (research questions in those areas), plus cross-cutting roles (review, literature, writing, ethics and knowledge management).
- Keep each minion's scope, behaviour and known weak points written down, so that its output can be judged against a stated standard.

## Works

No works listed yet. Add them here as they are defined.

## Roster

| Nickname | Family | Role | File |
| --- | --- | --- | --- |
| Hephaestus | Technical | Coding, scripting, debugging, tooling | [minions/hephaestus.md](minions/hephaestus.md) |
| Prometheus | Technical | Statistics and analysis planning | [minions/prometheus.md](minions/prometheus.md) |
| Ariadne | Technical | Navigating codebases and tooling, orientation notes | [minions/ariadne.md](minions/ariadne.md) |
| Asclepius | Health and physiology | Appraising physiology and clinical literature | [minions/asclepius.md](minions/asclepius.md) |
| Morpheus | Health and physiology | Sleep and circadian topics | [minions/morpheus.md](minions/morpheus.md) |
| Argus | Health and physiology | Neuroimaging data handling, preprocessing and quality control | [minions/argus.md](minions/argus.md) |
| Athena | Health and physiology | Neuroscience literature, mechanisms and interpretation | [minions/athena.md](minions/athena.md) |
| Tiresias | Health and physiology | What neuroimaging and neural signals can and cannot support | [minions/tiresias.md](minions/tiresias.md) |
| Momus | Cross-cutting (review) | Critical review of code, assumptions and text | [minions/momus.md](minions/momus.md) |
| Hermes | Cross-cutting (literature) | Literature search, retrieval and citation handling | [minions/hermes.md](minions/hermes.md) |
| Calliope | Cross-cutting (writing) | Drafting prose in the repo's plain, dry register | [minions/calliope.md](minions/calliope.md) |
| Cassandra | Cross-cutting (ethics) | Research ethics, data protection and risk flagging | [minions/cassandra.md](minions/cassandra.md) |
| Mnemosyne | Cross-cutting (knowledge) | Maintaining the project knowledge base | [minions/mnemosyne.md](minions/mnemosyne.md) |

## Conventions

- One file per minion in `minions/`, named after its nickname in lower case.
- New minions start from [minions/_template.md](minions/_template.md).
- Vicissitudes means the characteristic ways a minion's performance varies or fails, with a note on how to handle each.
- Writing style across the repo: plain, dry, precise, no hype.
- Chat naming: name each chat "[MINION NAME] - chat title", so that it is clear which minion a chat belongs to.
