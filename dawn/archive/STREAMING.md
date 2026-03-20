# LLM Streaming Implementation Plan

## Overview

Implement Server-Sent Events (SSE) streaming for all LLM providers (OpenAI, Claude, llama.cpp) to enable real-time token-by-token response delivery and immediate text-to-speech output.

**Benefits:**
- Faster perceived response time (TTS starts immediately)
- Better user experience (no long silence while waiting for complete response)
- Lower latency between user query and first spoken word
- Efficient use of network bandwidth

---

## Current Architecture

### Existing Flow
```
User speaks → Speech recognition → llm_chat_completion() →
Wait for complete response → text_to_speech() → Audio output
```

**Problem:** User waits for entire LLM response before hearing anything.

### Target Flow
```
User speaks → Speech recognition → llm_chat_completion_streaming() →
Receive tokens → text_to_speech(chunk) → Audio output (immediate)
                    ↓
                Continue receiving tokens...
```

---

## SSE Protocol Overview

All three providers use Server-Sent Events (SSE), an HTTP streaming protocol where the server sends multiple responses for a single request.

### SSE Format
```
data: <JSON_PAYLOAD>\n\n
```

Each event is:
- Prefixed with `data: `
- Followed by JSON payload
- Terminated with double newline `\n\n`

### Special Considerations
- libcurl's `CURLOPT_WRITEFUNCTION` callback receives **chunks**, not necessarily complete lines
- Need to buffer partial events and parse on `\n\n` boundaries
- Must handle mid-JSON chunks gracefully

---

## Provider-Specific Formats

### 1. OpenAI Streaming

**Request:**
```json
{
  "model": "gpt-4o",
  "messages": [...],
  "stream": true
}
```

**Response Events:**
```
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1234567890,"model":"gpt-4o","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1234567890,"model":"gpt-4o","choices":[{"index":0,"delta":{"content":" world"},"finish_reason":null}]}

data: [DONE]
```

**Key Fields:**
- Incremental text: `choices[0].delta.content`
- Completion signal: `data: [DONE]`
- Finish reason: `choices[0].finish_reason` (null until final chunk)

**Edge Cases:**
- First chunk may only contain role: `{"delta":{"role":"assistant"}}`
- Empty content chunks are possible
- `[DONE]` is a special string, not JSON

---

### 2. Claude Streaming

**Request:**
```json
{
  "model": "claude-sonnet-4-5-20250929",
  "messages": [...],
  "stream": true
}
```

**Response Event Sequence:**
```
1. message_start
2. content_block_start
3. content_block_delta (multiple)
4. content_block_stop
5. message_delta
6. message_stop
```

**Event Examples:**

**message_start:**
```json
{
  "type": "message_start",
  "message": {
    "id": "msg_...",
    "role": "assistant",
    "content": [],
    "model": "claude-sonnet-4-5-20250929",
    "usage": {"input_tokens": 25, "output_tokens": 1}
  }
}
```

**content_block_start:**
```json
{
  "type": "content_block_start",
  "index": 0,
  "content_block": {"type": "text", "text": ""}
}
```

**content_block_delta (TEXT):**
```json
{
  "type": "content_block_delta",
  "index": 0,
  "delta": {
    "type": "text_delta",
    "text": "Hello world"
  }
}
```

**content_block_stop:**
```json
{
  "type": "content_block_stop",
  "index": 0
}
```

**message_delta:**
```json
{
  "type": "message_delta",
  "delta": {
    "stop_reason": "end_turn",
    "stop_sequence": null
  },
  "usage": {"output_tokens": 15}
}
```

**message_stop:**
```json
{
  "type": "message_stop"
}
```

**Key Fields:**
- Event type: `type` field determines how to parse
- Incremental text: `delta.text` in `content_block_delta` events
- Completion: `message_stop` event
- Token counts: Cumulative in `message_delta.usage`

**Edge Cases:**
- Multiple content blocks possible (e.g., text + tool_use)
- Thinking blocks for extended thinking models
- Tool use blocks contain `partial_json` instead of `text`
- `ping` events may arrive at any time (ignore these)

---

### 3. llama.cpp Streaming

**Request:**
```json
{
  "model": "llama-3.2-1b",
  "messages": [...],
  "stream": true
}
```

**Response Format:**
**Identical to OpenAI** - llama.cpp implements OpenAI-compatible endpoints.

**Endpoint:**
- `/v1/chat/completions`
- Default port: `8080`

**Same parsing logic as OpenAI applies.**

---

## Implementation Architecture

### Phase 1: SSE Parser Core

**New Files:**
- `sse_parser.c` / `sse_parser.h` - Generic SSE event parser

