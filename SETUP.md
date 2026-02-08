# AI Analysis Calculator -> Create GHL Contact

## Workflow Overview

n8n workflow that processes incoming webhook requests from the AI Analysis Calculator, sends a confirmation SMS to the contact via GoHighLevel, runs the analysis, and creates a GHL contact.

### Flow

```
Webhook Trigger -> Send Confirmation Text -> AI Analysis Calculator -> Create GHL Contact -> Respond to Webhook
```

## Setup Instructions

### 1. Import the Workflow

1. Open your n8n instance
2. Go to **Workflows** > **Add Workflow** > **Import from File**
3. Select `n8n-workflow-ai-analysis-calculator.json`

### 2. Configure GHL API Credentials

The workflow uses the GoHighLevel Conversations API to send SMS messages and the Contacts API to create contacts.

1. In n8n, go to **Credentials** > **Add Credential** > **Header Auth**
2. Set the header name to `Authorization`
3. Set the header value to `Bearer YOUR_GHL_API_TOKEN`
4. Name it `GHL API Key`
5. Update both the **Send Confirmation Text** and **Create GHL Contact** nodes to use this credential

### 3. GoHighLevel API Token

You need a GoHighLevel Private Integration Token or OAuth token with the following scopes:

- `conversations/message.write` — for sending SMS messages
- `contacts.write` — for creating contacts

Generate a token at: [GoHighLevel Developer Portal](https://marketplace.gohighlevel.com/)

### 4. Webhook Configuration

The webhook listens at path `/webhook/ai-analysis-calculator` and expects a POST request with the following body:

```json
{
  "contactId": "ghl_contact_id",
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com",
  "phone": "+15551234567"
}
```

### 5. Customize the Confirmation Text

Edit the **Send Confirmation Text** node's `jsonBody` parameter to change the SMS message content. The default message is:

> Thank you for submitting your analysis request! We have received your information and are processing your results. You will receive your AI analysis shortly. Reply STOP to unsubscribe.

### 6. Activate the Workflow

After configuration, toggle the workflow to **Active** to begin processing webhook requests.

## SMS Compliance

The confirmation text includes "Reply STOP to unsubscribe" for opt-out compliance. Ensure your GHL account and phone numbers are properly registered for A2P messaging.
