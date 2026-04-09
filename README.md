# Building a HOLOCRON: Persistent AI Memory for Security Work

> Built and documented by **Belouve** | BHIS Community Leader | DC608 DEF CON Local Group  
> Base system by Nate B. Jones — all core infrastructure credit to the OB1 project.
>
> Was discussed on [AI Security Ops Podcast](https://www.youtube.com/@AISecurityOps)

---

## What This Is

This guide documents how to build a personal AI knowledge base, which Belouve called a **HOLOCRON** , that gives every AI you use a persistent, searchable memory of everything you've captured. Claude, ChatGPT, whatever ships next month. One database. All of them.

The base OB1 system handles the infrastructure. This guide adds:

- **Discord as the capture interface** (instead of Slack)
- **Multi-channel capture** with per-channel auto-tagging
- **CTI-aware metadata extraction** for threat intelligence workflows
- **Context bucket isolation** for separating professional, personal, and sensitive content
- **Protocol CTI-ALPHA** — an evergreen monthly threat intel briefing command, as an example you can use right away
- **Windows-specific gotchas** documented throughout

If you work in blue team, red team, or threat intelligence, the use cases go somewhere interesting fast.

---

## Why Three (or more) LLMs

Running Claude, ChatGPT, and Gemini against the same shared memory creates a **consensus layer**. One model hallucinates, but the others don't confirm it. Think of it as a Minority Report brain: three independent readers, one shared record. For security work where accuracy matters, this isn't a gimmick, and it's a meaningful reliability improvement.

---

## What You're Building

```
Discord Channels → Edge Function → Supabase (vector DB)
                                        ↑
Claude / ChatGPT / Gemini ←→ MCP Server (hosted)
```

- **Supabase** — your database, vector search, and API
- **OpenRouter** — AI gateway for embeddings and metadata extraction
- **Discord** — capture interface (there are guides for also using Slack)
- **MCP Server** — lets any AI read and write to your brain via a URL

---

## Prerequisites

- Windows with PowerShell (guide is Windows-native throughout)
- A Discord account and server you control
- Supabase account (free tier)
- OpenRouter account (free, ~$5 in credits lasts months)
- Claude Desktop installed (If using Claude as primary AI interface)
- Supabase CLI installed via Scoop (we have instructions [here](https://github.com/belouve/open-brain-holocron/edit/main/README.md#windows-specific-notes-for-part-1) )

---

## Credential Tracker

Copy this into a text file and fill in as you go. You will need every value across multiple steps.

```
HOLOCRON CREDENTIAL TRACKER
-----------------------------
Supabase Project Ref:         ____________
Supabase Project URL:         ____________
Supabase Secret Key:          ____________
OpenRouter API Key:           ____________
MCP Access Key:               ____________
MCP Server URL:               https://____________.supabase.co/functions/v1/open-brain-mcp
MCP Connection URL:           https://____________.supabase.co/functions/v1/open-brain-mcp?key=____________

Discord Bot Token:            ____________
Discord Public Key:           ____________
Discord Application ID:       ____________
Discord Server ID:            ____________
Discord Command ID:           ____________

Channel: #holocron-capture    ID: ____________
Channel: #cti-inbox           ID: ____________
Channel: #personal-private    ID: ____________  (optional, personal/private)

Webhook: #holocron-capture    ____________
Webhook: #cti-inbox           ____________
```

> ⚠️ **Windows/Excel warning:** If you use Excel as your tracker, Discord and Supabase IDs are 18-19 digit numbers. Excel will silently mangle them by rounding or adding trailing zeros. Use a plain text file, Notepad++, or a text column format in Excel. Always verify IDs copied from Excel against the Discord Developer Portal before using them in code. This...was a significant headache for Belouve.

---

## Part 1 — Core Infrastructure (Steps 1–7)

Follow the base guide for steps 1–6:  
**[Base Guide For initial steps](https://github.com/belouve/open-brain-holocron/blob/main/01-getting-started.md)**

Also consult the Windows-specific notes below, these are mostly some hiccups that were smoothed out when Belouve built his HOLOCRON.

This covers:
- Supabase project creation
- Database setup (pgvector, thoughts table, match_thoughts function, RLS policy)
- OpenRouter API key
- MCP Access Key generation
- MCP server deployment
- Connecting Claude Desktop and ChatGPT

### Windows-Specific Notes for Part 1

**Supabase CLI installation:**
```powershell
# Install Scoop first if needed
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression

# Install Supabase via Scoop
scoop bucket add supabase https://github.com/supabase/scoop-bucket.git
scoop install supabase

# Update when prompted (you will see update nags — keep current)
scoop update supabase
```

**Project folder setup:**
```powershell
# Navigate to your project folder before ANY supabase command
cd "C:\Projects\open-brain"   # adjust to your path
Get-Location                        # always verify before proceeding
```

**Downloading function files — use Invoke-WebRequest:**
```powershell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/NateBJones-Projects/OB1/main/server/index.ts `
  -OutFile supabase\functions\open-brain-mcp\index.ts
```

**Verify download worked:**
```powershell
Get-Content supabase\functions\open-brain-mcp\index.ts -Head 1
# Should show: import "jsr:@supabase/functions-js/edge-runtime.d.ts";
# If it shows: console.log("Hello from Functions!") — download failed, redo it
```

**MCP Access Key — reset and verify process:**

The `supabase secrets list` command shows secret names and a truncated DIGEST — not the full value. You cannot verify the stored key from the CLI. If you ever get 401 errors connecting an AI client:

```powershell
# Generate a new key and set it in one command
$newkey = -join ((1..32) | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) })
Write-Host "Your new key: $newkey"
supabase secrets set MCP_ACCESS_KEY=$newkey

# Then redeploy
supabase functions deploy open-brain-mcp --no-verify-jwt
```

Copy the printed key immediately and update your credential tracker AND your MCP Connection URL.

**Test your MCP server before connecting any AI:**

Paste your MCP Connection URL (with `?key=`) into a browser. A working server returns:
```
{"jsonrpc":"2.0","error":{"code":-32000,"message":"Not Acceptable: Client must accept text/event-stream"},"id":null}
```
This is correct. The server is live and responding. A browser is not an MCP client — this error is expected.

If you get a 404, check your URL structure. The correct format is:
```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=YOUR_KEY
```
Not:
```
https://supabase.com/dashboard/project/YOUR_PROJECT_REF.supabase.co/...
```

---

## Part 2 — Discord Capture Integration

The guide initially used Slack. This section replaces Slack entirely with Discord and adds security-workflow specific features.

### Step 8 — Create Your Discord Bot

**8.1 Create the Application**

1. Go to **discord.com/developers/applications** → **New Application**
2. Name it `HOLOCRON Capture`
3. Go to **General Information** → copy **Application ID** and **Public Key** → save to tracker

**8.2 Configure the Bot**

1. Left sidebar → **Bot**
2. Scroll to **Privileged Gateway Intents** → enable **Message Content Intent**
3. Click **Reset Token** → copy the Bot Token → save to tracker immediately
4. Save changes

> ⚠️ **Token exposure:** If your bot token is ever exposed in a chat, screenshot, or shared document — regenerate it immediately in the Developer Portal and update your Supabase secret. A leaked token gives anyone full control of your bot.

**8.3 Invite the Bot to Your Server**

1. Left sidebar → **OAuth2** → **URL Generator**
2. Scopes: check `bot`
3. Bot Permissions: check `View Channels`, `Read Message History`, `Send Messages`, `Add Reactions`
4. Copy the generated URL → paste in browser → select your server → Authorize

**8.4 Set Up Discord Channels**

Create these channels in your server:

| Channel | Purpose | Auto-tag |
|---|---|---|
| `#holocron-capture` | General thoughts, notes, ideas | none |
| `#cti-inbox` | Threat intel articles and findings | `cti` |
| `#personal-private` | Personal/private content (optional) | `personal-private` |

Organize these under a channel group/category. New channels added to that group automatically inherit bot access.

Get each Channel ID: right-click channel → **Copy Channel ID** (requires Developer Mode: User Settings → Advanced → Developer Mode).

> ⚠️ **Excel ID mangling:** Paste channel IDs into Notepad or a text file first. Verify digit count (should be 18-19 digits). Never trust Excel with these values. This...was a significant headache for Belouve. #askmehowiknow

**8.5 Create Webhook URLs for Confirmation Replies**

For each channel: right-click → **Edit Channel** → **Integrations** → **Webhooks** → **New Webhook** → copy URL → save to tracker.

These webhooks are how the bot sends confirmation replies back into the channel.

---

### Step 9 — Deploy the Discord Capture Function

**9.1 Create the function folder:**
```powershell
supabase functions new discord-capture
```

**9.2 Create `index.ts`** in Notepad++ (or your preferred editor) at:
```
supabase\functions\discord-capture\index.ts
```

Paste the full function code below:

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY")!;
const DISCORD_BOT_TOKEN = Deno.env.get("DISCORD_BOT_TOKEN")!;
const DISCORD_PUBLIC_KEY = Deno.env.get("DISCORD_PUBLIC_KEY")!;

// Channel IDs — replace with your own
const CTI_CHANNEL_ID = "YOUR_CTI_CHANNEL_ID";
const CAPTURE_CHANNEL_ID = "YOUR_HOLOCRON_CAPTURE_CHANNEL_ID";
const PERSONAL_PRIVATE_CHANNEL_ID = "YOUR_PERSONAL_PRIVATE_CHANNEL_ID";

// Webhook URLs for confirmation replies — replace with your own
const HOLOCRON_WEBHOOK = "YOUR_HOLOCRON_CAPTURE_WEBHOOK_URL";
const CTI_WEBHOOK = "YOUR_CTI_INBOX_WEBHOOK_URL";

const OPENROUTER_BASE = "https://openrouter.ai/api/v1";
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

function hexToBytes(hex: string): Uint8Array {
  const bytes = new Uint8Array(hex.length / 2);
  for (let i = 0; i < hex.length; i += 2) {
    bytes[i / 2] = parseInt(hex.slice(i, i + 2), 16);
  }
  return bytes;
}

async function verifyDiscordSignature(req: Request, body: string): Promise<boolean> {
  const signature = req.headers.get("x-signature-ed25519");
  const timestamp = req.headers.get("x-signature-timestamp");
  if (!signature || !timestamp) return false;
  const key = await crypto.subtle.importKey(
    "raw",
    hexToBytes(DISCORD_PUBLIC_KEY),
    { name: "Ed25519", namedCurve: "Ed25519" },
    false,
    ["verify"]
  );
  return await crypto.subtle.verify(
    "Ed25519",
    key,
    hexToBytes(signature),
    new TextEncoder().encode(timestamp + body)
  );
}

async function getEmbedding(text: string): Promise<number[]> {
  const r = await fetch(`${OPENROUTER_BASE}/embeddings`, {
    method: "POST",
    headers: { "Authorization": `Bearer ${OPENROUTER_API_KEY}`, "Content-Type": "application/json" },
    body: JSON.stringify({ model: "openai/text-embedding-3-small", input: text }),
  });
  const d = await r.json();
  return d.data[0].embedding;
}

async function extractMetadata(text: string, isCTI: boolean): Promise<Record<string, unknown>> {
  const systemHint = isCTI
    ? "This is a cyber threat intelligence capture. Prioritize security-relevant topics, threat actors, CVEs, IOCs, and TTPs in your extraction."
    : "Extract metadata from the user's captured thought.";

  const r = await fetch(`${OPENROUTER_BASE}/chat/completions`, {
    method: "POST",
    headers: { "Authorization": `Bearer ${OPENROUTER_API_KEY}`, "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "openai/gpt-4o-mini",
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `${systemHint} Return JSON with:
- "people": array of people mentioned (empty if none)
- "action_items": array of implied to-dos (empty if none)
- "dates_mentioned": array of dates YYYY-MM-DD (empty if none)
- "topics": array of 1-3 short topic tags (always at least one)
- "type": one of "observation", "task", "idea", "reference", "person_note"
Only extract what's explicitly there.`,
        },
        { role: "user", content: text },
      ],
    }),
  });
  const d = await r.json();
  try { return JSON.parse(d.choices[0].message.content); }
  catch { return { topics: ["uncategorized"], type: "observation" }; }
}