**Functions:**
```c
typedef void (*sse_event_callback)(const char *event_type,
                                   const char *event_data,
                                   void *userdata);

typedef struct {
    char *buffer;           // Accumulation buffer for partial events
    size_t buffer_size;
    size_t buffer_capacity;
    sse_event_callback callback;
    void *callback_userdata;
} sse_parser_t;

sse_parser_t *sse_parser_create(sse_event_callback callback, void *userdata);
void sse_parser_free(sse_parser_t *parser);
void sse_parser_feed(sse_parser_t *parser, const char *data, size_t len);
```

**Responsibilities:**
1. Buffer incoming data chunks
2. Split on `\n\n` to find complete events
3. Parse `event:` and `data:` fields
4. Call user callback for each complete event
5. Handle partial events across multiple chunks

---

### Phase 2: Provider-Specific Delta Extractors

**New Files:**
- `llm_streaming.c` / `llm_streaming.h` - Streaming coordinator

**Functions:**
```c
typedef void (*text_chunk_callback)(const char *text, void *userdata);

typedef struct {
    llm_provider_t provider;  // OPENAI, CLAUDE, LOCAL
    text_chunk_callback text_callback;
    void *text_callback_userdata;

    // State tracking
    int message_started;
    int content_block_active;
    char *accumulated_response;  // For conversation history
} llm_stream_context_t;

llm_stream_context_t *llm_stream_create(llm_provider_t provider,
                                        text_chunk_callback callback,
                                        void *userdata);
void llm_stream_free(llm_stream_context_t *ctx);

// Called by SSE parser for each event
void llm_stream_handle_event(llm_stream_context_t *ctx,
                             const char *event_data);

// Get complete accumulated response for conversation history
char *llm_stream_get_response(llm_stream_context_t *ctx);
```

**Provider-Specific Parsing:**

**OpenAI/llama.cpp:**
```c
static void parse_openai_chunk(json_object *chunk, llm_stream_context_t *ctx) {
    // Check for [DONE]
    // Extract choices[0].delta.content
    // Call text_callback if content exists
    // Append to accumulated_response
}
```

**Claude:**
```c
static void parse_claude_event(json_object *event, llm_stream_context_t *ctx) {
    const char *type = get_event_type(event);

    if (strcmp(type, "content_block_delta") == 0) {
        json_object *delta = get_delta(event);
        const char *delta_type = get_delta_type(delta);

        if (strcmp(delta_type, "text_delta") == 0) {
            const char *text = get_text(delta);
            ctx->text_callback(text, ctx->text_callback_userdata);
            append_to_accumulated(ctx, text);
        }
    } else if (strcmp(type, "message_stop") == 0) {
        // Stream complete
    }
}
```

---

### Phase 3: Integration with llm_interface

**Modify `llm_interface.h`:**
```c
// Existing non-streaming function
char *llm_chat_completion(struct json_object *conversation_history,
                         const char *input_text,
                         char *vision_image,
                         size_t vision_image_size);

// New streaming function
typedef void (*llm_text_chunk_callback)(const char *chunk, void *userdata);

char *llm_chat_completion_streaming(struct json_object *conversation_history,
                                    const char *input_text,
                                    char *vision_image,
                                    size_t vision_image_size,
                                    llm_text_chunk_callback chunk_callback,
                                    void *callback_userdata);
```

**Return Value:**
- Returns complete accumulated response (for conversation history)
- Chunks are delivered via callback during execution

---

### Phase 4: CURL Integration

**Modify provider implementations (llm_openai.c, llm_claude.c):**

```c
// Existing struct for non-streaming
struct MemoryStruct {
    char *memory;
    size_t size;
};

// New struct for streaming
struct StreamingContext {
    sse_parser_t *parser;
    llm_stream_context_t *stream_ctx;
};

static size_t streaming_write_callback(void *contents, size_t size,
                                       size_t nmemb, void *userp) {
    size_t realsize = size * nmemb;
    struct StreamingContext *ctx = (struct StreamingContext *)userp;

    // Feed data to SSE parser
    sse_parser_feed(ctx->parser, contents, realsize);

    return realsize;
}

// In llm_openai_chat_completion_streaming():
struct StreamingContext stream_ctx;
stream_ctx.parser = sse_parser_create(sse_event_handler, &stream_ctx);
stream_ctx.stream_ctx = llm_stream_create(LLM_OPENAI, chunk_callback, userdata);

curl_easy_setopt(curl_handle, CURLOPT_WRITEFUNCTION, streaming_write_callback);
curl_easy_setopt(curl_handle, CURLOPT_WRITEDATA, &stream_ctx);

// Add "stream": true to JSON payload
json_object_object_add(root, "stream", json_object_new_boolean(TRUE));

curl_easy_perform(curl_handle);

// Get accumulated response
char *response = llm_stream_get_response(stream_ctx.stream_ctx);

// Cleanup
llm_stream_free(stream_ctx.stream_ctx);
sse_parser_free(stream_ctx.parser);
```

