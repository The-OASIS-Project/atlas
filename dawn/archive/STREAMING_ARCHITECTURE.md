# Streaming Architecture Diagram

## High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              dawn.c (Main Application)                       │
│                                                                              │
│  State Machine: PROCESS_COMMAND / VISION_AI_READY / NETWORK_PROCESSING      │
│         │                                                                    │
│         │ llm_chat_completion_streaming_tts(...)                            │
│         └──────────────────────────────┐                                    │
│                                        │                                    │
│         ┌──────────────────────────────┘                                    │
│         │                                                                    │
│         │  dawn_tts_sentence_callback() ◄────────────────────────┐          │
│         │         │                                               │          │
│         │         │ 1. Clean text (remove commands, emojis)      │          │
│         │         │ 2. text_to_speech(cleaned)                   │          │
│         │         │ 3. usleep(300000)  // 300ms pause            │          │
│         │         │                                               │          │
└─────────┼─────────┴───────────────────────────────────────────────┼──────────┘
          │                                                         │
          ▼                                                         │
┌─────────────────────────────────────────────────────────────────┼──────────┐
│                       llm_interface.c (Routing Layer)            │          │
│                                                                  │          │
│  llm_chat_completion_streaming_tts()                            │          │
│         │                                                        │          │
│         │ 1. Create sentence_buffer with tts_sentence_callback  │          │
│         │ 2. Call llm_chat_completion_streaming() with chunk CB │          │
│         │ 3. sentence_buffer_flush()                            │          │
│         │                                                        │          │
│         ├─► llm_chat_completion_streaming()                     │          │
│         │         │                                              │          │
│         │         │ Route based on llm_type & cloud_provider    │          │
│         │         │                                              │          │
│         │         ├─► llm_openai_chat_completion_streaming()    │          │
│         │         └─► llm_claude_chat_completion_streaming()    │          │
│         │                                                        │          │
│         │  tts_chunk_callback() ◄────────────────────┐          │          │
│         │         │                                   │          │          │
│         │         │ sentence_buffer_feed(chunk)       │          │          │
│         │         │                                   │          │          │
│         │  tts_sentence_callback() ───────────────────┼──────────┘          │
│         │         │                                   │                     │
│         │         │ user_callback(sentence) ──────────┘                     │
│         │         │                                                         │
└─────────┼─────────┴─────────────────────────────────────────────────────────┘
          │                                             ▲
          ▼                                             │
┌─────────────────────────────────────────────────────┼─────────────────────┐
│            llm_openai.c / llm_claude.c (Providers)   │                     │
│                                                      │                     │
│  llm_[provider]_chat_completion_streaming()         │                     │
│         │                                            │                     │
│         │ 1. Build JSON with "stream": true          │                     │
│         │ 2. Create llm_stream_context_t             │                     │
│         │ 3. Create sse_parser_t                     │                     │
│         │ 4. Setup CURL with streaming_write_callback│                     │
│         │ 5. curl_easy_perform()                     │                     │
│         │ 6. Return accumulated_response             │                     │
│         │                                            │                     │
│         │  streaming_write_callback()                │                     │
│         │         │                                  │                     │
│         │         │ Receives raw bytes from CURL     │                     │
│         │         │ sse_parser_feed(data, len)       │                     │
│         │         │                                  │                     │
│         │  [provider]_sse_event_handler()            │                     │
│         │         │                                  │                     │
│         │         │ llm_stream_handle_event()        │                     │
│         │         │                                  │                     │
└─────────┼─────────┴──────────────────────────────────┼─────────────────────┘
          │                                            │
          ▼                                            │
┌─────────────────────────────────────────────────────┼─────────────────────┐
│                    sse_parser.c (SSE Protocol)      │                     │
│                                                     │                     │
│  sse_parser_feed(data, len)                        │                     │
│         │                                           │                     │
│         │ 1. Append to buffer                       │                     │
│         │ 2. Split on newlines                      │                     │
│         │ 3. Parse "event:" and "data:" fields      │                     │
│         │ 4. On empty line (event complete):        │                     │
│         │                                           │                     │
│         │    dispatch_event() ──────────────────────┘                     │
│         │         │                                                       │
│         │         │ callback(event_type, event_data, userdata)            │
│         │         │                                                       │
└─────────┼─────────┴───────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                   llm_streaming.c (Delta Extraction)                         │
│                                                                              │
│  llm_stream_handle_event(event_data)                                        │
│         │                                                                    │
│         ├─► parse_openai_chunk()                                            │
│         │         │                                                          │
│         │         │ 1. Parse JSON: {"choices":[{"delta":{"content":"..."}}]}│
│         │         │ 2. Extract text from delta.content                      │
│         │         │ 3. Accumulate to response                               │
│         │         │ 4. callback(text_chunk)  ─────────────┐                 │
│         │         │ 5. Check for "[DONE]"                 │                 │
│         │                                                 │                 │
│         └─► parse_claude_event()                          │                 │
│                   │                                       │                 │
│                   │ 1. Parse JSON: {"type":"content_block_delta", ...}      │
│                   │ 2. Extract text from delta.text       │                 │
│                   │ 3. Accumulate to response             │                 │
│                   │ 4. callback(text_chunk)  ─────────────┤                 │
│                   │ 5. Check for "message_stop"           │                 │
│                                                           │                 │
└───────────────────────────────────────────────────────────┼─────────────────┘
                                                            │
          ┌─────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                  sentence_buffer.c (Sentence Boundary Detection)             │
