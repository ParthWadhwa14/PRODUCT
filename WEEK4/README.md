---
track: "product"
week: 4
---

# Week 4 Curriculum: Native Polish & Completion Sprint

**Overview**
With the Week 3 boilerplate done, the training wheels come off. This week is about making the agent feel *human* — handling interruptions gracefully, giving it a real personality — and then building out an original product. By the end of the sprint, every participant has a working, tool-capable voice agent on their machine, and spends the rest of the time actually being creative.

> 🛑 **Constraint Checklist:** > Everything still runs on the Gemini API **free tier** key you created in Week 3. The custom tools you build this week should also use free resources — free public APIs or a local SQLite database. **Do not enable billing on your Google Cloud project.**

---

## Module 1: Barge-in – Handling Interruptions

Voice bots feel robotic if you can't interrupt them. A human stops talking the instant you cut in; your agent must do the same. Gemini detects the interruption for you (via its built-in Voice Activity Detection) — your job is to *react* to the signal correctly on both ends of your proxy.

| Topic | Learning Goal | Required Resources |
| :--- | :--- | :--- |
| **The `interrupted` Flag** | When the user starts speaking over the model, Gemini stops generating and sends a `serverContent` message with `interrupted: true`. Learn to catch this in your backend receive loop. | *Doc:* [Live API Capabilities – Interruptions](https://ai.google.dev/gemini-api/docs/live-api/capabilities) |
| **Flushing the Playback Queue** | Your frontend buffers audio chunks ahead of playback. If you don't flush that queue on interruption, the bot "keeps talking" from stale audio. Learn why the flush must be instant. | *Doc:* [Troubleshooting Live API (Google Cloud)](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/live-api/troubleshooting) |
| **Voice Activity Detection (VAD)** | Understand the automatic VAD that powers barge-in: sensitivity settings, and why you must keep streaming mic audio continuously (no audio in = no interrupt signal out). | *Doc:* [Live API Capabilities – VAD Configuration](https://ai.google.dev/gemini-api/docs/live-api/capabilities) |
| **Reference Patterns** | See Google's own reference code for the receive loop: audio parts, transcriptions, and the interruption check. | *Code Reference:* [Gemini Live API Dev Patterns](https://github.com/google-gemini/gemini-skills/blob/main/skills/gemini-live-api-dev/SKILL.md) |

> 🛠️ **Module 1 Mini-Task: The Interruption Drill**
> Before wiring this into your live proxy, prove the handler logic works against a fake message stream.
>
> **How to run this:** Create `barge_in.py` and run it (`python barge_in.py`).
> ```python
> fake_gemini_stream = [
>     {"serverContent": {"modelTurn": {"parts": [{"inlineData": {"data": "audio-chunk-1"}}]}}},
>     {"serverContent": {"modelTurn": {"parts": [{"inlineData": {"data": "audio-chunk-2"}}]}}},
>     {"serverContent": {"interrupted": True}},   # <-- The user just barged in!
>     {"serverContent": {"modelTurn": {"parts": [{"inlineData": {"data": "new-response-chunk"}}]}}},
> ]
> 
> playback_queue = []
> 
> def handle_gemini_message(message: dict):
>     content = message.get("serverContent", {})
> 
>     # THE BARGE-IN CHECK: always do this FIRST, before touching audio.
>     if content.get("interrupted"):
>         playback_queue.clear()
>         print(">>> INTERRUPTED! Backend Queue flushed.")
>         print(">>> (In your real app: send {'type': 'flush'} to the frontend NOW)")
>         return
> 
>     # Otherwise, queue up audio chunks as normal
>     for part in content.get("modelTurn", {}).get("parts", []):
>         if "inlineData" in part:
>             playback_queue.append(part["inlineData"]["data"])
>             print(f"Queued audio chunk. Queue size: {len(playback_queue)}")
> 
> for msg in fake_gemini_stream:
>     handle_gemini_message(msg)
> ```
> **The Frontend Catch:** When your backend sends the flush signal, your frontend JavaScript must immediately halt the active `AudioContext`. 
> ```javascript
> // Example Frontend Logic
> socket.onmessage = (event) => {
>     const data = JSON.parse(event.data);
>     if (data.type === "flush") {
>         playbackQueue = []; // Empty the buffer
>         if (currentAudioSource) currentAudioSource.stop(); // Stop current sound
>     }
> };
> ```

---

## Module 2: Personality – System Prompts & Affective Dialog

A voice-native model isn't prompted like a chatbot. Word choice, sentence length, and tone all come out of the speakers, so the system prompt is effectively a *casting brief*, not just instructions.

| Topic | Learning Goal | Required Resources |
| :--- | :--- | :--- |
| **Prompting for Voice** | Learn the anatomy of a production voice prompt: identity, tone, operational boundary, guardrails, and why persona directly shapes pacing and word choice in speech. | *Guide:* [Voice AI Prompting Guide (Vapi)](https://docs.vapi.ai/prompting-guide)<br>*Video:* [Prompt Engineering: A Beginner's Guide (IBM Technology)](https://www.youtube.com/watch?v=OeVzbJ6XwYs) |
| **Persona Engineering** | How a defined role acts as a filter on everything the model says — and how to keep the persona stable across a long conversation. | *Article:* [How To Define an AI Agent Persona (The New Stack)](https://thenewstack.io/how-to-define-an-ai-agent-persona-by-tweaking-llm-prompts/) |
| **Affective Dialog (Optional)** | Gemini's native audio can respond to *how* you say something, not just what you say. | *Blog:* [Advanced Audio Dialog with Gemini 2.5 (Google DeepMind)](https://blog.google/technology/google-deepmind/gemini-2-5-native-audio/) |

> 🛠️ **Module 2 Mini-Task: The Persona Transplant**
> Take your boring Week 3 "smart echo box" and give it a soul.
>
> **How to run this:** Replace the `systemInstruction` in your setup message with a structured persona prompt.
> ```python
> PERSONA_PROMPT = """
> # IDENTITY
> You are 'Asha', a warm but no-nonsense campus canteen assistant at an engineering college.
> You speak in short, punchy sentences — this is a VOICE conversation, never read out lists or markdown.
> 
> # OPERATIONAL BOUNDARY
> You ONLY help with: today's menu, placing an order, and order status.
> If asked about anything else, redirect in one sentence and offer the menu.
> 
> # TONE
> Friendly, quick, slightly playful. Match the user's energy —
> if they sound rushed, be brief; if they're chatty, chat back.
> 
> # GUARDRAILS
> Never invent menu items. If a tool call fails, say so plainly and suggest retrying.
> """
> 
> setup_message = {
>     "setup": {
>         # Note: Preview models are subject to updates. Always check the docs 
>         # for the latest stable model string if you deploy this to production later.
>         "model": "models/gemini-2.5-flash-native-audio-preview-12-2025",
>         "generationConfig": {"responseModalities": ["AUDIO"]},
>         "systemInstruction": {"parts": [{"text": PERSONA_PROMPT}]},
>         "tools": [ ... ],  # your tools from Module 3
>     }
> }
> ```

---

## Module 3: Real Tools – From Guided Tool to Your Product

In Week 3 your bot could tell the time. Time to swap that toy out for the actual APIs and databases your hackathon idea needs. 

| Topic | Learning Goal | Required Resources |
| :--- | :--- | :--- |
| **Designing Tool Schemas** | Good descriptions = good tool selection. Learn how the model decides *when* to call your function. | *Doc:* [Function Calling with the Gemini API](https://ai.google.dev/gemini-api/docs/function-calling) |
| **Tool Formatting Rules** | `toolResponse` payloads must perfectly match the schema expected by the Live API, or the WebSocket will instantly terminate. | *Doc:* [Live API – WebSockets API Reference](https://ai.google.dev/api/live) |
| **Free Data Sources** | Browse free, no-auth public APIs for your product idea. | *Directory:* [Public APIs Collection (GitHub)](https://github.com/public-apis/public-apis) |
| **Local Databases** | For stateful products, use `sqlite3`. *Tip: SQLite locks the database during writes. In a fast WebSocket loop, keep your DB queries extremely brief to avoid "database is locked" errors.* | *Doc:* [Python `sqlite3` Module](https://docs.python.org/3/library/sqlite3.html) |

> 🛠️ **Module 3 Mini-Task: Your First Real Tool**
> Build a tool that hits a real external API — gracefully handling failures so the bot doesn't crash.
>
> **How to run this:** Create `custom_tool.py` and execute it (`python custom_tool.py`).
> ```python
> import json
> import requests
> 
> # STEP 1: The DECLARATION
> weather_tool = {
>     "name": "get_weather",
>     "description": "Gets the current temperature for a location given its coordinates.",
>     "parameters": {
>         "type": "OBJECT",
>         "properties": {
>             "latitude":  {"type": "NUMBER"},
>             "longitude": {"type": "NUMBER"}
>         },
>         "required": ["latitude", "longitude"]
>     }
> }
> 
> # STEP 2: The IMPLEMENTATION (With Error Handling!)
> def get_weather(latitude: float, longitude: float) -> dict:
>     url = "[https://api.open-meteo.com/v1/forecast](https://api.open-meteo.com/v1/forecast)"
>     params = {"latitude": latitude, "longitude": longitude, "current": "temperature_2m"}
>     try:
>         response = requests.get(url, params=params, timeout=10)
>         response.raise_for_status()
>         return {"temperature_celsius": response.json()["current"]["temperature_2m"]}
>     except requests.exceptions.RequestException as e:
>         # Return the error so the AI can smoothly inform the user, rather than crashing
>         return {"error": "Weather service currently unavailable."}
> 
> # STEP 3: The TOOL CALL Simulation
> fake_tool_call = {
>     "toolCall": {
>         "functionCalls": [
>             # Note: Gemini may send multiple function calls in parallel.
>             # In your real app, loop through this array!
>             {"id": "call-001", "name": "get_weather", "args": {"latitude": 28.61, "longitude": 77.21}}
>         ]
>     }
> }
> 
> call = fake_tool_call["toolCall"]["functionCalls"][0]
> result = get_weather(**call["args"])
> print(f"Executed {call['name']} -> {result}")
> 
> # STEP 4: The RESPONSE
> tool_response = {
>     "toolResponse": {
>         "functionResponses": [
>             {"id": call["id"], "name": call["name"], "response": {"output": result}}
>         ]
>     }
> }
> print("\nSend this perfectly formatted JSON back over the WebSocket:")
> print(json.dumps(tool_response, indent=2))
> ```

---

## Module 4: The Assignment – "The MVP Sprint"

**Goal:** Fork the Week 3 boilerplate and build an original, interactive voice product.

**Requirements:**
1. **A Custom Persona:** A structured system prompt giving the agent a specific persona and an operational boundary. "You are a helpful assistant" is an auto-fail. *(Note: The standard `v1beta` endpoint is fine for this; Affective Dialog via `v1alpha` is entirely optional).*
2. **At Least Two Original Custom Tools** executing real logic (hitting a real external API, querying a database, writing to a file, etc.). Dummy functions that return hardcoded strings do not count.
3. **Fully Functioning Barge-in:** Interrupting the agent mid-sentence must stop playback instantly on the frontend and let the conversation flow naturally. We will test this in your demo.
4. **Security Check:** Your API key must live strictly in the backend `.env` file. Any API key found exposed in your frontend source code will result in an **auto-fail**.

**Deliverable:**
* A link to your working, committed GitHub repository.
* A **live demo** or an **uncut video walkthrough** of a seamless conversation explicitly showing:
  1. At least one successful execution of *each* of your custom tools.
  2. At least one successful barge-in/interruption mid-response.
