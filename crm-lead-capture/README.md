# CRM Lead Capture Automation

## Business Problem

Sales teams often receive website leads but manually copying them into a CRM and notifying the team causes delays, missed follow-ups, and human error.

## Solution

This automation validates incoming leads, stores valid submissions in Airtable, and immediately notifies the sales team in Slack. Invalid submissions are rejected and logged.

## Architecture

```
Website Form
      │
      ▼
Webhook
      │
      ▼
Activepieces
      │
      ▼
Router
├── Invalid Email → Slack Warning → Stop
└── Valid Lead → Airtable → Slack
```

## Tech Stack

- Activepieces
- Airtable
- Slack
- Webhooks
- Postman

## Features

- Email validation
- Automatic CRM entry
- Instant Slack notifications
- Invalid lead rejection

## Folder Structure

- `flow.json`
- `README.md`
- `screenshots/`

## Status

✅ Production-ready learning project