async function sendWebhookReply(webhookUrl: string, content: string): Promise<void> {
  await fetch(webhookUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ content }),
  });
}

function getChannelContext(channelId: string): { label: string; autoTag: string | null; isCTI: boolean; emoji: string } {
  if (channelId === CTI_CHANNEL_ID) {
    return { label: "cti-inbox", autoTag: "cti", isCTI: true, emoji: "🛡️" };
  }
  if (channelId === PERSONAL_PRIVATE_CHANNEL_ID) {
    return { label: " personal-private", autoTag: " personal-private", isCTI: false, emoji: "🌙" };
  }
  return { label: "holocron-capture", autoTag: null, isCTI: false, emoji: "🧠" };
}

async function captureThought(text: string, channelId: string, messageId: string, manualTag?: string): Promise<string> {
  const { label: channelLabel, autoTag, isCTI, emoji } = getChannelContext(channelId);

  // Deduplicate
  const { data: existing } = await supabase
    .from("thoughts")
    .select("id")
    .contains("metadata", { discord_message_id: messageId })
    .limit(1);
  if (existing && existing.length > 0) return "Already captured.";

  const [embedding, metadata] = await Promise.all([
    getEmbedding(text),
    extractMetadata(text, isCTI),
  ]);

  const { error } = await supabase.from("thoughts").insert({
    content: text,
    embedding,
    metadata: {
      ...metadata,
      source: "discord",
      channel: channelLabel,
      discord_message_id: messageId,
      ...(autoTag ? { context: autoTag } : {}),
      ...(manualTag ? { user_tag: manualTag } : {}),
    },
  });

  if (error) {
    console.error("Supabase insert error:", error);
    return `❌ Failed to capture: ${error.message}`;
  }

  const meta = metadata as Record<string, unknown>;
  let confirmation = `${emoji} Captured as **${meta.type || "thought"}**`;
  if (Array.isArray(meta.topics) && meta.topics.length > 0)
    confirmation += ` — ${meta.topics.join(", ")}`;
  if (Array.isArray(meta.people) && meta.people.length > 0)
    confirmation += `\nPeople: ${meta.people.join(", ")}`;
  if (Array.isArray(meta.action_items) && meta.action_items.length > 0)
    confirmation += `\nAction items: ${meta.action_items.join("; ")}`;

  return confirmation;
}