│                                                                              │
│  sentence_buffer_feed(chunk)                                                │
│         │                                                                    │
│         │ 1. Append chunk to buffer                                         │
│         │ 2. Search for sentence terminators: . ! ? :                       │
│         │ 3. Check if followed by space/newline/end                         │
│         │ 4. Extract complete sentence                                      │
│         │ 5. Trim whitespace                                                │
│         │ 6. callback(sentence) ────────┐                                   │
│         │ 7. Remove processed from buffer│                                   │
│         │                                │                                   │
│  sentence_buffer_flush()                 │                                   │
│         │                                │                                   │
│         │ Send remaining incomplete text │                                   │
│         │ callback(remaining) ───────────┤                                   │
│                                          │                                   │
└──────────────────────────────────────────┼───────────────────────────────────┘
                                           │
                                           │
                      ┌────────────────────┘
                      │
                      └───► Back to tts_sentence_callback in llm_interface.c
                            └───► Back to dawn_tts_sentence_callback in dawn.c
                                  └───► text_to_speech() + usleep(300000)
```

## Data Flow Example

**Input**: "Hello! How are you today?"

```
LLM Stream → "Hello" → "!" → " How" → " are" → " you" → " today" → "?"

                               ↓ (chunk by chunk)

llm_stream_handle_event()     [Extracts text from JSON deltas]

                               ↓

sentence_buffer_feed()        [Accumulates: "Hello! "]
   └─► callback("Hello!")     [Sentence 1 complete - has ! + space]

sentence_buffer_feed()        [Accumulates: "How are you today"]

sentence_buffer_flush()       [End of stream]
   └─► callback("How are you today?")  [Sentence 2 complete]

                               ↓

dawn_tts_sentence_callback()
   └─► text_to_speech("Hello!")
   └─► usleep(300ms)

dawn_tts_sentence_callback()
   └─► text_to_speech("How are you today?")
   └─► usleep(300ms)
```

## Key Components

### 1. SSE Parser (`sse_parser.c`)
- **Purpose**: Parse Server-Sent Events protocol
- **Input**: Raw bytes from CURL (may be partial)
- **Output**: Complete SSE events via callback
- **Format**: `event: <type>\ndata: <json>\n\n`

### 2. LLM Streaming (`llm_streaming.c`)
- **Purpose**: Extract text deltas from provider-specific JSON
- **OpenAI**: `choices[0].delta.content` + `[DONE]` signal
- **Claude**: `delta.text` with `content_block_delta` event type
- **Accumulates**: Complete response for conversation history

### 3. Sentence Buffer (`sentence_buffer.c`)
- **Purpose**: Detect sentence boundaries for natural TTS
- **Terminators**: `.` `!` `?` `:` followed by space/newline/end
- **Buffering**: Holds partial sentences across chunks
- **Flushing**: Sends remaining text at end of stream

### 4. LLM Interface (`llm_interface.c`)
- **Purpose**: High-level API with provider routing
- **Streaming**: `llm_chat_completion_streaming()` - raw chunks
- **TTS Streaming**: `llm_chat_completion_streaming_tts()` - sentences
- **Routing**: Selects OpenAI vs Claude vs Local based on config

### 5. Provider Implementations (`llm_openai.c`, `llm_claude.c`)
- **Purpose**: HTTP streaming with provider-specific APIs
- **Setup**: CURL with `CURLOPT_WRITEFUNCTION` callback
- **JSON**: Adds `"stream": true` to request
- **State**: Manages SSE parser and stream context

## Thread Safety

- **Vosk Model**: Read-only, separate recognizers per thread
- **TTS Engine**: Protected by `tts_mutex` in dawn.c
- **LLM Endpoint**: Stateless HTTP requests (inherently thread-safe)
- **Streaming Contexts**: Created per-request (not shared)

## Error Handling

- **Network Errors**: Falls back to local LLM if cloud fails
- **Malformed JSON**: Logged and skipped (stream continues)
- **Incomplete Events**: Buffered until complete (SSE parser)
- **Memory Allocation**: Checked and logged, graceful failures

## Performance Characteristics

- **Latency Reduction**: ~10-15s → ~1-2s for first sentence
- **Memory Usage**: Dynamic buffers grow as needed (2x strategy)
- **CPU Overhead**: Minimal - JSON parsing + string operations
- **Network**: Single long-lived HTTP connection per request
