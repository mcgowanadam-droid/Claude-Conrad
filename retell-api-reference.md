# Retell AI API Reference — Local Reference for Claude Code

> **IMPORTANT**: Do NOT attempt to fetch Retell AI documentation from the web. All required API schemas are in this file. The Retell docs site blocks automated fetchers (403 errors).

## Authentication

All API calls use Bearer token authentication.

```
Authorization: Bearer YOUR_RETELL_API_KEY
Content-Type: application/json
```

Base URL: `https://api.retellai.com`

---

## API Workflow Overview

Building a Retell voice agent requires two steps:

1. **Create a Conversation Flow** → returns `conversation_flow_id`
2. **Create a Voice Agent** → attaches the conversation flow via `response_engine`

---

## 1. Create Conversation Flow

**`POST /create-conversation-flow`**

This creates the conversation logic — nodes, edges, tools, and global prompt.

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `model_choice` | object | `{ "type": "cascading", "model": "gpt-4.1" }` |
| `start_speaker` | string | `"agent"` or `"user"` |
| `nodes` | array | Array of node objects (see Node Types below) |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `model_temperature` | number (0-1) | Controls randomness. Default 0.7 |
| `tool_call_strict_mode` | boolean | Strict mode for tool calls |
| `knowledge_base_ids` | string[] | KB IDs for RAG |
| `kb_config` | object | `{ "top_k": 3, "filter_score": 0.6 }` |
| `begin_after_user_silence_ms` | integer | Ms to wait before AI begins if user speaks first |
| `global_prompt` | string | Prompt used in every node |
| `tools` | array | Tools available in the flow (see Tools below) |
| `components` | array | Reusable sub-flows (see Components below) |
| `start_node_id` | string | ID of the start node |
| `default_dynamic_variables` | object | Key-value pairs accessible via `{{variable_name}}` |
| `is_transfer_llm` | boolean | Whether this flow is for transfer LLM |

### Complete Example Request

```json
{
  "model_choice": {
    "type": "cascading",
    "model": "gpt-4.1"
  },
  "model_temperature": 0.7,
  "start_speaker": "agent",
  "start_node_id": "greeting",
  "global_prompt": "You are a helpful customer service agent for Acme Corp.",
  "default_dynamic_variables": {
    "company_name": "Acme Corp",
    "support_hours": "9 AM - 5 PM EST"
  },
  "tools": [
    {
      "type": "custom",
      "name": "lookup_customer",
      "description": "Look up customer info by phone number",
      "url": "https://api.example.com/lookup",
      "method": "POST"
    }
  ],
  "nodes": [
    {
      "id": "greeting",
      "type": "conversation",
      "instruction": {
        "type": "prompt",
        "text": "Greet the customer warmly and ask how you can help."
      },
      "edges": [
        {
          "id": "edge_greeting_to_booking",
          "transition_condition": {
            "type": "prompt",
            "prompt": "Customer wants to book an appointment"
          },
          "destination_node_id": "book_appointment"
        },
        {
          "id": "edge_greeting_to_support",
          "transition_condition": {
            "type": "prompt",
            "prompt": "Customer has a support question"
          },
          "destination_node_id": "support"
        }
      ]
    },
    {
      "id": "book_appointment",
      "type": "conversation",
      "instruction": {
        "type": "prompt",
        "text": "Help the customer book an appointment. Ask for their preferred date and time."
      },
      "edges": [
        {
          "id": "edge_booking_to_end",
          "transition_condition": {
            "type": "prompt",
            "prompt": "Appointment has been confirmed"
          },
          "destination_node_id": "goodbye"
        }
      ]
    },
    {
      "id": "goodbye",
      "type": "end_call",
      "instruction": {
        "type": "prompt",
        "text": "Thank the customer and say goodbye."
      }
    }
  ]
}
```

### Response (201)

```json
{
  "conversation_flow_id": "cf_xxxxxxxxxxxxxxxx",
  "version": 1,
  "model_choice": { ... },
  "nodes": [ ... ],
  ...
}
```

---

## 2. Create Voice Agent

**`POST /create-agent`**

Creates the voice agent and attaches a conversation flow (or retell-llm) to it.

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `response_engine` | object | Links to conversation flow or LLM |
| `voice_id` | string | Voice ID from dashboard (e.g., `"11labs-Adrian"`) |

### response_engine Options

**For Conversation Flow agent:**
```json
{
  "type": "conversation-flow",
  "conversation_flow_id": "cf_xxxxxxxxxxxxxxxx"
}
```

