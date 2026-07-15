---
title: "Projects"
permalink: /projects/
---



## AI Document Research Agent

[GitHub Repository](https://github.com/bobstrickland/AIDocumentResearchAgent)

This project explores what it takes to build a genuinely *autonomous* AI agent — one that decides its own tool-use path in pursuit of a goal, rather than following a hard-coded workflow. Given a research question, the agent should be able to:

- Search a private document collection via retrieval-augmented generation (RAG)
- Decide whether it has enough information, or needs to search again with refined queries
- Synthesize an answer, citing which documents it drew from
- Save its findings for later retrieval
- Support natural follow-up questions within a session

The project is also a deliberate exercise in modern backend architecture — Spring Boot, PostgreSQL, vector search, containerization, and CI/CD — applied to a genuinely current problem domain (AI agent design) rather than a toy example.

## Japa — Android Mantra Counter

[GitHub Repository](https://github.com/bobstrickland/Japa)

A mantra-counting app for Android, built to solve a real problem that 
existing apps handle poorly: most japa counters require the screen to 
stay on and the user to tap the display for every repetition. For a 
practice that can run 1.5–2 hours a day, that's impractical.

**The key design decision:** counting happens through the device's volume 
buttons, so the screen can stay off for the entire session. Getting this 
right turned out to be the hardest engineering problem in the project — 
capturing hardware key events while the screen is off requires a 
foreground service running on a background thread, and Android's process 
lifecycle management (Doze mode, OEM battery optimization) actively works 
against you. Getting the threading model right so the service doesn't get 
interrupted mid-session took real trial and error.

**Other features:**
- Customizable mantras, repetitions per round, and rounds per day
- Audio chanting support — triggered by volume button or set to repeat 
  continuously, with adjustable playback speed
- Dual-script text display: Roman transliteration and Devanagari
- Additional prayers screen for reference
- Automated build pipeline via GitHub Actions

**Status:** Alpha testing

---

## GPU Covert Channel Security Research

[GitHub Repository](https://github.com/bobstrickland/KMDFCudaLogger)

Graduate-level security research (2015) demonstrating a novel attack 
vector: malware that hides its activity entirely within a GPU, out of 
reach of CPU-bound security software.

**The attack:** A Windows kernel mode driver written in C exploits 
Direct Memory Access (DMA) to locate and monitor the system keystroke 
buffer without CPU involvement, rendering it invisible to contemporary 
anti-malware software. Captured keystrokes are encrypted and exfiltrated 
to a remote system entirely from GPU memory — the CPU never touches the 
data after initial capture.

**The defense:** A companion detection tool leverages the PCI bus to 
inspect video card memory from a CPU-bound process, exposing GPU-resident 
malware activity. Performance characteristics of the detection approach 
were analyzed and mitigation strategies identified.

This project involved kernel mode driver development (KMDF), CUDA GPU 
programming, DMA exploitation, and PCI bus inspection — a stack most 
software developers never touch. The research demonstrated both the 
attack vector and a viable detection countermeasure.

The full project report and presentation are available in the repository.

---

## Personal AWS Project

*In active development — details coming soon*

A personal project I'm using to build hands-on experience with the 
cloud-native patterns I want to bring into my professional work. Current 
stack includes:

- **AWS** — Lambda, S3, CloudFormation, serverless Valkey for caching
- **Docker** — containerized PostgreSQL with persistent volume storage
- **AI-assisted development** — using Claude for scaffolding, 
  troubleshooting, and design tradeoffs in a real engineering context

This is where I'm closing the gap between the enterprise Java background 
I've built over 25 years and the modern distributed, cloud-native 
architecture the industry has moved toward.

---

## IBM OnDemand Migration

*Federal system — code not publicly available*

One of the more challenging projects of my career, worth describing even 
without a code link.

At Carelon Federal Services, document storage for a federal platform 
serving 3M+ users was running on IBM OnDemand, with annual licensing 
costs in the millions of dollars. I volunteered to lead the migration to 
an in-house Vault system — a project nobody else wanted to own, because 
once committed there was no rollback path. A mistake meant permanent loss 
of critical federal records.

I designed the migration approach, built the integration layer in Java 
using REST calls to the Vault API, validated data integrity at each stage, 
and executed the migration without losing a single record. The result: 
several million dollars in annual licensing costs eliminated.

The technical work was a plain Java library consumed by both legacy Struts 
applications and a Spring Batch application — nothing exotic, but the 
stakes made it memorable.
