# Real-time Transcription Parsing from Gemini Live API

## How it works

This project uses the Gemini Live API (via LiveKit Agents) for real-time voice conversations. Gemini streams audio responses back to the browser, and the LiveKit SDK provides real-time transcriptions of that audio via `output_audio_transcription` (enabled by default in the LiveKit Gemini plugin).

The demo includes **two modes** for detecting colors from Gemini's speech and changing the UI background — you can toggle between them in the UI to compare latency and behavior.

## Two Modes

### Mode 1: Transcription Parsing (fast)

Parses color names directly from the transcription text on the frontend. No round-trip to the agent.

```
User speaks → LiveKit Cloud → Python Agent → Gemini Live API
                                                    ↓
                                              Audio + Transcription
                                                    ↓
User hears ← LiveKit Cloud ← Python Agent ← Audio stream
Browser UI  ← LiveKit SDK  ← Transcription events (parsed client-side)
```

**Key code** (`web/public/simple.html`):

```javascript
const CSS_COLORS = ['red', 'orange', 'yellow', 'green', 'blue', 'purple', ...];

room.on(LivekitClient.RoomEvent.TranscriptionReceived, (segs, participant) => {
  const isAgent = participant?.isAgent || false;

  for (const seg of segs) {
    if (isAgent) {
      const words = seg.text.toLowerCase().split(/\s+/);
      for (const word of words) {
        const clean = word.replace(/[^a-z]/g, '');
        if (CSS_COLORS.includes(clean)) {
          document.body.style.backgroundColor = clean;
          break;
        }
      }
    }
  }
});
```

### Mode 2: Function Call + RPC (structured)

Gemini calls a `set_color` function tool with structured JSON (`{"color": "blue"}`). The agent executes the tool and sends the color to the browser via LiveKit RPC.

```
User speaks → LiveKit Cloud → Python Agent → Gemini Live API
                                                    ↓
                                              Gemini calls set_color({"color": "blue"})
                                                    ↓
                                              Agent executes tool
                                                    ↓
Browser UI  ← RPC ← Python Agent (pg.setColor RPC call)
```

**Agent code** (`agent/main.py`):

```python
@function_tool(raw_schema=raw_schema)
async def set_color(raw_arguments: dict) -> str:
    color = raw_arguments["color"]
    await session_manager.ctx.room.local_participant.perform_rpc(
        destination_identity=session_manager.participant.identity,
        method="pg.setColor",
        payload=json.dumps({"color": color}),
    )
    return f"Color changed to {color}"
```

**Frontend code** (`web/public/simple.html`):

```javascript
room.localParticipant.registerRpcMethod('pg.setColor', async (data) => {
  const { color } = JSON.parse(data.payload);
  document.body.style.backgroundColor = color;
  return JSON.stringify({ ok: true });
});
```

## Latency Comparison

| Mode | Latency | Pros | Cons |
|------|---------|------|------|
| **Transcription parsing** | ~instant | Fast, no agent round-trip | Simple string matching only |
| **Function call + RPC** | ~3-7s overhead | Structured JSON, Gemini decides when to call | Gemini pauses audio to execute tool |

Function calling adds latency because Gemini pauses to execute the tool and wait for the result before continuing its audio response. Transcription parsing is instant because it happens client-side as text streams in.

**When to use which:**
- Use **transcription parsing** when speed matters and the classification is simple (keyword matching, regex)
- Use **function calling** when you need structured data, complex decisions, or Gemini to reason about when to trigger an action

## Extending this: Use a classifier model

The simple string matching works for color names, but for more complex classification (e.g., determining if a student's answer is correct/incorrect/irrelevant in an education app), you can replace the matching logic with a classifier:

### Option 1: Client-side with a small model (e.g., BERT via ONNX)

```javascript
// Load a fine-tuned BERT model in the browser via ONNX Runtime Web
const session = await ort.InferenceSession.create('/models/classifier.onnx');

room.on(LivekitClient.RoomEvent.TranscriptionReceived, async (segs, participant) => {
  if (!participant?.isAgent) return;

  for (const seg of segs) {
    if (!seg.final) continue; // Only classify complete segments

    // Run classifier on the transcription text
    const result = await classify(session, seg.text);
    // result could be: { label: 'correct', confidence: 0.95 }

    updateUI(result.label); // green for correct, red for incorrect, etc.
  }
});
```

### Option 2: Server-side classifier

```javascript
room.on(LivekitClient.RoomEvent.TranscriptionReceived, async (segs, participant) => {
  if (!participant?.isAgent) return;

  for (const seg of segs) {
    if (!seg.final) continue;

    // Send to your classification API
    const resp = await fetch('/api/classify', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text: seg.text }),
    });
    const { label } = await resp.json();
    updateUI(label);
  }
});
```

### Option 3: Classify in the Python agent

You can also intercept transcriptions in the agent process itself, run a model there (e.g., Hugging Face transformers), and send the classification result to the frontend via RPC or data channels.

## Running the demo

1. Set up `.env.local` with your LiveKit and Google API credentials
2. Start the agent: `cd agent && source .venv/bin/activate && python main.py dev`
3. Start the web frontend: `cd web && pnpm dev`
4. Open `http://localhost:3000/simple.html`
5. Select a color detection mode from the dropdown
6. Edit the system prompt, click Connect, and say color names
7. Compare the latency between the two modes