**For Retell LLM agent:**
```json
{
  "type": "retell-llm",
  "llm_id": "llm_xxxxxxxxxxxxxxxx",
  "version": 0
}
```

### Optional Fields (commonly used)

| Field | Type | Description |
|-------|------|-------------|
| `agent_name` | string | Name for reference |
| `version_description` | string | Description of this version |
| `voice_model` | string | e.g. `"eleven_turbo_v2"`, `"sonic-3"`, `"tts-1"` |
| `fallback_voice_ids` | string[] | Fallback voices from different providers |
| `voice_temperature` | number (0-2) | Voice stability. Default 1 |
| `voice_speed` | number (0.5-2) | Speech speed. Default 1 |
| `volume` | number (0-2) | Agent volume. Default 1 |
| `voice_emotion` | string | `"calm"`, `"sympathetic"`, `"happy"` etc. (Cartesia/Minimax only) |
| `responsiveness` | number (0-1) | How quickly agent responds. Default 1 |
| `interruption_sensitivity` | number (0-1) | Ease of interrupting agent. Default 1. 0 = never interrupted |
| `enable_backchannel` | boolean | Agent says "yeah", "uh-huh" etc. |
| `backchannel_frequency` | number (0-1) | How often to backchannel. Default 0.8 |
| `backchannel_words` | string[] | Custom backchannel words |
| `reminder_trigger_ms` | number | Remind after silence. Default 10000 (10s) |
| `reminder_max_count` | integer | Max reminders. Default 1. 0 = disable |
| `ambient_sound` | string | `"coffee-shop"`, `"call-center"`, `"convention-hall"`, `"summer-outdoor"`, `"mountain-outdoor"`, `"static-noise"` |
| `ambient_sound_volume` | number (0-2) | Default 1 |
| `language` | string | `"en-US"`, `"en-GB"`, `"multi"`, etc. |
| `webhook_url` | string | URL for call event webhooks |
| `boosted_keywords` | string[] | Words to boost in transcription |
| `normalize_for_speech` | boolean | Normalize numbers, dates, currency for speech |
| `end_call_after_silence_ms` | integer | End call after silence. Default 600000 (10 min) |
| `max_call_duration_ms` | integer | Max call length. Default 3600000 (1 hour) |
| `enable_voicemail_detection` | boolean | Detect voicemail (phone calls only) |
| `voicemail_message` | string | Message for voicemail |
| `voicemail_detection_timeout_ms` | integer | Default 30000 (30s) |
| `begin_message_delay_ms` | integer (0-5000) | Delay before first message |
| `stt_mode` | string | `"fast"`, `"accurate"`, `"custom"` |
| `vocab_specialization` | string | `"general"` or `"medical"` (English only) |
| `denoising_mode` | string | `"none"`, `"noise-cancellation"`, `"noise-and-background-speech-cancellation"` |

### Post-Call Analysis Fields

| Field | Type | Description |
|-------|------|-------------|
| `post_call_analysis_data` | array | Custom data to extract after call |
| `post_call_analysis_model` | string | Model for analysis. Default `"gpt-4.1-mini"` |
| `analysis_successful_prompt` | string | Prompt to determine call success |
| `analysis_summary_prompt` | string | Prompt to generate call summary |

**post_call_analysis_data item types:**
- `"string"` — extract a text value
- `"number"` — extract a number
- `"boolean"` — extract true/false
- `"enum"` — extract from predefined options

```json
{
  "post_call_analysis_data": [
    {
      "type": "string",
      "name": "customer_name",
      "description": "The name of the customer.",
      "examples": ["John Doe", "Jane Smith"]
    },
    {
      "type": "enum",
      "name": "call_outcome",
      "description": "The outcome of the call.",
      "choices": ["appointment_booked", "callback_requested", "not_interested", "wrong_number"]
    }
  ]
}
```

### Voicemail Option

```json
{
  "voicemail_option": {
    "action": {
      "type": "static_text",
      "text": "Hi, this is a message from Acme Corp. Please call us back."
    }
  }
}
```

Or to hang up on voicemail:
```json
{
  "voicemail_option": {
    "action": { "type": "hangup" }
  }
}
```

### IVR Option

```json
{
  "ivr_option": {
    "action": { "type": "hangup" }
  }
}
```

### Complete Example Request