---

### Phase 5: TTS Integration

**Challenge:** Current TTS expects complete sentences, but streaming delivers partial words.

**Solutions:**

**Option A: Word Count Buffering**
- Buffer N words (e.g., 5-10) before sending to TTS
- Fixed chunk sizes provide predictable pacing
- Trade-off: May break mid-phrase

**Option B: Sentence Buffering (Recommended)**
- Buffer chunks until sentence boundary (`.`, `!`, `?`, `:`)
- Send complete sentences to TTS
- Best quality - TTS engines work best with complete thoughts
- Trade-off: Slightly higher latency, but acceptable for FRIDAY's short responses

**Option C: Real-time Streaming**
- Send chunks immediately to TTS
- May sound choppy for most TTS engines
- Not recommended

**Why Sentence Buffering:**
1. TTS engines produce most natural speech with complete sentences
2. FRIDAY persona uses short, punchy sentences (rarely long)
3. Prevents awkward mid-phrase pauses
4. Better prosody and intonation
5. Worth the minor latency for quality improvement

**Implementation:**
```c
typedef struct {
    char *buffer;
    size_t size;
    size_t capacity;
} tts_chunk_buffer_t;

// Check if character is a sentence boundary
static int is_sentence_boundary(char c) {
    return (c == '.' || c == '!' || c == '?' || c == ':');
}

void tts_chunk_handler(const char *chunk, void *userdata) {
    tts_chunk_buffer_t *buf = (tts_chunk_buffer_t *)userdata;

    append_to_buffer(buf, chunk);

    // Look for sentence boundaries
    char *sentence_end = NULL;
    for (size_t i = 0; i < buf->size; i++) {
        if (is_sentence_boundary(buf->buffer[i])) {
            // Found a sentence boundary
            // Check if there's whitespace after (e.g., ". " not "...")
            if (i + 1 < buf->size && buf->buffer[i + 1] == ' ') {
                sentence_end = &buf->buffer[i + 1];
                break;
            }
            // Also break if it's the last character
            if (i == buf->size - 1) {
                sentence_end = &buf->buffer[i + 1];
                break;
            }
        }
    }

    if (sentence_end != NULL) {
        // Save the position
        size_t sentence_len = sentence_end - buf->buffer;

        // Null-terminate at sentence boundary
        char saved_char = *sentence_end;
        *sentence_end = '\0';

        // Send complete sentence(s) to TTS
        text_to_speech(buf->buffer);

        // Restore character
        *sentence_end = saved_char;

        // Keep remainder in buffer (skip the space after punctuation)
        size_t remainder_len = buf->size - sentence_len;
        if (remainder_len > 0) {
            memmove(buf->buffer, sentence_end, remainder_len);
            buf->buffer[remainder_len] = '\0';
            buf->size = remainder_len;
        } else {
            buf->buffer[0] = '\0';
            buf->size = 0;
        }
    }
}

// At end of stream, flush remaining buffer
void flush_tts_buffer(tts_chunk_buffer_t *buf) {
    if (buf->size > 0) {
        text_to_speech(buf->buffer);
    }
}
```

**Edge Cases:**
- Abbreviations (e.g., "Dr." or "U.S.A.") - May cause premature splits
- Ellipsis ("...") - Check for space after period
- Numbers with decimals (e.g., "3.14") - Check for digit after period
- Empty sentences - Skip if sentence is only punctuation

**Optimization:**
- Could add simple heuristic: if period followed by lowercase, likely abbreviation
- Example: "Dr. Smith" - don't split, "Great. Next" - do split

---

### Phase 6: Dawn Integration

**Modify `dawn.c` PROCESS_COMMAND state:**

```c
// Current:
response_text = llm_chat_completion(conversation_history, command_text, NULL, 0);
text_to_speech(response_text);

// New:
tts_chunk_buffer_t tts_buffer = {0};
tts_buffer.buffer = malloc(4096);
tts_buffer.capacity = 4096;

response_text = llm_chat_completion_streaming(
    conversation_history,
    command_text,
    NULL, 0,
    tts_chunk_handler,  // Callback for each chunk
    &tts_buffer         // Userdata
);

// Flush any remaining buffered text
flush_tts_buffer(&tts_buffer);

// Clean up command tags from response before adding to history
// (same as current implementation)
```

**Modify `dawn.c` VISION_AI_READY state similarly.**

---

## Testing Strategy