Deno.serve(async (req: Request): Promise<Response> => {
  try {
    const rawBody = await req.text();

    // Verify Discord signature — required, Discord rejects without this
    const isValid = await verifyDiscordSignature(req, rawBody);
    if (!isValid) return new Response("Invalid signature", { status: 401 });

    const body = JSON.parse(rawBody);

    // PING verification — Discord sends this when you set the Interactions Endpoint URL
    if (body.type === 1) {
      return new Response(JSON.stringify({ type: 1 }), {
        headers: { "Content-Type": "application/json" },
      });
    }

    // Slash command: /capture
    if (body.type === 2 && body.data?.name === "capture") {
      const text = body.data.options?.find((o: {name: string}) => o.name === "thought")?.value ?? "";
      const manualTag = body.data.options?.find((o: {name: string}) => o.name === "tags")?.value ?? undefined;
      const channelId: string = body.channel_id;
      const interactionId: string = body.id;
      const interactionToken: string = body.token;

      if (!text.trim()) {
        return new Response(
          JSON.stringify({ type: 4, data: { content: "Please provide a thought to capture.", flags: 64 } }),
          { headers: { "Content-Type": "application/json" } }
        );
      }

      // Acknowledge immediately — Discord requires response within 3 seconds
      const ackResponse = new Response(
        JSON.stringify({ type: 5 }),
        { headers: { "Content-Type": "application/json" } }
      );

      // Process in background
      EdgeRuntime.waitUntil((async () => {
        const confirmation = await captureThought(text, channelId, interactionId, manualTag);
        await fetch(
          `https://discord.com/api/v10/webhooks/YOUR_APPLICATION_ID/${interactionToken}/messages/@original`,
          {
            method: "PATCH",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ content: confirmation }),
          }
        );
      })());

      return ackResponse;
    }

    return new Response("ok", { status: 200 });

  } catch (err) {
    console.error("Function error:", err);
    return new Response("error", { status: 500 });
  }
});
```

> ⚠️ Replace all `YOUR_*` placeholders with your actual values before saving.
> Replace `YOUR_APPLICATION_ID` in the webhook PATCH URL with your Discord Application ID.

**9.3 Set Supabase secrets:**
```powershell
supabase secrets set DISCORD_BOT_TOKEN=your-bot-token
supabase secrets set DISCORD_PUBLIC_KEY=your-public-key
```

**9.4 Deploy:**
```powershell
supabase functions deploy discord-capture --no-verify-jwt
```

Copy the function URL from the Supabase dashboard → Functions. It looks like:
```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/discord-capture
```

**9.5 Set the Interactions Endpoint URL in Discord:**

1. Discord Developer Portal → your app → **General Information**
2. Paste your function URL into **Interactions Endpoint URL**
3. Click Save — Discord sends a PING, your function returns `{ type: 1 }`, it verifies green ✅

---

### Step 10 — Register the /capture Slash Command

Create `register-command.ps1` in your project folder:

```powershell
$appId = "YOUR_APPLICATION_ID"
$guildId = "YOUR_SERVER_ID"
$botToken = "YOUR_BOT_TOKEN"

