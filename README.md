<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# n8n WAHA Workflow — Setup \& Usage Guide

This README explains how to import, configure, and run the provided n8n workflow that integrates with WAHA (WhatsApp HTTP API) to:

- Always send “Seen” on incoming messages
- Reply “pong” when the message text is “ping” or “Ping”
- Send an image with caption “Check this” when the message text is “image” or “Image”


## What’s in the workflow

Nodes:

- WAHA Trigger: Entry point that receives WhatsApp events/webhooks from WAHA.
- Switch: Routes a single incoming event to multiple downstream flows.
- Send Seen: Marks incoming messages as seen.
- IF body=ping: Checks if payload.body equals “ping” or “Ping”.
- Send "pong": Sends a text “pong”.
- IF body=image: Checks if payload.body equals “image” or “Image”.
- Send Image: Sends an image with caption “Check this”.
- Do nothing / Do nothing1: No-op branches for unmatched conditions.
- Sticky Notes: In-workflow documentation.

Connections:

- WAHA Trigger → Switch
- Switch → Send Seen (always)
- Switch → body=ping
- Switch → body=image
- body=ping (true) → Send "pong"; (false) → Do nothing1
- body=image (true) → Send Image; (false) → Do nothing

Behavior summary:

- Every inbound message triggers “Send Seen”.
- If text is “ping”/“Ping” → responds with “pong”.
- If text is “image”/“Image” → responds with an image + caption “Check this”.

Note: The WAHA nodes reference a credential named “WAHA account”.

***

## Prerequisites

- n8n installed and running (Docker, desktop, or server).
- WAHA (WhatsApp HTTP API) server running and reachable by n8n.
- A publicly reachable URL for n8n webhook endpoints (use a domain with HTTPS or a tunneling tool like Cloudflare Tunnel or ngrok if testing locally).
- WhatsApp device connected to WAHA.

Recommended:

- n8n version supporting:
    - @devlikeapro/n8n-nodes-waha (typeVersion 202409)
    - n8n-nodes-base.if (v2.1)
    - n8n-nodes-base.switch (v3.1)

***

## Step 1 — Install the WAHA node in n8n

- Open n8n → Settings → Community Nodes.
- Search and install “@devlikeapro/n8n-nodes-waha”.
- Restart n8n if prompted.

***

## Step 2 — Create WAHA credentials in n8n

- Go to n8n → Credentials → New Credential.
- Choose the credential type used by the WAHA node (usually “WAHA API” or similar as provided by the community node).
- Fill in:
    - Base URL: the HTTP(S) endpoint of your WAHA server (e.g., https://waha.example.com or http://localhost:3000).
    - API key/token or auth fields as required by your WAHA setup.
- Save it as “WAHA account” (to match the workflow reference), or edit the workflow node later to point to your credential name.

***

## Step 3 — Import the workflow

- In n8n, click Workflows → Import from File.
- Select the provided waha.json and import.
- Confirm the workflow appears with nodes and connections intact.

If credentials show as missing:

- Open the “Send Seen”, “Send "pong"”, and “Send Image” nodes.
- Re-select the WAHA credential created in Step 2 (named “WAHA account” or your chosen name).

***

## Step 4 — Configure WAHA Trigger webhook

The WAHA Trigger node exposes a webhook for incoming events. To use it:

- Open the WAHA Trigger node in n8n.
- Note the webhook URL(s) (test and production) shown by n8n. Ensure n8n is accessible from the internet for production webhooks.
- In your WAHA server, configure the webhook callback to point to the n8n production webhook URL of this trigger.
    - This is typically done in WAHA’s config or via its API/console, setting the “webhookUrl” to the n8n trigger URL.
- Ensure the HTTP method and payload format from WAHA match what the trigger expects (default for the community node).

Tip:

- If testing locally, use a tunnel to expose n8n’s webhook URL to the internet and use the “Production URL” that maps to your tunnel domain.

***

## Step 5 — Configure message content matching

The workflow checks JSON at:

- \$json.payload.body

Ensure WAHA sends incoming message text in payload.body. If your WAHA version uses a different path/key:

- Update the IF nodes (“body=ping”, “body=image”) leftValue fields to the correct JSON path (for example, \$json.message.text or \$json.data.body).
- Case sensitivity: currently true, and checks both lowercase and capitalized variants. Adjust as needed.

***

## Step 6 — Configure Send Image node

The Send Image node needs either:

- A file URL or file data field, or
- A configured default media in the node (depending on the WAHA node’s parameters).

Open “Send Image” and set:

- Image source: Provide a direct URL, binary data reference, or file id supported by your WAHA deployment.
- Caption: Already set to “Check this” (customize as you like).

If the node is currently missing a media reference:

- Add the required parameter (e.g., “url” or “mediaId”) in the node’s fields per the @devlikeapro/n8n-nodes-waha documentation.
- Test with a publicly accessible image URL to simplify initial setup.

***

## Step 7 — Activate the workflow

- Click “Activate” in n8n.
- Send a WhatsApp message to the WAHA-connected number:
    - Any message should be marked as Seen.
    - “ping” or “Ping” should receive “pong”.
    - “image” or “Image” should receive the configured image with caption.

***

## Troubleshooting

- Webhook not triggering:
    - Verify WAHA server can reach n8n’s production webhook URL.
    - Check firewalls, HTTPS certificates, and that the workflow is active.
- payload.body not found:
    - Inspect the incoming execution data in n8n (Execution view) to confirm the path; adjust IF nodes accordingly.
- Send Image fails:
    - Ensure a valid image source is configured in the node.
    - Check WAHA logs for media permission/format issues.
- Authentication errors:
    - Recheck WAHA credential base URL and tokens in n8n credentials.
- Multiple triggers:
    - Ensure only one WAHA webhook is configured for the same event stream or that your WAHA Trigger is set correctly.

***

## Customizations

- Add more commands: Duplicate the IF node pattern to match other keywords and route to different WAHA actions (send document, location, buttons, templates).
- Make matching case-insensitive: Turn off “caseSensitive” and simplify conditions to single strings.
- Rate limit or guard: Insert a Function/Code or IF node to ignore group messages, handle only specific contacts, or throttle replies.
- Observability: Add a Slack/Email/Log node after Switch to record inbound messages.

***

## Security and Deployment Notes

- Protect WAHA and n8n behind HTTPS with proper authentication.
- Do not expose n8n editor publicly; use Basic Auth, VPN, or restricted networks.
- Rotate WAHA API tokens periodically.
- Keep the @devlikeapro/n8n-nodes-waha node updated to the specified version or retest after upgrades.

***

## File reference

- waha.json: The complete n8n workflow including:
    - WAHA Trigger (webhookId: 71251449-2644-4325-8ba7-2a1550a81fb4)
    - Switch with three outputs (Seen, ping, image)
    - IF nodes checking \$json.payload.body for “ping”/“Ping” and “image”/“Image”
    - WAHA actions: Send Seen, Send Text (“pong”), Send Image (caption “Check this”)
    - Credential reference: “WAHA account”

***

## Quick Start Checklist

- Install @devlikeapro/n8n-nodes-waha.
- Create WAHA credential in n8n (“WAHA account”).
- Import waha.json.
- Point WAHA’s webhook to the WAHA Trigger production URL.
- Configure Send Image with a valid media source.
- Activate and test with “ping” and “image”.
<span style="display:none">[^1]</span>

<div style="text-align: center">⁂</div>

[^1]: waha.json

# n8n-templates
