# FHIR data → your AI Primary Care Physician in your pocket.

**An iOS app that connects to Epic via SMART on FHIR, pulls your clinical history, and asks an LLM to act as a Primary Care Physician — returning a plain-English summary and a 10-point personalized care plan.**

Private codebase. Public README.

**Author:** Rajan Gupta · [rajangupta.ai](https://rajangupta.ai)

---

## The idea

Patients can access their own health records through FHIR APIs — mandated by the 21st Century Cures Act. But raw FHIR is a wall of JSON: `MedicationRequest`, `Observation`, `DiagnosticReport`, nested codes, units, reference chains. Useful to systems, unreadable to people.

**fhir-interop-ai** is an iOS app that closes that gap:

1. **Authenticates** the patient to their EHR (Epic Sandbox in development) via **SMART on FHIR** — OAuth2 + PKCE.
2. **Pulls** their clinical data — Patient, Medications, Lab Results, Observations.
3. **Prompts the LLM to act as a Primary Care Physician** (not just a summarizer). System prompt: *"You are a highly skilled Primary Care Physician. Review the following patient details, medications, observations, and lab results and provide (1) a detailed summary and (2) detailed advice to the patient in 10 bullet points for their care."*
4. **Returns two things** to the patient:
   - A plain-English **summary** of their clinical picture.
   - A **10-point personalized care advice list** — what to do next, what to watch, what to ask their real PCP about.
5. **Displays** the AI output alongside the raw FHIR data, so the patient sees both the structured truth and a practical set of next steps.

No PHI leaves the device until the patient explicitly asks for advice.

---

## Why it matters

Two trends collide here:

- **Patient access to FHIR data is now a regulation, not a feature** — Cures Act, CMS-0057, Information Blocking rules.
- **LLMs are finally good enough to read clinical JSON and explain it.**

Most patient-portal experiences still dump FHIR into tables nobody reads. This is a working proof that the whole loop — EHR OAuth → FHIR pull → LLM acting as a PCP → 10-point care plan in the patient's hand — fits in a small app a solo developer can build in a weekend.

---

## Architecture

```
       Patient opens the app
                │
                ▼
         SMART on FHIR
         (OAuth2 + PKCE)
                │
                ▼
           Epic Sandbox
        (fhir.epic.com)
                │
         FHIR resources
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
 Patient   Medication   Observation
                │
                ▼
      PatientDataProcessor
       (structured prompt)
                │
                ▼
            OpenAI
       (gpt-3.5-turbo)
                │
                ▼
       Plain-English summary
        shown to patient
```

---

## What's in the app

| Component | Purpose |
|---|---|
| **OAuthManager** | SMART on FHIR OAuth2 + PKCE flow against `fhir.epic.com` |
| **Patient / Medication / LabResult / Observation** | Swift models for core FHIR resources |
| **PatientDataProcessor** | Turns FHIR JSON into a structured prompt the LLM can reason over |
| **PatientSummaryGenerator** | Builds the LLM request — system + user messages tuned for clinical summarization |
| **OpenAIClient** | Thin wrapper over `POST /v1/chat/completions` |
| **Secrets.plist** (gitignored) | OpenAI API key + Epic client ID — never committed |
| **ContentView** | SwiftUI surface — login, raw data tabs, summary tab |

---

## Security posture

This app touches real EHR endpoints and sends data to a third-party LLM. The security boundaries matter:

- **OAuth2 + PKCE** for Epic — no static secret on the device
- **Access token** held in `@Published` state only for the session (not persisted)
- **Secrets** (OpenAI key, Epic client ID) loaded from `Secrets.plist` at runtime — **gitignored**
- **Sandbox-only by design** — the current build targets Epic's public sandbox with synthetic patients; going to production requires Epic App Orchard approval and real privacy/consent flows
- **No PHI persistence** — the app holds FHIR data in memory, not in Core Data or SQLite

---

## Stack

Swift · SwiftUI · OAuthSwift · Epic SMART-on-FHIR Sandbox · OpenAI Chat Completions (`gpt-3.5-turbo`) · iOS 17+

---

## Status

Working prototype against Epic Sandbox with synthetic patients. Functional end-to-end: login → FHIR fetch → LLM summary.

**Not** production-ready yet. Path to production:

- Epic App Orchard approval
- Consent UI + audit logging
- Real privacy review
- Swap the LLM to a **HIPAA-compliant / BAA-covered** provider (see below)

---

## HIPAA-compliant LLM options

PHI can only go to an LLM provider that will sign a **Business Associate Agreement (BAA)**. OpenAI's default API is *not* covered; the swap is straightforward because the prompt layer stays the same.

| Provider | BAA available | Notes |
|---|---|---|
| **Azure OpenAI Service** | ✅ Yes | Same GPT-4 / GPT-3.5 models, Microsoft signs BAA — the easiest lift from the current code |
| **AWS Bedrock (Anthropic Claude)** | ✅ Yes | Claude Sonnet / Opus under AWS BAA |
| **Google Vertex AI (Gemini / Claude)** | ✅ Yes | Google BAA covers Vertex-hosted models |
| **OpenAI Enterprise / Zero Data Retention** | ✅ Yes | Direct BAA on Enterprise tier, not on pay-as-you-go |
| **On-device** (Llama / Mistral via MLX / llama.cpp) | N/A — PHI never leaves device | Hardest technically, strongest privacy story |
| **Purpose-built clinical LLMs** (Hippocratic AI, Abridge, Ambience) | ✅ Yes | Trained on clinical data; BAA standard in the sector |

**My default for v2:** Azure OpenAI — minimal code change (just endpoint + auth header) and enterprise-grade compliance posture out of the box.

---

## Code access

The code is private to protect the work in progress and the integration keys. If you're evaluating me for a role and want to see the implementation, **contact me** and I'll grant 14-day read access.
