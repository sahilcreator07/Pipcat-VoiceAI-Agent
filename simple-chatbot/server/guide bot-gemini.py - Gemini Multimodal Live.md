# Guide: bot-gemini.py - Gemini Multimodal Live

## Overview

This guide explains the `bot-gemini.py` implementation, which creates an animated voice agent using Google's Gemini Multimodal Live model. The bot features real-time audio/video interaction through Daily, animated robot avatar, and speech-to-speech capabilities.

## Architecture Components

### Transport

#### DailyTransport
```python
transport = DailyTransport(
    room_url,
    token,
    "Chatbot",
    DailyParams(
        audio_in_enabled=True,      # Receives user audio
        audio_out_enabled=True,     # Sends bot audio
        video_out_enabled=True,     # Sends bot video
        video_out_width=1024,       # Video resolution
        video_out_height=576,
        vad_analyzer=SileroVADAnalyzer(params=VADParams(stop_secs=0.5)),
    ),
)
```

**Features:**
- Bidirectional audio/video streaming
- Voice Activity Detection (VAD) with Silero
- Daily room integration
- Real-time communication

### Processors

#### 1. RTVIProcessor
```python
rtvi = RTVIProcessor(config=RTVIConfig(config=[]))
```
- Handles RTVI (Real-Time Voice Interface) events
- Manages client connections and bot readiness
- Provides UI integration capabilities

#### 2. Context Aggregators
```python
context_aggregator = llm.create_context_aggregator(context)
```
- **User Context Aggregator**: Collects user messages
- **Assistant Context Aggregator**: Collects bot responses
- Maintains conversation history

#### 3. GeminiMultimodalLiveLLMService
```python
llm = GeminiMultimodalLiveLLMService(
    api_key=os.getenv("GEMINI_API_KEY"),
    voice_id="Puck",
    transcribe_user_audio=True,
)
```
- Processes audio/video input
- Generates multimodal responses
- Handles speech-to-speech conversion

#### 4. TalkingAnimation
```python
class TalkingAnimation(FrameProcessor):
    # Manages visual animation states
    # Switches between static and animated robot frames
```

**Animation System:**
- Loads 25 robot sprite frames (robot01.png to robot25.png)
- Creates smooth animation by adding reversed frames (50 total frames)
- Switches between static (listening) and animated (talking) states
- Event-driven animation based on speaking state

## Pipeline

```python
pipeline = Pipeline([
    transport.input(),              # 1. Audio/video input from Daily
    rtvi,                          # 2. RTVI event processing
    context_aggregator.user(),     # 3. Collect user context
    llm,                           # 4. Gemini LLM processing
    ta,                            # 5. Animation management
    transport.output(),            # 6. Audio/video output to Daily
    context_aggregator.assistant(), # 7. Collect assistant context
])
```

**Pipeline Features:**
- Linear processing flow
- Frame-based architecture
- Event-driven responses
- Real-time performance

## Execution Flow

### 1. Initialization Phase
```python
async def main():
    # 1. Create aiohttp session
    async with aiohttp.ClientSession() as session:
        
        # 2. Configure room connection
        (room_url, token) = await configure(session)
        
        # 3. Initialize transport
        transport = DailyTransport(...)
        
        # 4. Initialize LLM service
        llm = GeminiMultimodalLiveLLMService(...)
        
        # 5. Set up context and processors
        context = OpenAILLMContext(messages)
        context_aggregator = llm.create_context_aggregator(context)
        ta = TalkingAnimation()
        rtvi = RTVIProcessor(...)
        
        # 6. Create pipeline
        pipeline = Pipeline([...])
        
        # 7. Create task with observers
        task = PipelineTask(pipeline, observers=[RTVIObserver(rtvi)])
        
        # 8. Set initial animation state
        await task.queue_frame(quiet_frame)
```

### 2. Event-Driven Flow

#### Client Ready Event:
```python
@rtvi.event_handler("on_client_ready")
async def on_client_ready(rtvi):
    await rtvi.set_bot_ready()
    # Start conversation
    await task.queue_frames([context_aggregator.user().get_context_frame()])
```

#### Participant Events:
```python
@transport.event_handler("on_first_participant_joined")
async def on_first_participant_joined(transport, participant):
    print(f"Participant joined: {participant}")

@transport.event_handler("on_participant_left")
async def on_participant_left(transport, participant, reason):
    print(f"Participant left: {participant}")
    await task.cancel()
```

### 3. Real-Time Processing Flow

```
User speaks → Audio input
    ↓
transport.input() → Captures audio/video
    ↓
rtvi → Processes RTVI events
    ↓
context_aggregator.user() → Collects user message
    ↓
llm → Gemini processes input, generates response
    ↓
ta → Manages animation (talking/listening states)
    ↓
transport.output() → Sends audio/video to user
    ↓
context_aggregator.assistant() → Collects bot response
```

### 4. Animation Flow

```
Bot State Change → Frame Event → Animation Response
    ↓
BotStartedSpeakingFrame → Switch to animated sprite
    ↓
BotStoppedSpeakingFrame → Switch to static frame
```

## Animation System Details

### Sprite Loading
```python
sprites = []
for i in range(1, 26):
    full_path = os.path.join(script_dir, f"assets/robot0{i}.png")
    with Image.open(full_path) as img:
        sprites.append(OutputImageRawFrame(image=img.tobytes(), size=img.size, format=img.format))

# Create smooth animation by adding reversed frames
flipped = sprites[::-1]
sprites.extend(flipped)  # 50 frames total

# Define animation states
quiet_frame = sprites[0]  # Static frame when listening
talking_frame = SpriteFrame(images=sprites)  # Animated sequence when talking
```

### Animation States

| Bot State | Visual | Trigger |
|-----------|--------|---------|
| **Listening** | Static robot frame | Default state |
| **Talking** | Animated robot sequence | `BotStartedSpeakingFrame` |
| **Listening** | Static robot frame | `BotStoppedSpeakingFrame` |

## Key Features

### Transport Features:
- **Bidirectional audio/video** streaming
- **Voice Activity Detection** (VAD) with Silero
- **Daily room integration**
- **Real-time communication**

### Processor Features:
- **Multimodal processing** (audio + video)
- **Context management** (conversation history)
- **Visual animation** synchronization
- **RTVI event handling**

### Pipeline Features:
- **Linear processing** flow
- **Frame-based architecture**
- **Event-driven responses**
- **Real-time performance**

### Execution Features:
- **Async/await** architecture
- **Event handlers** for various states
- **Resource management** (session cleanup)
- **Error handling** and graceful shutdown

## Requirements

- Python 3.10+
- Daily API key
- Gemini API key
- Required dependencies (see requirements.txt)
- Robot sprite images (robot01.png to robot25.png)

## Usage

1. Set up environment variables:
   ```bash
   GEMINI_API_KEY=your_gemini_api_key
   DAILY_API_KEY=your_daily_api_key
   ```

2. Run the bot:
   ```bash
   python bot-gemini.py
   ```

3. Connect to the Daily room and start conversing with the animated robot!

## Technical Notes

- The bot uses **event-driven animation** based on speaking state
- **Real-time synchronization** between audio and visual elements
- **Multimodal processing** allows for rich conversational experiences
- **Frame-based architecture** ensures smooth performance
- **Resource cleanup** is handled automatically on shutdown

This creates a **complete voice agent system** that can engage in real-time multimodal conversations with animated visual feedback! 🎬🤖 