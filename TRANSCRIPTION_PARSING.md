# Real-time Transcription Parsing from Gemini Live API

## How it works

This project uses the Gemini Live API (via LiveKit Agents) for real-time voice conversations. Gemini streams audio responses back to the browser, and the LiveKit SDK provides real-time transcriptions of that audio via `output_audio_transcription` (enabled by default in the LiveKit Gemini plugin).

We parse these transcriptions **on the frontend** to trigger UI actions — in this demo, changing the background color when Gemini says a color name.

### Architecture

```
User speaks → LiveKit Cloud → Python Agent → Gemini Live API
                                                    ↓
                                              Audio + Transcription
                                                    ↓
User hears ← LiveKit Cloud ← Python Agent ← Audio stream
Browser UI  ← LiveKit SDK  ← Transcription events (parsed client-side)
```

### The key code (`web/public/simple.html`)

Transcriptions arrive via LiveKit's `TranscriptionReceived` event. We parse them in real-time:

```javascript
// List of colors to detect
const CSS_COLORS = ['red', 'orange', 'yellow', 'green', 'blue', 'purple', ...];

room.on(LivekitClient.RoomEvent.TranscriptionReceived, (segs, participant) => {
  const isAgent = participant?.isAgent || false;

  for (const seg of segs) {
    // Parse color from agent transcription immediately
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

This is **significantly faster** than using Gemini's function calling for UI updates, because:
- Transcription text arrives as Gemini speaks (streaming)
- No round-trip through the agent for tool execution
- No waiting for Gemini to process the tool result before continuing

## Why not function calling?

We tested both approaches:

| Approach | Latency | How it works |
|----------|---------|-------------|
| **Transcription parsing** (current) | ~instant | Parse text as it streams in |
| **Function calling** (`set_color` tool) | ~3-7s overhead | Gemini calls tool → agent executes → RPC to browser → Gemini waits for result → continues speaking |

Function calling adds latency because Gemini pauses to execute the tool and wait for the result before continuing its audio response.

## Extending this: Use a classifier model

The simple string matching above works for color names, but for more complex classification (e.g., determining if a student's answer is correct/incorrect/irrelevant in an education app), you can replace the matching logic with a classifier:

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
5. Edit the system prompt, click Connect, and say color names