$json = '{"name":"capture","description":"Capture a thought to the HOLOCRON","options":[{"name":"thought","description":"The thought to capture","type":3,"required":true},{"name":"tags","description":"Optional tag (e.g. Malware, Phishing, CTI,  personal-private)","type":3,"required":false}]}'

curl.exe -X POST "https://discord.com/api/v10/applications/$appId/guilds/$guildId/commands" `
    -H "Authorization: Bot $botToken" `
    -H "Content-Type: application/json" `
    -d $json
```

Run it:
```powershell
.\register-command.ps1
```

Success response includes `"name":"capture"` and an `"id"` field. Save the command ID to your tracker.

> **Note:** Use `curl.exe` not `curl` in PowerShell. Windows aliases `curl` to `Invoke-WebRequest` which handles JSON encoding differently and will cause errors. `curl.exe` calls the actual curl binary.

> **Note:** Register as a **guild command** (`/guilds/YOUR_SERVER_ID/commands`) not a global command. Guild commands register instantly and don't have the network restrictions that global commands have.

---

### Step 11 — Test All Three Channels

In Discord, test each channel with `/capture`:

| Channel | Test | Expected Response |
|---|---|---|
| `#holocron-capture` | `/capture` thought: "test" tags: "test" | 🧠 Captured as... |
| `#cti-inbox` | `/capture` thought: "CVE test capture" (no tags) | 🛡️ Captured as... |
| `#personal-private` | `/capture` thought: "test personal capture" (no tags) | 🌙 Captured as... |

