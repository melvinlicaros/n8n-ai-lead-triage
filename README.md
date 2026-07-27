# AI Lead Triage API

An AI-powered webhook API that receives an incoming lead, scores and tiers it with an LLM, routes it based on that tier, logs every lead to a spreadsheet, and alerts the sales team in real time when a lead is hot. Built with n8n.

## Overview

Sales teams waste time reading every inbound inquiry to figure out which ones are worth chasing. This automation does that triage automatically. A lead is submitted to a single API endpoint, and within a few seconds the system:

1. Qualifies the lead with an AI agent (score 1–10, plus a tier and a written reason)
2. Routes hot leads and standard leads down different paths
3. Logs every lead to Google Sheets for reporting
4. Sends an instant email alert to the sales team, but only for hot leads
5. Responds to the caller with a structured JSON result

The domain used here is reverse-mortgage lending, but the pattern applies to any lead-qualification workflow.

## Architecture

```
                         POST /triage
                              |
                         [ Webhook ]
                              |
                     [ AI Agent (Gemini) ]
                     score + tier + reason
                              |
                        [ If: tier = Hot? ]
                        /                \
                     true                false
                       |                    |
              [ Gmail alert ]        [ Google Sheets ]
                       |                    |
             [ Google Sheets ]      [ Respond to Webhook ]
                       |
             [ Respond to Webhook ]
```

## Tech and nodes used

- n8n (Cloud) as the automation platform
- Webhook node — exposes the workflow as a POST API endpoint
- AI Agent node with Google Gemini as the chat model
- Structured Output Parser — forces the LLM to return clean, typed JSON
- If node — conditional routing based on the AI-assigned tier
- Google Sheets node — appends every lead as a row for logging and reporting
- Gmail node — sends a formatted HTML alert for hot leads only
- Respond to Webhook node — returns a custom JSON response to the caller

## How it works

The caller sends a POST request to the `/triage` endpoint with a JSON body describing the lead:

```json
{
  "name": "Melvin",
  "email": "melvin@example.com",
  "message": "I am 65 and want to tap my home equity. House is worth 8M.",
  "propertyValue": 8000000
}
```

The AI agent scores the lead against a fixed rubric (product interest, age, home equity, contact detail) and returns a structured result. The If node checks the tier. Hot leads trigger an email alert and are logged; standard leads are logged only. Both paths return a response to the caller.

Hot lead response:

```json
{
  "received": true,
  "tier": "Hot",
  "score": 8,
  "message": "A specialist will contact you shortly."
}
```

Standard lead response:

```json
{
  "received": true,
  "tier": "Cold",
  "score": 2,
  "message": "Thank you. Our team will review your inquiry."
}
```

## Design decisions

The parts that make this reliable rather than just a demo:

- **Structured output over free text.** The AI agent is paired with a Structured Output Parser so the model always returns typed fields (`score`, `tier`, `reason`, `summary`) instead of a paragraph. This is what makes the downstream nodes work: you cannot branch on a tier or write clean spreadsheet columns from an unpredictable block of prose.

- **A scoring rubric in the system prompt.** Without explicit rules an LLM invents its own scale and returns inconsistent scores. The system prompt defines how points are earned and where the tier boundaries sit, so scoring is repeatable and explainable.

- **Conditional routing with an If node.** Hot and standard leads deserve different handling. Hot leads get an immediate alert so sales can act while intent is high; standard leads are logged quietly. Each branch has its own Respond node, because a webhook caller is waiting on an open connection and every path must return a response or the request times out.

- **Referencing nodes by name in expressions.** Deep in the workflow, `$json` refers to the most recent node, which changes as nodes are inserted. Expressions here reference source nodes explicitly (for example `$('Webhook')` and `$('AI Agent')`) so they keep working even when the flow is rearranged.

## Setup

1. Import the workflow into n8n.
2. Connect a Google Gemini credential (or any supported chat model) to the AI Agent's chat model slot.
3. Connect Google and Gmail credentials for the Sheets and Gmail nodes.
4. Create a Google Sheet with the header row: `Date | Name | Email | Score | Tier | Reason | Summary`, and point both Google Sheets nodes at it.
5. Set the Gmail alert recipient.
6. Activate the workflow and send a POST request to the production webhook URL.

## Possible improvements

- Add an input-validation step that returns a 400 error when required fields are missing, before spending an AI call.
- Replace the If node with a Switch node to route three tiers (Hot to alert, Warm to a nurture list, Cold to archive).
- Respond to the caller first, then log and alert asynchronously, to minimize response latency.

## Demo
Watch a 55-second walkthrough showing a hot lead and a cold lead routed differently in real time.

[Watch the 55-second demo](https://www.loom.com/share/0512def0cc90410fb076928fd714c0bd)

## Built by

Melvin Licaros — Data Analyst exploring AI automation and agentic workflows.