```json
{
  "response_engine": {
    "type": "conversation-flow",
    "conversation_flow_id": "cf_xxxxxxxxxxxxxxxx"
  },
  "voice_id": "11labs-Adrian",
  "agent_name": "Acme Support Agent",
  "language": "en-US",
  "voice_speed": 1,
  "responsiveness": 0.9,
  "interruption_sensitivity": 0.8,
  "enable_backchannel": true,
  "backchannel_frequency": 0.8,
  "reminder_trigger_ms": 10000,
  "reminder_max_count": 2,
  "normalize_for_speech": true,
  "enable_voicemail_detection": true,
  "voicemail_message": "Hi, this is Acme Corp. Please give us a callback.",
  "boosted_keywords": ["Acme", "appointment"],
  "post_call_analysis_data": [
    {
      "type": "string",
      "name": "customer_name",
      "description": "The name of the customer"
    }
  ]
}
```

### Response (201)

```json
{
  "agent_id": "oBeDLoLOeuAbiuaMFXRtDOLriTJ5tSxD",
  "version": 0,
  "response_engine": { ... },
  "voice_id": "11labs-Adrian",
  "is_published": false,
  ...
}
```

---

## Node Types

Nodes are the building blocks of a conversation flow. Each node has an `id`, `type`, and type-specific fields.

### Conversation Node (`type: "conversation"`)

The most common node. Handles dialogue with the user. Supports multi-turn conversation within a single node.

```json
{
  "id": "collect_info",
  "type": "conversation",
  "instruction": {
    "type": "prompt",
    "text": "Ask the customer for their name and phone number."
  },
  "edges": [
    {
      "id": "edge_1",
      "transition_condition": {
        "type": "prompt",
        "prompt": "Customer has provided their name and phone number"
      },
      "destination_node_id": "next_node"
    }
  ]
}
```

**Instruction types:**
- `"prompt"` — dynamically generate speech from a prompt
- `"static"` — say a fixed sentence first, then generate dynamically

**Optional node settings:**
- `skip_response` (boolean) — transition immediately after agent speaks (no user response needed). Useful for disclaimers. Only one edge allowed.
- `global_node` (boolean) — can be transitioned to from any node. Requires a transition condition to enter.
- `block_interruptions` (boolean) — prevent user from interrupting agent speech.
- `model_override` — use a different LLM model for this specific node.

### Function Node (`type: "function"`)

Executes a custom function (API call) during conversation.

```json
{
  "id": "lookup_customer",
  "type": "function",
  "function_name": "lookup_customer",
  "speak_during_execution": {
    "type": "prompt",
    "text": "Let me look that up for you."
  },
  "edges": [
    {
      "id": "edge_func_success",
      "transition_condition": {
        "type": "prompt",
        "prompt": "Customer lookup was successful"
      },
      "destination_node_id": "show_results"
    },
    {
      "id": "edge_func_fail",
      "transition_condition": {
        "type": "prompt",
        "prompt": "Customer lookup failed or customer not found"
      },
      "destination_node_id": "manual_collect"
    }
  ]
}
```

**Transition timing for function nodes:**
- With `speak_during_execution` ON: transitions after function result is ready AND agent finishes speaking
- With `speak_during_execution` OFF: transitions immediately after function is invoked

### Call Transfer Node (`type: "call_transfer"`)

Transfers the call to another phone number. Agent does not speak in this node. Only works for phone calls (not web calls).

```json
{
  "id": "transfer_to_human",
  "type": "call_transfer",
  "transfer_destination": "+14155551234",
  "edges": [
    {
      "id": "edge_transfer_fail",
      "transition_condition": {
        "type": "prompt",
        "prompt": "Transfer was unsuccessful"
      },
      "destination_node_id": "transfer_failed"
    }
  ]
}
```

**Tips:**
- Number must be in E.164 format or SIP URI (`sip:user@domain`)
- Pre-populate a conversation node with `skip_response: true` before transfer to say "Let me transfer you"
- Has a pre-populated edge for transfer failure

### End Node (`type: "end_call"`)

Ends the call. No edges. Call ends the moment agent enters this node.

```json
{
  "id": "goodbye",
  "type": "end_call",
  "instruction": {
    "type": "prompt",
    "text": "Thank the customer and wish them a good day."
  }
}
```

**Optional:** `speak_during_execution` — agent speaks a goodbye message before hanging up.

### Logic Split Node (`type: "logic_split"`)

Routes conversation based on dynamic variable values (equation conditions). No speech occurs.

