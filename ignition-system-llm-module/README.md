# Ignition Custom Modules

> Work-In-Progress modules - MVP functioanlity is completed; adding more functionalities and pending some testing.

## Table of Contents

1. [Modules Overview](#modules-overview)
2. [Using `system.llm.*` in Ignition](#using-systemllm-in-ignition)
   - [Configure a Provider](#1-configure-a-provider)
   - [Complete (single-turn)](#2-complete-single-turn)
   - [Chat (multi-turn)](#3-chat-multi-turn)
   - [Classify](#4-classify)
   - [Summarize](#5-summarize)
   - [Embed](#6-embed)
   - [Usage Stats](#7-usage-stats)
3. [Usage by Script Scope](#usage-by-script-scope)
   - [How Scopes Work](#how-scopes-work)
   - [Script Console (Designer)](#script-console-designer)
   - [Gateway Startup Script](#gateway-startup-script)
   - [Gateway Timer Script](#gateway-timer-script)
   - [Tag Event Script](#tag-event-script)
   - [Perspective — Action Button](#perspective--action-button)
   - [Perspective — Session Startup Script](#perspective--session-startup-script)
   - [Vision — Component Event Script](#vision--component-event-script)
   - [WebDev Endpoint Handler](#webdev-endpoint-handler)
   - [Alarm Notification Pipeline](#alarm-notification-pipeline)
4. [Real-World Examples](#real-world-examples)


---

## Modules Overview

| Module | Artifact | Scope | Ignition Min |
|--------|----------|-------|--------------|
| **LLM Scripting** | `Ignition-LLM-Scripting-Module.modl` | Gateway | 8.1.0 |

Modules are **free modules** (no Ignition licence tier required), work in trial mode, and are unsigned for development (see [Signing](#module-signing) for production).

---

## Using `system.llm.*` in Ignition

After the module is installed the `system.llm` namespace is available in **every** Ignition script context — Gateway, Designer, Vision client, Perspective session, Tag Event Scripts, etc.

### 1. Configure a Provider

Call `system.llm.configureProvider()` once — typically in a **Gateway Startup Script** or from the Script Console.

```python
# ── Ollama (local, free) ──────────────────────────────────────────────────────
system.llm.configureProvider(
    name         = "local-ollama",
    providerType = "ollama",              # "ollama" | "openai" | "azure-openai"
    endpoint     = "http://localhost:11434",
    apiKey       = "",                    # not used by Ollama
    defaultModel = "tinyllama",
    timeoutSeconds = 60
)

# ── OpenAI ────────────────────────────────────────────────────────────────────
system.llm.configureProvider(
    name         = "openai-gpt4o",
    providerType = "openai",
    endpoint     = "https://api.openai.com",
    apiKey       = "sk-...",
    defaultModel = "gpt-4o",
    timeoutSeconds = 30
)

# ── Azure OpenAI ──────────────────────────────────────────────────────────────
system.llm.configureProvider(
    name         = "azure-prod",
    providerType = "azure-openai",
    endpoint     = "https://MY-RESOURCE.openai.azure.com/openai/deployments/MY-DEPLOYMENT",
    apiKey       = "YOUR_AZURE_KEY",
    defaultModel = "gpt-4o",
    timeoutSeconds = 30
)
```

> **Security**: API keys are encrypted with AES-256-GCM (key derived from the gateway's installation UUID) before being stored in SQLite. They never appear in logs or exports.

---

### 2. Complete (single-turn)

The most common function — send a prompt, get a text reply.

```python
# Basic usage — uses the default provider
reply = system.llm.complete("Summarise the Ignition alarm history for reactor R-101.")
print(reply)

# Specify provider and model explicitly
reply = system.llm.complete(
    prompt      = "Is temperature 95°C within the safe operating range of 0–90°C?",
    model       = "gpt-4o-mini",
    provider    = "openai-gpt4o",
    maxTokens   = 100,
    temperature = 0.0               # 0.0 = deterministic, good for yes/no
)
print(reply)   # → "No, 95°C exceeds the maximum safe operating temperature of 90°C."
```

**Use in a Tag Event Script** (fires when a tag changes):

```python
def valueChanged(tag, tagPath, previousValue, currentValue, initialChange, missedEvents):
    if not initialChange and currentValue.value > 90:
        msg = system.llm.complete(
            "Temperature is {0:.1f}°C. Write a 1-sentence maintenance alert.".format(
                currentValue.value
            ),
            maxTokens = 60,
            temperature = 0.3
        )
        system.util.sendMessage("AlarmModule", "newAlert", {"message": msg})
```

---

### 3. Chat (multi-turn)

Pass a list of `{"role": ..., "content": ...}` dicts for conversation history.

```python
history = [
    {"role": "system",    "content": "You are an industrial process expert. Be concise."},
    {"role": "user",      "content": "The reactor pressure rose from 2 bar to 4 bar in 5 minutes."},
    {"role": "assistant", "content": "That's a 100% increase. What's the normal range?"},
    {"role": "user",      "content": "Normal is 1.5–3.5 bar. Should I shut down?"},
]

reply = system.llm.chat(history, maxTokens=150)
print(reply)
# → "Yes, 4 bar is above the normal maximum of 3.5 bar. Initiate a controlled shutdown..."
```

---

### 4. Classify

Classify text into one of a fixed set of labels. Returns the winning label string.

```python
alarm_text = "Cooling water flow dropped below 20 L/min — pump may have failed"

severity = system.llm.classify(
    text   = alarm_text,
    labels = ["LOW", "MEDIUM", "HIGH", "CRITICAL"]
)
print(severity)   # → "HIGH"

category = system.llm.classify(
    text   = alarm_text,
    labels = ["MECHANICAL", "ELECTRICAL", "PROCESS", "INSTRUMENT"]
)
print(category)   # → "MECHANICAL"
```

**Use in a Gateway Timer Script** (classify new alarms and write to a UDT):

```python
def runAction(self):
    alarms = system.alarm.queryStatus(priority=["High", "Critical"])
    for a in alarms:
        label = system.llm.classify(
            a.displayPath,
            ["PUMP", "VALVE", "SENSOR", "HEATER", "OTHER"]
        )
        system.tag.writeBlocking(
            ["[default]Alarms/" + a.id + "/Category"],
            [label]
        )
```

---

### 5. Summarize

Condense a block of text to a target word count.

```python
# Summarise the last 24 hours of process notes
notes = system.db.runScalarQuery(
    "SELECT GROUP_CONCAT(note, ' | ') FROM shift_notes WHERE ts > NOW() - INTERVAL 1 DAY"
)
summary = system.llm.summarize(notes, maxWords=80)
print(summary)
```

---

### 6. Embed

Get a vector embedding for semantic search or similarity scoring.

```python
vec = system.llm.embed("cooling water pump cavitation noise", model="text-embedding-3-small")
# Returns a Python list of floats, e.g. [-0.012, 0.034, ...]
print(len(vec))   # e.g. 1536 for OpenAI text-embedding-3-small

# Example: cosine similarity between two alarm descriptions
import math

def cosine_similarity(a, b):
    dot   = sum(x*y for x,y in zip(a, b))
    norm_a = math.sqrt(sum(x*x for x in a))
    norm_b = math.sqrt(sum(x*x for x in b))
    return dot / (norm_a * norm_b) if norm_a and norm_b else 0.0

v1 = system.llm.embed("pump bearing overheating")
v2 = system.llm.embed("motor bearing temperature high")
print(cosine_similarity(v1, v2))   # → ~0.92 (very similar)
```

---

### 7. Usage Stats

```python
# Total requests in the last 24 hours (all providers)
stats = system.llm.getUsage(sinceHours=24)
print(stats)
# → {totalRequests: 47, successCount: 45, failureCount: 2, ...}

# Filter by specific provider
stats = system.llm.getUsage(provider="openai-gpt4o", sinceHours=1)

# List all configured providers
providers = system.llm.getProviders()
print(list(providers))   # → ['local-ollama', 'openai-gpt4o']

# Get / change default
print(system.llm.getDefaultProvider())   # → 'local-ollama'
system.llm.setDefaultProvider("openai-gpt4o")
```

---

## Usage by Script Scope

### How Scopes Work

`system.llm.*` is registered on the **Gateway** via `addScriptModule`. This means:

| Scope | Where it runs | How it calls the module |
|---|---|---|
| **Gateway Startup / Shutdown Script** | Gateway JVM | Direct — same process |
| **Gateway Timer Script** | Gateway JVM | Direct — same process |
| **Tag Event Script** | Gateway JVM | Direct — same process |
| **Alarm Pipeline Script** | Gateway JVM | Direct — same process |
| **WebDev `doGet` / `doPost`** | Gateway JVM | Direct — same process |
| **Perspective Action / Event Script** | Gateway JVM | Direct — Perspective runs server-side |
| **Designer Script Console** | Gateway JVM | Direct — Script Console executes on the gateway |
| **Vision Client Event Script** | Client JVM | RPC proxy — call is forwarded to the gateway transparently |

> **Latency note for Vision clients:** Every `system.llm.*` call from a Vision client triggers an RPC round-trip to the gateway before the actual LLM HTTP request. For long-running completions (>5 s), run the call from a **Gateway Timer Script** and write results to a tag, then read the tag from the Vision client.

---

### Script Console (Designer)

The **Script Console** (`Tools → Script Console` in the Designer) executes on the gateway. It is the fastest way to test provider config or try any `system.llm.*` function interactively — no project save or restart required.

```python
# ── Quick connectivity test ────────────────────────────────────────────────────
# Paste into Script Console and click Run

print(system.llm.getProviders())          # list all configured providers
print(system.llm.getDefaultProvider())    # which provider is used when none specified

# Test a single completion
reply = system.llm.complete("Say exactly: IGNITION LLM MODULE WORKING")
print(reply)
# → IGNITION LLM MODULE WORKING

# Test classify
label = system.llm.classify(
    "Cooling water pump flow dropped to 8 L/min",
    ["CRITICAL", "HIGH", "MEDIUM", "LOW"]
)
print("Severity:", label)

# Check usage stats (requests in the last hour)
stats = system.llm.getUsage(sinceHours=1)
print(stats)
```

**Tips:**
- Call `system.llm.configureProvider(...)` directly from the Script Console to register a provider instantly — no startup script needed. The provider persists to `config.idb`.
- The Script Console shows full exception stack traces — ideal for debugging bad API keys or unreachable endpoints.

---

### Gateway Startup Script

`Config → Gateway Scripts → Startup` — runs once when the gateway starts. Use it to register providers so they are always available after every restart.

```python
# Gateway Startup Script
# Config → Gateway Scripts → Startup

def main():
    logger   = system.util.getLogger("LLM-Startup")
    existing = system.llm.getProviders()

    # Register Ollama (local, free) if not already configured
    if "local-ollama" not in existing:
        system.llm.configureProvider(
            name           = "local-ollama",
            providerType   = "ollama",
            endpoint       = "http://ollama-dev:11434",   # Docker service name
            apiKey         = "",
            defaultModel   = "tinyllama",
            timeoutSeconds = 60
        )
        logger.info("Registered provider: local-ollama")

    # Register OpenAI — read key from a Restricted tag, not hardcoded
    if "openai-gpt4o" not in existing:
        api_key = system.tag.readBlocking(["[default]Secrets/OpenAI_ApiKey"])[0].value
        if api_key:
            system.llm.configureProvider(
                name           = "openai-gpt4o",
                providerType   = "openai",
                endpoint       = "https://api.openai.com",
                apiKey         = api_key,
                defaultModel   = "gpt-4o-mini",
                timeoutSeconds = 30
            )
            logger.info("Registered provider: openai-gpt4o")

    system.llm.setDefaultProvider("local-ollama")
    logger.info("system.llm ready. Providers: " + str(system.llm.getProviders()))

main()
```

> **Security:** Store API keys in a tag with `Security Zone = Restricted` or Ignition's built-in Secrets Manager — never hardcode them in the script.

---

### Gateway Timer Script

`Config → Gateway Scripts → Timer` — runs on a fixed schedule entirely on the gateway. Use for background AI tasks that should not block any client session.

```python
# Gateway Timer Script — Fixed Rate, every 5 minutes
# Reads unacknowledged alarms, classifies them, writes a summary to a tag

def runAction(self):
    logger = system.util.getLogger("LLM-AlarmClassifier")
    try:
        results = system.alarm.queryStatus(
            source   = "[default]",
            priority = ["High", "Critical"],
            state    = ["ActiveUnacknowledged"]
        )
        alarms = results.getDataset()
        count  = system.dataset.getRowCount(alarms)

        if count == 0:
            return

        lines = []
        for i in range(count):
            path  = system.dataset.getValueAt(alarms, i, "Source")
            label = system.dataset.getValueAt(alarms, i, "Label")
            cat   = system.llm.classify(
                "{} — {}".format(path, label),
                ["PUMP_FAILURE", "SENSOR_FAULT", "PROCESS_DEVIATION", "UTILITY_ISSUE"],
                provider = "local-ollama"
            )
            lines.append("{}: {}".format(label, cat))

        # Summarise all classified alarms into one tag value
        summary = system.llm.summarize("\n".join(lines), maxWords=60, provider="local-ollama")
        system.tag.writeBlocking(
            ["[default]AI/AlarmSummary", "[default]AI/AlarmSummaryTime"],
            [summary, system.date.now()]
        )
        logger.info("Classified {} alarms".format(count))

    except Exception as e:
        logger.error("LLM timer error: " + str(e))
```

---

### Tag Event Script

`Tag Browser → right-click a tag → Edit → Event Scripts → Value Changed` — fires on the gateway each time the tag value changes.

```python
# Tag Event Script on: [default]Reactor/Temperature

def valueChanged(tag, tagPath, previousValue, currentValue, initialChange, missedEvents):
    # Ignore the first read at gateway startup
    if initialChange:
        return

    temp  = currentValue.value
    limit = 90.0

    if temp > limit:
        try:
            alert = system.llm.complete(
                "Reactor temperature is {:.1f}°C (safe limit {}°C). "
                "Write a single-sentence operator action.".format(temp, limit),
                maxTokens   = 60,
                temperature = 0.0,
                provider    = "local-ollama"
            )
            system.tag.writeBlocking(["[default]AI/ReactorAlert"], [alert])
        except Exception as e:
            system.util.getLogger("LLM-TagEvent").warn("LLM call failed: " + str(e))
```

> **Throttle tip:** Guard against rapid oscillation — add `if abs(currentValue.value - previousValue.value) < 1.0: return` to skip LLM calls when the value barely changed.

---

### Perspective — Action Button

Perspective scripts run **server-side on the gateway**, so `system.llm.*` is called directly with no RPC overhead.

In the Designer: select a Button component → `Events → onActionPerformed → Script`.

```python
# Perspective Button — onActionPerformed
# Reads a prompt from an Input component, writes the reply to a Text Area

def runAction(self, event):
    logger = system.util.getLogger("LLM-Perspective")
    prompt = self.view.getChild("promptInput").props.text

    if not prompt or not prompt.strip():
        self.view.getChild("responseArea").props.text = "Please enter a prompt."
        return

    self.view.getChild("responseArea").props.text = "Thinking..."
    try:
        reply = system.llm.complete(
            prompt.strip(),
            maxTokens   = 300,
            temperature = 0.7,
            provider    = "local-ollama"
        )
        self.view.getChild("responseArea").props.text = reply
    except Exception as e:
        logger.error("LLM error: " + str(e))
        self.view.getChild("responseArea").props.text = "Error: " + str(e)
```

**Multi-turn chat with session-level history (chat mode):**

```python
# Perspective Button — onActionPerformed (multi-turn chat)
# History is stored in a session custom property of type List named 'chatHistory'

def runAction(self, event):
    prompt  = self.view.getChild("promptInput").props.text.strip()
    if not prompt:
        return

    history = list(self.session.custom.chatHistory or [])
    history.append({"role": "user", "content": prompt})

    try:
        reply = system.llm.chat(
            history,
            maxTokens   = 300,
            temperature = 0.7,
            provider    = "local-ollama"
        )
        history.append({"role": "assistant", "content": reply})
        self.session.custom.chatHistory = history
        self.view.getChild("responseArea").props.text = reply
    except Exception as e:
        self.view.getChild("responseArea").props.text = "Error: " + str(e)
```

---

### Perspective — Session Startup Script

`Project → Perspective → Session Props → onStartup` — runs once per user session when the app opens.

```python
# Perspective Session Startup Script

def onStartup(session):
    logger = system.util.getLogger("LLM-SessionStart")
    try:
        username = session.props.auth.user.userName
        greeting = system.llm.complete(
            "Write a 1-sentence shift-start safety reminder for operator {}.".format(username),
            maxTokens   = 60,
            temperature = 0.5,
            provider    = "local-ollama"
        )
        session.custom.llmGreeting  = greeting
        session.custom.chatHistory  = []   # initialise empty chat history
        logger.info("Greeting generated for: " + username)
    except Exception as e:
        session.custom.llmGreeting = "Welcome. Have a safe shift."
        logger.warn("LLM session startup failed (non-fatal): " + str(e))
```

Bind the `Text` property of any Label component to `{session.custom.llmGreeting}` to display it automatically.

---

### Vision — Component Event Script

Vision client scripts run in the **client JVM** and proxy `system.llm.*` to the gateway via RPC. The API is identical, but long LLM calls will **block the Swing Event Dispatch Thread** and freeze the UI. Always run them on a background thread.

```python
# Vision Button — actionPerformed
# Runs the LLM call on a background thread; updates UI via SwingUtilities.invokeLater

def runAction(event):
    import threading
    from javax.swing import SwingUtilities

    prompt_comp   = event.source.parent.getComponent("PromptField")
    response_comp = event.source.parent.getComponent("ResponseLabel")
    button        = event.source

    prompt = prompt_comp.text.strip()
    if not prompt:
        return

    button.enabled     = False
    response_comp.text = "Thinking..."

    def call_llm():
        try:
            reply = system.llm.complete(
                prompt,
                maxTokens   = 200,
                temperature = 0.7,
                provider    = "local-ollama"
            )
        except Exception as e:
            reply = "Error: " + str(e)
        def update():
            response_comp.text = reply
            button.enabled     = True
        SwingUtilities.invokeLater(update)

    t = threading.Thread(target=call_llm)
    t.daemon = True
    t.start()
```

> **Simpler alternative:** Use a **Gateway Timer Script** to pre-compute and write LLM results to a tag. Bind the Vision label to that tag — zero latency for the client, no threading needed.

---

### WebDev Endpoint Handler

`Project → WebDev → New Resource` — creates an HTTP endpoint inside your Ignition project. WebDev scripts run server-side, so `system.llm.*` is a direct call with no RPC.

```python
# WebDev Resource: /system/webdev/<project>/llm/complete
# Method: doPost
# Body (JSON): {"prompt": "...", "provider": "local-ollama", "maxTokens": 200}

def doPost(request, session):
    import json
    logger = system.util.getLogger("LLM-WebDev")
    try:
        body     = json.loads(request["data"])
        prompt   = body.get("prompt", "").strip()
        provider = body.get("provider", None)
        max_tok  = int(body.get("maxTokens", 200))
        temp     = float(body.get("temperature", 0.7))

        if not prompt:
            return {"json": {"error": "'prompt' is required"}, "code": 400}

        reply = system.llm.complete(prompt, maxTokens=max_tok, temperature=temp, provider=provider)
        logger.info("WebDev /llm/complete — {} chars".format(len(reply)))
        return {"json": {"reply": reply}}

    except Exception as e:
        logger.error("WebDev LLM error: " + str(e))
        return {"json": {"error": str(e)}, "code": 500}
```

**Call from cURL or an external system:**

```bash
curl -s -u admin:password \
  -X POST http://localhost:8088/system/webdev/MyProject/llm/complete \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Is 95°C safe for a reactor with a 90°C limit?", "maxTokens": 80}'
# → {"reply": "No, 95°C exceeds the safe limit of 90°C ..."}
```

**`doGet` handler — query-parameter classify API:**

```python
# WebDev doGet — /system/webdev/<project>/llm/classify
# Query: ?text=pump+bearing+noise&labels=MECHANICAL,ELECTRICAL,PROCESS

def doGet(request, session):
    text   = request["params"].get("text", "")
    raw    = request["params"].get("labels", "UNKNOWN")
    labels = [l.strip() for l in raw.split(",") if l.strip()]

    if not text or not labels:
        return {"json": {"error": "'text' and 'labels' params required"}, "code": 400}

    result = system.llm.classify(text, labels)
    return {"json": {"label": result, "text": text}}
```

---

### Alarm Notification Pipeline

`Config → Alarming → Notification Pipelines` — add a **Script Notification** block. The pipeline runs on the gateway, so `system.llm.*` is a direct call.

```python
# Alarm Notification Pipeline — Script block
# Enriches the alarm with an LLM-generated remediation hint
# before passing it to an email or SMS block downstream

def handleNotification(alarm, pipeline):
    logger = system.util.getLogger("LLM-AlarmPipeline")
    try:
        source   = str(alarm.source)
        label    = str(alarm.label)
        priority = str(alarm.priority)
        value    = str(alarm.eventValue)

        guidance = system.llm.complete(
            "Alarm: {label} (priority={priority})\n"
            "Source: {source}\nTrigger value: {value}\n\n"
            "Provide a 2-sentence operator remediation action.".format(
                label=label, priority=priority, source=source, value=value
            ),
            maxTokens   = 80,
            temperature = 0.2,
            provider    = "local-ollama"
        )
        alarm.setCustomProperty("llmGuidance", guidance)

    except Exception as e:
        # Never block the pipeline — fall back silently
        alarm.setCustomProperty("llmGuidance", "Auto-guidance unavailable.")
        logger.warn("LLM pipeline error (non-blocking): " + str(e))

    return alarm   # always return the alarm so the pipeline continues
```

**Email template using the custom property:**

```
Subject: [{Priority}] {DisplayPath}

Alarm:    {DisplayPath}
Priority: {Priority}
Value:    {EventValue}
Time:     {EventTime}

Operator Guidance (AI-generated):
{AlarmProperty:llmGuidance}
```

---

---

## Real-World Examples

### Alarm Classification Pipeline

```python
# Gateway Timer Script — runs every 5 minutes
def runAction(self):
    alarms = system.alarm.queryStatus(
        source   = "[default]",
        priority = ["High", "Critical"],
        state    = ["ActiveUnacknowledged"]
    )
    
    for alarm in alarms:
        desc = str(alarm.displayPath) + " — " + str(alarm.label)
        
        # Classify alarm type
        category = system.llm.classify(
            text   = desc,
            labels = ["PUMP_FAILURE", "SENSOR_FAULT", "PROCESS_DEVIATION", "UTILITY_ISSUE"]
        )
        
        # Generate operator guidance
        guidance = system.llm.complete(
            "Alarm: {0}\nCategory: {1}\nProvide a 2-sentence operator action in plain English.".format(
                desc, category
            ),
            maxTokens   = 80,
            temperature = 0.2
        )
        
        system.tag.writeBlocking(
            ["[default]Alarms/AutoCategory", "[default]Alarms/Guidance"],
            [category, guidance]
        )
```

---

---

## Dev Environment Setup

### Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| **JDK 21** | 21.x | Build runtime (Kotlin 2.0 requires JDK 11+, but JDK 21 is recommended) |
| **Docker Desktop** | 4.x+ | Local Ignition gateway + Ollama containers |
| **Git** | any | Version control |
| Python 3 | 3.9+ | Module acceptance script (`infra/scripts/accept-module.sh`) |

> ⚠️ **JDK 25 is not compatible** with Kotlin 2.0 (version string parse issue). Use JDK 21.

---

---

### Project Structure

```
ignition-custom-modules/
├── modules/
│   ├── system-llm/                     # system.llm.* scripting module
│   │   ├── build.gradle.kts
│   │   └── src/
│   │       ├── main/kotlin/com/customignition/llm/
│   │       │   ├── common/             # LlmRequest, LlmResponse, LlmConfig, LlmException
│   │       │   ├── crypto/             # AesGcmCipher — AES-256-GCM key encryption
│   │       │   ├── db/                 # LlmAuditDb — SQLite audit log
│   │       │   ├── providers/          # LlmProviderRegistry, OpenAiClient, AzureOpenAiClient, OllamaClient
│   │       │   ├── script/             # LlmScriptModule — all system.llm.* functions
│   │       │   └── gateway/            # GatewayHook, LlmConfigServlet
│   │       └── test/                   # 34 unit tests
│
├── gradle/
│   ├── libs.versions.toml              # Version catalog (all dependency versions)
│   └── wrapper/                        # Gradle wrapper (gradlew / gradlew.bat)
│
├── infra/
│   ├── docker/
│   │   └── docker-compose.yml          # ignition-dev + ollama-dev containers
│   └── scripts/
│       ├── accept-module.sh            # Insert EULAS acceptance into gateway DB
│       ├── deploy-module.sh            # Upload .modl via gateway REST API
│       ├── wait-for-gateway.sh         # Poll until gateway is healthy
│       └── reset-gateway.sh            # Wipe and restart the gateway volume
│
├── .github/workflows/
│   └── ci.yml                          # GitHub Actions: test → build → integration test
│
├── build.gradle.kts                    # Root build — shared Kotlin/test config
├── settings.gradle.kts                 # Includes modules/system-llm, modules/file-tag-provider
├── gradle.properties                   # Gradle performance flags, skipModlSigning
└── TEST-RESULTS.md                     # Integration test results & evidence
```

---

## Module Reference

### `system.llm.*` API

| Function | Signature | Returns |
|----------|-----------|---------|
| `configureProvider` | `(name, providerType, endpoint, apiKey, defaultModel, timeoutSeconds=30)` | `None` |
| `complete` | `(prompt, model=None, provider=None, maxTokens=500, temperature=0.7)` | `str` |
| `chat` | `(messages, model=None, provider=None, maxTokens=500, temperature=0.7)` | `str` |
| `embed` | `(text, model=None, provider=None)` | `list[float]` |
| `classify` | `(text, labels, provider=None)` | `str` |
| `summarize` | `(text, maxWords=100, provider=None)` | `str` |
| `getProviders` | `()` | `list[str]` |
| `getDefaultProvider` | `()` | `str` |
| `setDefaultProvider` | `(name)` | `None` |
| `getUsage` | `(provider=None, sinceHours=24)` | `dict` |

**Provider types:** `"openai"` · `"azure-openai"` · `"ollama"`

---
