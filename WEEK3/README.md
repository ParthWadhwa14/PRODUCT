---
track: "product"
week: 3
---

# Week 3 Curriculum

**Overview**
This week takes you from the absolute basics of what Generative AI is, to building the foundational architecture for your final voice product. We will use visual analogies to understand APIs, and then step-by-step, we will build a local server that connects to Google’s Gemini 2.5 Flash Native Audio model (API model string: `gemini-2.5-flash-native-audio-preview-12-2025`). 

> Everything in this curriculum runs on the Gemini API **free tier**.
> 1. Go to [Google AI Studio](https://aistudio.google.com/) and sign in with any Google account.
> 2. Click **Get API key** → **Create API key**. No credit card or billing setup is required.
> 3. Save the key in your `.env` file as `GEMINI_API_KEY=...` (Module 3 covers this in detail).
>
> The free tier covers both `gemini-2.5-flash` (used in Module 1) and the Live API native audio model (used in Modules 4–5). It has rate limits (requests per minute/day), which are more than enough for this assignment — if you hit a limit, wait a minute and retry. **Do not enable billing** on your project; you don't need it for this Curriculum.

---

## Module 1: Ground Zero – Demystifying GenAI & APIs
Before touching any code, we must understand the building blocks. 

| Topic | Learning Goal | Required Resources |
| :--- | :--- | :--- |
| **What is Generative AI?** | Learn how Large Language Models (LLMs) are essentially advanced autocomplete engines rather than conscious entities. | *Video:* [Predictive vs Generative AI (IBM Technology)](https://www.youtube.com/watch?v=phOhGqpXss4) |
| **What is an AI Agent?** | Understand the difference between a chatbot that just talks, and an "Agent" that has tools to actually *do* things. | *Video:* [What is an AI Agent? (IBM Technology)](https://www.youtube.com/watch?v=F8NKVhkZZWI) |
| **The Restaurant Analogy (APIs)** | Demystify APIs. The frontend is the customer, the database is the kitchen, and the API is the waiter. | *Video:* [REST API Crash Course (Traversy Media)](https://www.youtube.com/watch?v=-MTSQjw5DrM)<br>*Video:* [What is a REST API (IBM)](https://www.youtube.com/watch?v=lsMQRaeKNDk) |
| **The Credit Card Rule** | Learn why API keys are sensitive financial credentials and why they can **never** be exposed in frontend code. | *Article:* [What Is an API Key? (IBM)](https://www.ibm.com/think/topics/api-key) |

> 🛠️ **Module 1 Mini-Task: The REST Baseline**
> Before building a real-time WebSocket, you need to feel what a standard REST API call feels like. 
> 
> **How to run this:** Create a file named `test_rest.py`, paste the code below, run `pip install requests python-dotenv`, and execute it in your terminal (`python test_rest.py`).
> ```python
> import os
> import requests
> from dotenv import load_dotenv
> 
> # 1. Load the hidden API key from your .env file
> load_dotenv()
> api_key = os.getenv("GEMINI_API_KEY")
> 
> # 2. Define where we are sending the request (The REST Endpoint)
> url = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent"
> 
> # Note: we send the key in a header, NOT in the URL.
> # URLs end up in logs and browser history — headers are safer. (Remember the Credit Card Rule!)
> headers = {"x-goog-api-key": api_key}
> 
> # 3. Define the JSON payload (Our text prompt)
> payload = {
>     "contents": [{
>         "parts": [{"text": "Explain the difference between a REST API and a WebSocket in one sentence."}]
>     }]
> }
> 
> # 4. Send the request, wait for the response, and store it
> print("Sending request to Google... waiting...")
> response = requests.post(url, headers=headers, json=payload)
> 
> # If something went wrong (bad key, wrong model name), show the error instead of crashing
> if response.status_code != 200:
>     print(f"Error {response.status_code}: {response.text}")
>     raise SystemExit(1)
> 
> # 5. Extract the text from the JSON response and print it
> answer = response.json()['candidates'][0]['content']['parts'][0]['text']
> print(f"\nGemini says: {answer}")
> print("\nNotice how the script has now completely stopped. This is stateless REST!")
> ```

---

## Module 2: The Voice Paradigm – WebSockets
Interacting with voice requires a completely different mental model than typing text. Here, we transition from basic APIs to the architecture of live audio.

| Topic | Learning Goal | Required Resources |
| :--- | :--- | :--- |
| **REST vs. WebSockets** | Understand why continuous voice streaming requires WebSockets (`wss://`) instead of standard REST HTTP calls. | *Video:* [WebSockets in 100 Seconds & Beyond (Fireship)](https://www.youtube.com/watch?v=1BfCnjr_Vjg)<br>*Article:* [WebSocket Basics (Postman)](https://www.postman.com/templates/collections/websocket-basics/) |
| **Gemini 2.5 Flash Live API** | Learn what makes this model special: Affective Dialog (emotional tone matching) and Barge-in (the ability to interrupt the AI). *Note: Affective Dialog requires the experimental `v1alpha` endpoint; everything else in this course uses the standard `v1beta`.* | *Doc:* [Gemini Live API Overview (Google AI)](https://ai.google.dev/gemini-api/docs/live-api) |
| **Initiating the Session** | Understand how the `BidiGenerateContentSetup` JSON message initiates the session. | *Doc:* [Get started with Gemini Live API using WebSockets](https://ai.google.dev/gemini-api/docs/live-api/get-started-websocket) |

> 🛠️ **Module 2 Mini-Task: The Open Line Test**
> Let's prove how WebSockets stay open. 
> 
> **How to run this:** Create a file named `test_ws.py`, paste the code below, run `pip install websockets`, and execute it (`python test_ws.py`).
> ```python
> import asyncio
> import websockets
> 
> async def test_websocket():
>     uri = "wss://echo.websocket.events"
>     print(f"Connecting to {uri}...")
>     
>     async with websockets.connect(uri) as websocket:
>         print("Connection open!")
>         
>         # Send 3 messages over the SAME connection — no reconnecting in between.
>         for i in range(1, 4):
>             await websocket.send(f"Hello, WebSocket! (message {i})")
>             response = await websocket.recv()
>             print(f"Server replied: {response}")
>             await asyncio.sleep(2)
>         
>         print("One connection handled 3 round-trips. This is a stateful connection —")
>         print("compare that to REST, where every request is a brand new conversation.")
> 
> if __name__ == "__main__":
>     asyncio.run(test_websocket())
> ```

---

## Module 3: The Architecture – Middleman Proxy Servers
You will set up your development environment and build the bridge between the frontend user and the Gemini AI.

| Topic | Learning Goal | Required Resources |
| :--- | :--- | :--- |
| **Setting up the Server** | Build a basic Python (FastAPI) or Node.js (Express/NestJS) server. | *Docs:* [FastAPI Official Documentation](https://fastapi.tiangolo.com/) OR<br>*Docs:* [Express.js Getting Started](https://expressjs.com/en/starter/installing.html) |
| **Protecting the Keys** | Implement `.env` variables. Understand why clients should never hold the Gemini API keys. | *Video:* [How to Hide API Keys using dotenv (Web Dev Simplified)](https://www.youtube.com/watch?v=17UVejOw3zA) |
| **Piping Binary Data** | Learn how to funnel raw audio data from the client, through your server, to Gemini, and back. | *Interactive Notebook:* [Getting Started with Gemini Live API (Google Cloud Colab)](https://colab.research.google.com/github/GoogleCloudPlatform/generative-ai/blob/main/gemini/multimodal-live-api/intro_multimodal_live_api.ipynb) |

> 🛠️ **Module 3 Mini-Task: The Local Proxy Shell**
> Build the absolute bare minimum version of your backend. 
> 
> **How to run this:** Create a file named `proxy.py`, paste the code below, run `pip install fastapi uvicorn`, and execute it (`python proxy.py`).
> ```python
> from fastapi import FastAPI, WebSocket, WebSocketDisconnect
> import uvicorn
> 
> app = FastAPI()
> 
> # 1. Standard REST Endpoint
> @app.get("/health")
> def health_check():
>     return {"status": "ok", "message": "Middleman is running!"}
> 
> # 2. WebSocket Endpoint
> @app.websocket("/ws")
> async def websocket_endpoint(websocket: WebSocket):
>     await websocket.accept()
>     print("Frontend client connected!")
>     
>     try:
>         while True:
>             # Listen for messages from the frontend and echo them back
>             data = await websocket.receive_text()
>             print(f"Received from frontend: {data}")
>             await websocket.send_text(f"Proxy echo: {data}")
>     except WebSocketDisconnect:
>         print("Frontend client disconnected.")
> 
> if __name__ == "__main__":
>     print("Starting proxy server on http://localhost:8000")
>     uvicorn.run(app, host="0.0.0.0", port=8000)
> ```

---

## Module 4: Voice & Action – Audio Streaming & Tool Calling
This is where the magic happens. You will connect the microphone and give your AI the ability to trigger real-world code.

| Topic | Learning Goal | Required Resources |
| :--- | :--- | :--- |
| **Frontend Audio Wiring** | We provide a pre-written hook (`useAudioStreamer()`) that handles mic permissions, 16kHz downsampling, and 24kHz audio queue playback. You will wire this to the frontend WebSocket so you can push-to-talk. | *Code Base:* [Reference the PRODUCT Repo](https://github.com/ParthWadhwa14/PRODUCT/tree/main) |
| **Function Calling Concepts** | How do you give an AI abilities? You give it a "Tool". Learn how the AI outputs a JSON message requesting a function, and your server executes it. | *Video:* [Introduction to Function Calling with Gemini](https://www.youtube.com/watch?v=lJYj1UKbhAg) |
| **The Guided Tool Call** | Walk through adding a single, predefined tool (like `get_weather_in_city` or `get_current_time`). Define the JSON schema in the setup message. | *Docs:* [Function Calling with the Gemini API](https://ai.google.dev/gemini-api/docs/function-calling) |
| **Catch & Respond** | Catch the `toolCall` message on your backend, execute the dummy function, and send back the `BidiGenerateContentToolResponse` so the model can speak the result. | *Docs:* [Live API – WebSockets API Reference (tool use messages)](https://ai.google.dev/api/live) |

> 🛠️ **Module 4 Mini-Task: The Schema Blueprint**
> Before the model can use a tool, it needs to know the tool exists. 
> 
> **How to run this:** Create a file named `schema.py`, paste the code below, and execute it (`python schema.py`). It will generate and format the JSON dictionary you need for your final setup message.
> ```python
> import json
> 
> setup_message = {
>     "setup": {
>         # REQUIRED: the setup message must name the model.
>         # Without this field, Gemini will reject the session.
>         "model": "models/gemini-2.5-flash-native-audio-preview-12-2025",
>         "generationConfig": {
>             "responseModalities": ["AUDIO"]  # We want the model to talk back
>         },
>         "systemInstruction": {
>             "parts": [{"text": "You are a helpful smart echo box."}]
>         },
>         "tools": [{
>             "functionDeclarations": [{
>                 "name": "get_current_time",
>                 "description": "Returns the current local time.",
>                 "parameters": {
>                     "type": "OBJECT",
>                     "properties": {}, # No arguments needed for time
>                     "required": []
>                 }
>             }]
>         }]
>     }
> }
> 
> print("Use this JSON as the FIRST message you send over your final WebSocket:")
> print(json.dumps(setup_message, indent=2))
> ```

---

## Module 5: The Assignment – "The Smart Echo Box"

Now, the training wheels come off. You must synthesize everything you learned in Modules 2, 3, and 4 to build the complete architecture.

**Goal:** Have a working, local voice client communicating with Gemini through your own securely configured backend proxy server.

> 🛠️ **Assignment Starter Code: `server.py`**
> Copy this minimal scaffold to start your assignment. It sets up the API keys and endpoints but leaves the core orchestration logic to you. 
> 
> ```python
> import os
> import json
> import asyncio
> import websockets
> from fastapi import FastAPI, WebSocket, WebSocketDisconnect
> from dotenv import load_dotenv
> 
> # Load environment variables securely
> load_dotenv()
> GEMINI_API_KEY = os.getenv("GEMINI_API_KEY")
> 
> MODEL = "models/gemini-2.5-flash-native-audio-preview-12-2025"
> 
> # v1beta is the standard Live API endpoint.
> # (Swap "v1beta" for "v1alpha" ONLY if you enable experimental features
> #  like Affective Dialog or Proactive Audio.)
> GEMINI_WS_URL = f"wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent?key={GEMINI_API_KEY}"
> 
> app = FastAPI()
> 
> @app.websocket("/ws")
> async def proxy_endpoint(client_ws: WebSocket):
>     await client_ws.accept()
>     print("Frontend connected.")
> 
>     # Connect to Gemini
>     async with websockets.connect(GEMINI_WS_URL) as gemini_ws:
>         print("Connected to Gemini.")
> 
>         # TODO 1: Send the BidiGenerateContentSetup message (from your schema.py)
>         #         as the FIRST message — it must include the "model" field.
> 
>         # TODO 2: Create a task/loop to receive audio from frontend (client_ws) and send to Gemini (gemini_ws)
> 
>         # TODO 3: Create a task/loop to receive responses from Gemini
>         # - If it's a normal message, forward to frontend
>         #   (Careful: Gemini sends audio as BINARY frames — forward with send_bytes, not send_text)
>         # - If it's a "toolCall", intercept it, run your python function, and send the response back to Gemini
> 
>         # TODO 4: Run both tasks concurrently (Hint: look into asyncio.gather)
> ```

**Requirements:**
1. **The Architecture:** Your backend server must establish a WebSocket connection with the frontend UI *and* a separate WebSocket connection with the Gemini Live API simultaneously. 
2. **Audio Data Flow:** You must wire the provided `useAudioStreamer()` hook so that when a user speaks, binary data flows from the Client → Backend Proxy → Gemini. Gemini's audio response must flow back through the exact same pipe to be played on the speakers.
3. **The Guided Tool:** The bot must successfully invoke the guided tool (e.g., answering "What time is it right now?"). Your backend must intercept the `toolCall`, execute the Python/Node logic, and pipe the `BidiGenerateContentToolResponse` back to the model.
4. **Security:** The Gemini API key must exist strictly in the backend `.env` file. A key found in the frontend source code is an auto-fail.

**Deliverable:**
* A link to your working, committed GitHub repository.
* A completed `SUBMISSION.md` file (300-600 words) in your repo explaining your data flow architecture, how you handled the dual-WebSocket connections, and any challenges you faced.
