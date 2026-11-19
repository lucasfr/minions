# 🏛️ Minions

**Task-specific research agents, each with a defined role, persona and known failure modes, named after figures from Greek mythology.**

---

## 📖 About

Minions are agents that assist with research work. Instead of one general-purpose assistant, each minion has a single area of responsibility, a persona that sets how it works and communicates, and a written list of vicissitudes: the characteristic ways its performance varies or fails, with a note on how to handle each.

The names are chosen for a trait that matches the job. Hephaestus builds, and is slow about it. Momus finds fault. Cassandra is always right and never believed.

The project overview, objectives and conventions are in [PROJECT.md](PROJECT.md).

## 🧭 The Minions

| Minion | Family | Role |
| --- | --- | --- |
| [Hephaestus](minions/hephaestus.md) | 🛠️ Technical | Coding, scripting, debugging, tooling |
| [Prometheus](minions/prometheus.md) | 🛠️ Technical | Statistics and analysis planning |
| [Ariadne](minions/ariadne.md) | 🛠️ Technical | Navigating codebases and tooling, orientation notes |
| [Asclepius](minions/asclepius.md) | 🩺 Health and physiology | Appraising physiology and clinical literature |
| [Morpheus](minions/morpheus.md) | 🩺 Health and physiology | Sleep and circadian topics |
| [Argus](minions/argus.md) | 🩺 Health and physiology | Neuroimaging data handling, preprocessing and quality control |
| [Athena](minions/athena.md) | 🩺 Health and physiology | Neuroscience literature, mechanisms and interpretation |
| [Tiresias](minions/tiresias.md) | 🩺 Health and physiology | What neuroimaging and neural signals can and cannot support |
| [Momus](minions/momus.md) | 🧭 Cross-cutting | Critical review of code, assumptions and text |
| [Hermes](minions/hermes.md) | 🧭 Cross-cutting | Literature search, retrieval and citation handling |
| [Calliope](minions/calliope.md) | 🧭 Cross-cutting | Drafting prose in the repo's plain, dry register |
| [Cassandra](minions/cassandra.md) | 🧭 Cross-cutting | Research ethics, data protection and risk flagging |
| [Mnemosyne](minions/mnemosyne.md) | 🧭 Cross-cutting | Maintaining the project knowledge base |

## 🗂️ Repository Structure

```
minions/
├── README.md
├── PROJECT.md
└── minions/
    ├── _template.md
    ├── hephaestus.md
    ├── prometheus.md
    ├── ariadne.md
    ├── asclepius.md
    ├── morpheus.md
    ├── argus.md
    ├── athena.md
    ├── tiresias.md
    ├── momus.md
    ├── hermes.md
    ├── calliope.md
    ├── cassandra.md
    └── mnemosyne.md
```

## 🧩 How a Minion Is Defined

Each minion has one file in `minions/`, named after its nickname in lower case. Every file has the same sections:

| Section | Content |
| --- | --- |
| Header | Nickname, family, and a one-sentence role |
| Persona | Temperament, working style, and how it communicates |
| Responsibilities | What the minion does |
| Out of scope | What it does not do, and which minion handles it instead |
| Vicissitudes | A table of vicissitude, myth, how it shows, and how to handle it |
| Handoffs | What it receives, what it returns, and which minions it works with |

New minions start from [minions/_template.md](minions/_template.md).

## 🏷️ Chat Naming

Name each chat "[MINION NAME] - chat title", so that it is clear which minion a chat belongs to.

## 👥 Authors

| Role | Name |
| --- | --- |
| Author | Lucas França, Assistant Professor in Computer and Information Sciences, Northumbria University |