The `cti` and ` personal-private` context tags are applied automatically based on channel — you don't need to type them.

---

## Part 3 — Context Buckets and Protocols

### Context Separation

The HOLOCRON uses metadata tags to separate content by context. This is behavioral isolation, not technical — Claude respects the boundaries because they're defined in the HOLOCRON itself.

Capture this rule into your HOLOCRON once it's running:

> "Context separation rule: Do not mix professional/CTI context with personal/ personal-private context in outputs. Only surface personal-private content when explicitly requested. Never reference personal hobbies or lifestyle details in work-related outputs."

**Context buckets:**

| Bucket | Tag | Use for |
|---|---|---|
| Professional/Work | `work` | CTI, employer-related, security work |
| Technical/Projects | `technical` | Build notes, configs, project tracking |
| Personal/Life | `personal` | General notes |
| CTI | `cti` | Threat intel (auto-applied in #cti-inbox) |
| Personal/Private | `personal-private` | Private personal content (auto-applied in # personal-private) |

---

### Protocol CTI-ALPHA — Monthly Threat Intel Briefing

Edit the areas to make it specific to your sector (FINANCIAL, MEDICAL, INDSUTRIAL, etc)
Paste this query into any HOLOCRON-connected AI to generate your monthly briefing.
This is an initial version to get you started, you can tune and tweak and update what it saves, then can best be invoked by saying, even in a new cold chat:
Connect to open brain HOLOCRON and run CTI-ALPHA.

That can even be tuned in context with "...run CTI-ALPHA with emphasis on ransomware threats"

```
Save the following to open brain HOLOCRON as CTI-ALPHA, to be invoked whenever I say "Run Protocol CTI-ALPHA".  Future updates can be edited with a note of version number, and to superced previous versions.

You are preparing a monthly Cyber Threat Intelligence briefing for an Information 
Security team at a [EDIT FOR YOUR TYPE OF] company.

Using the HOLOCRON search_thoughts and list_thoughts tools, retrieve all CTI intel 
captured over the past 30 days. Cast a wide net — search for: vulnerability exploits, 
malware, phishing, fraud, cybercrime, ransomware, APT, supply chain, and any [EDIT FOR YOURE SECTOR] sector threats.

Produce a structured 10-minute readout briefing:

MONTHLY THREAT INTELLIGENCE BRIEFING — [Month Year]
Prepared for: Information Security Team

EXECUTIVE SUMMARY
2-3 sentences on the overall threat landscape this month.

PRIORITY: [EDIT TO YOUR INDUSTRY] FINTECH & FINANCIAL SECTOR THREATS
Example: FINTECH & FINANCIAL SECTOR, MEDICAL SECTOR, INDUSTRIAL CONTROLS SECTOR
Lead with anything directly relevant to (EDIT FOR YOUR SECTOR TERMS!!!): financial technology, payment systems, or 
financial data. For each: threat name, what it does, why it matters, recommended action.

ELEVATED THREATS — BROADER LANDSCAPE
Other significant threats worth awareness. Same format.

ACTION ITEMS FOR THIS TEAM
Specific actions for compliance, testers, security awareness, or defensive blue team.

SOURCES CAPTURED THIS MONTH
Brief list of what was ingested and where it came from.

Keep tone professional but accessible — non-technical stakeholders are in the audience. 
Flag anything with active exploitation or KEV deadlines prominently.
```

**Invoke as:** `Run Protocol CTI-ALPHA`

---

## Security Considerations

### What the HOLOCRON Knows

Your HOLOCRON accumulates sensitive professional context over time — threat research, employer details, tooling notes. Treat it with the same care as any internal knowledge base.

**Access control:**
- The MCP Connection URL with `?key=` is your only authentication layer
- Anyone with that URL and key has full read/write access to your brain
- Store it like a password — credential manager, not a sticky note

**Data isolation options:**
- Single Supabase project with context bucket tags — behavioral isolation only
- Separate Supabase projects for sensitive content buckets — true technical isolation, recommended for anything you'd classify as confidential

**For red team engagements:**
Consider a dedicated HOLOCRON instance per engagement that gets destroyed at engagement close. The architecture supports this — spin up a new Supabase project, connect it, work, export, destroy.

### Bot Token Security

The Discord bot token grants full control of your bot. If exposed:
1. Regenerate immediately in the Developer Portal
2. Update the Supabase secret: `supabase secrets set DISCORD_BOT_TOKEN=new-token`
3. Redeploy: `supabase functions deploy discord-capture --no-verify-jwt`

---

## Troubleshooting

### 401 errors connecting AI client
MCP Access Key mismatch. The `supabase secrets list` DIGEST is truncated — you cannot verify the full stored value from the CLI. Reset the key:
```powershell
$newkey = -join ((1..32) | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) })
Write-Host "Your new key: $newkey"
supabase secrets set MCP_ACCESS_KEY=$newkey
supabase functions deploy open-brain-mcp --no-verify-jwt
```

### Discord slash command returns internal network error (code 40333)
Use `curl.exe` instead of `Invoke-WebRequest`. Register as a guild command, not a global command.

### Discord bot shows offline
Expected until the edge function is deployed and the Interactions Endpoint URL is set. "Offline" status is normal for interaction-based bots that don't use the Gateway.

### Function deployed but logs are empty
The AI client never reached the function. Check the MCP Connection URL structure — the most common issue is the Supabase dashboard URL prefix accidentally included.

### Captures not appearing in Supabase
Check Edge Function logs in the Supabase dashboard → Functions → discord-capture → Logs. Most likely cause: OpenRouter key missing credits or incorrect.

### Docker warning on every supabase command
Harmless. The CLI checks for Docker even when deploying to hosted infrastructure. Ignore it.

---

## What's Next

Once your HOLOCRON is running:

1. **Memory Migration** — ask your AI to extract everything it already knows about you and capture it into the HOLOCRON. Every future session starts with context.

2. **ChatGPT/Gemini import** — export your conversation history from both platforms. The OB1 repo includes import recipes for ChatGPT exports.

3. **Notion/Obsidian migration** — the OB1 companion prompts cover migrating existing second brain content.

4. **Tiered architecture** — as your HOLOCRON grows, consider a separate Supabase project for sensitive content buckets.

---

## Credits

- **OB1 Open Brain** by Nate B. Jones — https://github.com/NateBJones-Projects/OB1
- Discord integration, CTI workflow, and security-specific adaptations by **Belouve** / DC608
- Built during a single session with Claude Desktop as the primary AI interface

---

*This guide is a living document. Updates will follow as the build evolves.*  
*Questions or contributions — find Belouve in the BHIS Discord or DC608.*