```json
{
  "id": "check_state",
  "type": "logic_split",
  "edges": [
    {
      "id": "edge_ca",
      "transition_condition": {
        "type": "equation",
        "lhs": "{{user_state}}",
        "operator": "==",
        "rhs": "California"
      },
      "destination_node_id": "ca_flow"
    },
    {
      "id": "edge_ny",
      "transition_condition": {
        "type": "equation",
        "lhs": "{{user_state}}",
        "operator": "==",
        "rhs": "New York"
      },
      "destination_node_id": "ny_flow"
    },
    {
      "id": "edge_default",
      "transition_condition": {
        "type": "prompt",
        "prompt": "Default fallback"
      },
      "destination_node_id": "general_flow"
    }
  ]
}
```

### SMS Node (`type: "sms"`)

Sends an SMS message during the call.

### Press Digit Node (`type: "press_digit"`)

Sends DTMF tones.

### Extract Dynamic Variable Node (`type: "extract_dv"`)

Extracts and stores dynamic variables from the conversation.

### Agent Transfer Node (`type: "agent_transfer"`)

Transfers to another Retell agent.

### MCP Node (`type: "mcp"`)

Calls an MCP (Model Context Protocol) server.

---

## Edges & Transition Conditions

Edges connect nodes and define when transitions occur.

### Edge Structure

```json
{
  "id": "unique_edge_id",
  "transition_condition": { ... },
  "destination_node_id": "target_node_id"
}
```

### Transition Condition Types

**Prompt condition** — LLM evaluates whether the condition is met:
```json
{
  "type": "prompt",
  "prompt": "Customer wants to book an appointment"
}
```

**Equation condition** — evaluates dynamic variables:
```json
{
  "type": "equation",
  "lhs": "{{variable_name}}",
  "operator": "==",
  "rhs": "expected_value"
}
```

**Equation operators:** `==`, `!=`, `Contains`, `Not Contains`, `exists`, `does not exist`

Note: All equation comparisons are string-based.

### When Transitions Are Checked

- **Conversation nodes**: After user finishes speaking. Also after agent finishes speaking if `skip_response` is enabled.
- **Function nodes**: After function result is ready (and after agent speaks if `speak_during_execution` is on).
- **Call Transfer nodes**: When transfer fails.

### Best Practices

- Cover all possible cases in transition conditions to prevent the agent from getting stuck.
- Use global nodes for universal scenarios (objections, "not a good time", etc.).
- Write clear, specific transition conditions that don't heavily reference the node instruction.
- You can reference function results in transition conditions for function nodes.

---

## Tools (Custom Functions)

Tools are defined at the flow level and referenced in function nodes.

### Custom Tool Definition

```json
{
  "type": "custom",
  "name": "get_customer_info",
  "description": "Get customer information from database",
  "tool_id": "tool_001",
  "url": "https://your-webhook-url.com/endpoint",
  "method": "POST"
}
```

**Methods:** `GET`, `POST`, `PUT`, `PATCH`, `DELETE`

### Custom Function Request Spec

When a custom function is called, Retell sends a request to your URL:

**Request headers:**
- `X-Retell-Signature` — HMAC signature for verification
- `Content-Type: application/json`

**Request body (for POST/PUT/PATCH):**
```json
{
  "name": "function_name",
  "call": { /* call object with transcript, call_id, etc. */ },
  "args": { /* function arguments as JSON */ }
}
```

**Response:** Return status 200-299 with string, JSON, or buffer. Result is capped at 15,000 characters.

**Timeout:** Configurable, defaults to 2 minutes. Failed requests are retried up to 2 times.

### Verifying Requests from Retell

```javascript
import { Retell } from "retell-sdk";

const isValid = Retell.verify(
  JSON.stringify(req.body),
  process.env.RETELL_API_KEY,
  req.headers["x-retell-signature"]
);
```

Retell's IP address for allowlisting: `100.20.5.228`

### Response Variables

You can extract values from the API response and save them as dynamic variables:
- Configure in the function definition
- Access later in conversation using `{{variable_name}}`

---

## Components

Components are reusable sub-flows that can be embedded within a conversation flow.

```json
{
  "components": [
    {
      "name": "Customer Information Collector",
      "nodes": [
        {
          "id": "collect_info",
          "type": "conversation",
          "instruction": {
            "type": "prompt",
            "text": "Ask for name and contact info."
          }
        }
      ],
      "tools": [],
      "start_node_id": "collect_info",
      "begin_tag_display_position": { "x": 100, "y": 200 }
    }
  ]
}
```

---

## Dynamic Variables

Dynamic variables can be used throughout the conversation flow:

- **Default variables**: Set in `default_dynamic_variables` at flow creation
- **Runtime variables**: Passed via `retell_llm_dynamic_variables` when creating a call
- **Extracted variables**: Set from custom function responses
- **Reference syntax**: `{{variable_name}}`

---

## SDK Usage

### Node.js SDK

```bash
npm install retell-sdk
```

```javascript
import Retell from 'retell-sdk';

const client = new Retell({
  apiKey: 'YOUR_RETELL_API_KEY',
});

// Create conversation flow
const flow = await client.conversationFlow.create({
  model_choice: { model: 'gpt-4.1', type: 'cascading' },
  start_speaker: 'agent',
  nodes: [ /* ... */ ],
});

// Create voice agent
const agent = await client.agent.create({
  response_engine: {
    type: 'conversation-flow',
    conversation_flow_id: flow.conversation_flow_id,
  },
  voice_id: '11labs-Adrian',
});
```

### REST API (cURL)

```bash
# Create conversation flow
curl -X POST https://api.retellai.com/create-conversation-flow \
  -H "Authorization: Bearer YOUR_RETELL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ ... }'

# Create voice agent
curl -X POST https://api.retellai.com/create-agent \
  -H "Authorization: Bearer YOUR_RETELL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ ... }'

# Create phone call
curl -X POST https://api.retellai.com/create-phone-call \
  -H "Authorization: Bearer YOUR_RETELL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from_number": "+14157774444",
    "to_number": "+12137774445"
  }'
```

---

## Other Useful Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/get-conversation-flow/{id}` | Get a conversation flow |
| GET | `/list-conversation-flows` | List all conversation flows |
| PATCH | `/update-conversation-flow/{id}` | Update a conversation flow |
| DELETE | `/delete-conversation-flow/{id}` | Delete a conversation flow |
| GET | `/get-agent/{id}` | Get a voice agent |
| GET | `/list-agents` | List all voice agents |
| PATCH | `/update-agent/{id}` | Update a voice agent |
| DELETE | `/delete-agent/{id}` | Delete a voice agent |
| POST | `/publish-agent/{id}` | Publish an agent version |
| POST | `/create-phone-call` | Create an outbound phone call |
| POST | `/create-web-call` | Create a web call |
| GET | `/get-call/{id}` | Get call details |
| POST | `/list-calls` | List calls |
| POST | `/search-voice` | Search available voices |
| GET | `/list-voices` | List all voices |

---

## Available Models

For `model_choice.model` in conversation flows:
- `gpt-4.1` (recommended)
- `gpt-4.1-mini`
- `gpt-4.1-nano`
- `gpt-5`
- `gpt-5-mini`
- `claude-4.5-sonnet`
- `claude-4.5-haiku`
- `gemini-2.5-flash`
- `gemini-2.5-flash-lite`

For `post_call_analysis_model`:
Same models as above.

---

## Common Voice IDs

Check the Retell dashboard for a full list with previews. Examples:
- `11labs-Adrian`
- `11labs-Rachel`
- `openai-Alloy`
- `openai-Nova`
- `openai-Shimmer`
- `deepgram-Angus`

---

## Error Codes

| Code | Description |
|------|-------------|
| 400 | Bad request — invalid parameters |
| 401 | Unauthorized — invalid API key |
| 422 | Unprocessable entity — validation error |
| 429 | Rate limited |
| 500 | Internal server error |

---

## Global Node Pattern

For handling universal scenarios (objections, "not a good time", etc.) from any point in the conversation:

```json
{
  "id": "objection_handler",
  "type": "conversation",
  "global_node": true,
  "instruction": {
    "type": "prompt",
    "text": "The caller has indicated this is not a good time. Acknowledge their concern, ask when would be a better time to call back, and offer to schedule a callback."
  },
  "edges": [
    {
      "id": "edge_objection_enter",
      "transition_condition": {
        "type": "prompt",
        "prompt": "User indicates this is not a good time to talk or wants to call back later"
      },
      "destination_node_id": "objection_handler"
    },
    {
      "id": "edge_objection_to_end",
      "transition_condition": {
        "type": "prompt",
        "prompt": "Callback time has been noted or user wants to end the call"
      },
      "destination_node_id": "goodbye"
    }
  ]
}
```

---

## Pronunciation Dictionary

For consistent pronunciation of domain-specific terms:

```json
{
  "pronunciation_dictionary": [
    {
      "word": "Acme",
      "alphabet": "ipa",
      "phoneme": "ˈækmi"
    }
  ]
}
```
