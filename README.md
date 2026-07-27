# AI Lead Triage API

An AI-powered lead pipeline that captures a lead from a website form, scores and tiers it with an LLM, routes it based on that tier, pushes qualified leads into a CRM, logs every lead to a spreadsheet, and alerts the sales team in real time when a lead is hot. Built with n8n.

## Overview

Sales teams waste time reading every inbound inquiry to figure out which ones are worth chasing. This automation does that triage automatically. A prospect submits a web form, and within a few seconds the system:

1. Qualifies the lead with an AI agent (score 1–10, plus a tier and a written reason)
2. Routes hot leads and standard leads down different paths
3. Creates a lead in Zoho CRM, but only for hot, qualified prospects
4. Logs every lead to Google Sheets for reporting
5. Sends an instant email alert to the sales team, but only for hot leads
6. Responds to the form with a structured result the page can display back to the user

The domain used here is reverse-mortgage lending, but the pattern applies to any lead-qualification workflow.

## Architecture

```
   Website form
        |
   POST /triage
        |
   [ Webhook ]
        |
   [ AI Agent (Gemini) ]
   score + tier + reason
        |
   [ If: tier = Hot? ]
      /            \
   Hot             Standard
    |                  |
[ Gmail alert ]   [ Google Sheets ]
    |                  |
[ Zoho CRM ]      [ Respond to Webhook ]
    |
[ Google Sheets ]
    |
[ Respond to Webhook ]
```

## Tech and nodes used

- HTML lead-capture form as the entry point (name, email, message)
- n8n (Cloud) as the automation platform
- Webhook node — exposes the workflow as a POST API endpoint (CORS enabled for browser form submits)
- AI Agent node with Google Gemini as the chat model
- Structured Output Parser — forces the LLM to return clean, typed JSON
- If node — conditional routing based on the AI-assigned tier
- Zoho CRM node — creates a lead for qualified (hot) prospects only
- Google Sheets node — appends every lead as a row for logging and reporting
- Gmail node — sends a formatted HTML alert for hot leads only
- Respond to Webhook node — returns a custom JSON response to the form

## How it works

A prospect fills in the website form. The page sends a POST request to the `/triage` endpoint with a JSON body:

```json
{
  "name": "Melvin",
  "email": "melvin@example.com",
  "message": "I am 65 years old and want to tap into my home equity. My house is worth 8M."
}
```

The AI agent scores the lead against a fixed rubric (product interest, age, home equity, contact detail) and returns a structured result. The If node checks the tier. Hot leads trigger an email alert, get created as a lead in Zoho CRM, and are logged. Standard leads are logged only. Both paths return a response to the form, which displays a confirmation to the user.

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

- **A real web form as the entry point, not a test tool.** The endpoint is fed by an actual HTML form that POSTs JSON, the same way a landing page, ad lead form, or chatbot would in production. CORS is enabled on the Webhook node so the browser submission is accepted, and the form reads the JSON response back to confirm the result to the user.

- **Structured output over free text.** The AI agent is paired with a Structured Output Parser so the model always returns typed fields (`score`, `tier`, `reason`, `summary`) instead of a paragraph. This is what makes the downstream nodes work: you cannot branch on a tier, create a clean CRM record, or write clean spreadsheet columns from an unpredictable block of prose.

- **A scoring rubric in the system prompt.** Without explicit rules an LLM invents its own scale and returns inconsistent scores. The system prompt defines how points are earned and where the tier boundaries sit, so scoring is repeatable and explainable.

- **Only qualified leads reach the CRM.** The Zoho CRM node sits on the hot branch only. Standard and cold leads are logged to the spreadsheet but never enter the CRM, which keeps the sales pipeline clean and focused on prospects worth a rep's time. This is the core value of the routing logic, not every lead is treated the same.

- **Conditional routing with an If node.** Hot leads get an immediate alert plus a CRM record so sales can act while intent is high; standard leads are logged quietly. Each branch has its own Respond node, because a form is waiting on an open connection and every path must return a response or the request times out.

- **Referencing nodes by name in expressions.** Deep in the workflow, `$json` refers to the most recent node, which changes as nodes are inserted. Expressions here reference source nodes explicitly (for example `$('Webhook')` and `$('AI Agent')`) so they keep working even when the flow is rearranged.

## Setup

1. Import the workflow into n8n.
2. Connect a Google Gemini credential (or any supported chat model) to the AI Agent's chat model slot.
3. Connect Google, Gmail, and Zoho CRM credentials for the Sheets, Gmail, and CRM nodes.
4. Create a Google Sheet with the header row: `Date | Name | Email | Score | Tier | Reason | Summary`, and point both Google Sheets nodes at it.
5. Set the Gmail alert recipient.
6. On the Webhook node, add the "Allowed Origins (CORS)" option and set it to `*` so browser form submissions are accepted.
7. Publish the workflow and point the form's `WEBHOOK_URL` at the production webhook URL.

## Demo

Watch a short walkthrough showing a hot lead and a cold lead routed differently in real time, from form submission to CRM record.

[Watch the demo](https://www.loom.com/share/0512def0cc90410fb076928fd714c0bd)

## Possible improvements

- Add an input-validation step that returns a 400 error when required fields are missing, before spending an AI call.
- Replace the If node with a Switch node to route three tiers (Hot to CRM + alert, Warm to a nurture list, Cold to archive).
- Use "Create or Update a lead" in Zoho to avoid duplicate records when the same prospect submits twice.
- Respond to the form first, then log and sync to the CRM asynchronously, to minimize response latency.

## Built by

Melvin Licaros — Data Analyst exploring AI automation and agentic workflows.