### Unit Tests

1. **SSE Parser Tests:**
   - Single complete event
   - Multiple events in one chunk
   - Event split across multiple chunks
   - Malformed events
   - Empty events

2. **Delta Extractor Tests:**
   - OpenAI single chunk
   - OpenAI multi-chunk
   - Claude event sequence
   - Claude with multiple content blocks
   - llama.cpp format

### Integration Tests

1. **Provider Tests:**
   - OpenAI streaming vs non-streaming (same output)
   - Claude streaming vs non-streaming (same output)
   - Local LLM streaming vs non-streaming (same output)

2. **TTS Tests:**
   - Verify word-boundary splitting works
   - Measure time-to-first-word
   - Verify complete response is preserved

### Manual Tests

1. Ask simple question, verify immediate TTS response
2. Ask complex question, verify streaming continues smoothly
3. Test vision with streaming
4. Test with poor network (simulated latency)
5. Test error handling (disconnect mid-stream)

---

## Error Handling

### Stream Interruption
- If stream stops mid-response, accumulate what we have
- Add partial response to conversation history
- Log warning about incomplete stream
- Consider retry with continuation (Claude supports this)

### JSON Parse Errors
- Skip malformed events
- Log error but continue processing stream
- Don't crash on unexpected event types

### Network Errors
- Fall back to non-streaming mode
- Existing error handling applies

---

## Performance Considerations

### Memory
- SSE parser buffer: Max 64KB (configurable)
- TTS chunk buffer: Max 4KB
- Accumulated response: Unbounded (same as current)

### Latency
- Target: <100ms from first token to TTS start
- Measure: Time between curl_easy_perform() and first text_to_speech() call

### Token Processing Rate
- OpenAI/Claude: ~50-100 tokens/sec typical
- Local LLM: Varies by model/hardware (10-50 tokens/sec)

---

## Configuration

### Enable/Disable Streaming
```c
// In secrets.h or runtime flag
#define ENABLE_LLM_STREAMING 1

// Runtime command-line flag
./dawn --streaming
./dawn --no-streaming
```

### Provider-Specific Settings
```c
// OpenAI: Always use streaming (well-supported)
// Claude: Always use streaming (well-supported)
// Local: Configurable (some models may not support it)
```

---

## Implementation Phases

### Phase 1: SSE Parser (Week 1)
- [ ] Create sse_parser.c/h
- [ ] Implement buffering and event extraction
- [ ] Write unit tests
- [ ] Document API

### Phase 2: Delta Extractors (Week 1-2)
- [ ] Create llm_streaming.c/h
- [ ] Implement OpenAI delta extraction
- [ ] Implement Claude delta extraction
- [ ] Write unit tests
- [ ] Document API

### Phase 3: Provider Integration (Week 2)
- [ ] Add streaming support to llm_openai.c
- [ ] Add streaming support to llm_claude.c
- [ ] Test with real APIs
- [ ] Verify response parity with non-streaming

### Phase 4: TTS Integration (Week 2-3)
- [ ] Implement word-boundary buffering
- [ ] Test with text_to_speech.cpp
- [ ] Measure latency improvements
- [ ] Fine-tune buffer sizes

### Phase 5: Dawn Integration (Week 3)
- [ ] Modify PROCESS_COMMAND state
- [ ] Modify VISION_AI_READY state
- [ ] Add configuration options
- [ ] Integration testing

### Phase 6: Polish & Documentation (Week 3-4)
- [ ] Error handling improvements
- [ ] Performance tuning
- [ ] Update README
- [ ] User documentation

---

## Success Metrics

1. **Latency:** Time-to-first-word reduced by >80%
2. **Quality:** TTS sounds natural (not choppy)
3. **Reliability:** No regressions in success rate
4. **Compatibility:** Works with all three providers
5. **Maintainability:** Clean separation of concerns

---

## Future Enhancements

1. **Parallel Streaming:** Stream to both TTS and display simultaneously
2. **Adaptive Buffering:** Adjust buffer size based on network conditions
3. **Resume on Disconnect:** Use Claude's continuation feature
4. **Streaming Commands:** Parse commands as they arrive, execute early
5. **Token Usage Tracking:** Real-time token count display

---

## References

- [OpenAI Streaming Docs](https://platform.openai.com/docs/api-reference/chat/streaming)
- [Claude Streaming Docs](https://docs.claude.com/en/docs/build-with-claude/streaming)
- [llama.cpp Server README](https://github.com/ggerganov/llama.cpp/tree/master/examples/server)
- [MDN SSE Guide](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [libcurl CURLOPT_WRITEFUNCTION](https://curl.se/libcurl/c/CURLOPT_WRITEFUNCTION.html)
