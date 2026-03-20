# Summarizer Fixes + TF-IDF Implementation Plan

**Created:** 2026-01-20
**Status:** Planning phase - implementation not started

---

## Overview

Two related improvements to the search/URL summarization system:

1. **Fix LLM summarizer fallback behavior** - Prevent summarizer failures from triggering global LLM fallback
2. **Add TF-IDF extractive summarization** - Local C implementation as alternative to LLM-based summarization

---

## Problem Analysis

From log analysis (session 2026-01-20), the following cascade occurred:

```
URL fetch (58KB content)
  → LLM summarizer called (Claude)
    → 60-second timeout hit
      → Fallback to local LLM triggered (WRONG!)
        → Global LLM type changed to LOCAL
        → TTS "Unable to contact cloud LLM" played
        → Local LLM fails (HTTP 400 - payload too big)
          → Content truncated 58KB → 16KB → 8KB
```

**Root causes:**
1. `llm_chat_completion()` has automatic fallback logic that's inappropriate for internal summarization
2. No local summarization option exists - always requires LLM API call
3. Truncation is naive (head-only), loses important content

---

## Part 1: Fix LLM Summarizer Fallback

### Approach: Add `allow_fallback` parameter (Option A)

**Why this approach:**
- Clean and explicit API
- No hidden state
- Callers explicitly choose fallback behavior

### Files to Modify

1. **`include/llm/llm_interface.h`**
   - Change signature: `llm_chat_completion(..., bool allow_fallback)`
   - Update documentation

2. **`src/llm/llm_interface.c`** (lines 755-840)
   - Add `allow_fallback` parameter
   - Check flag before fallback block (lines 826-837)
   - Same for `llm_chat_completion_streaming()` (lines 842+)

3. **Callers to update (pass `true` for existing behavior):**
   - `src/llm/llm_interface.c:1217` - internal wrapper
   - `src/llm/llm_context.c:752` - context compaction
   - `src/mosquitto_comms.c:620` - MQTT response

4. **`src/tools/search_summarizer.c:383`**
   - Pass `false` - summarizer should NOT trigger fallback

### Implementation Detail

```c
// In llm_interface.c, modify fallback block:
if (response == NULL && type == LLM_CLOUD && allow_fallback && !llm_is_interrupt_requested()) {
    // ... existing fallback logic
}
```

---

## Part 2: TF-IDF Extractive Summarization

### Why TF-IDF?

| Method | Output Size | LLM Call | Tokens | Speed |
|--------|-------------|----------|--------|-------|
| LLM Summarization | ~5-10% | Yes | API cost | Slow (network) |
| TF-IDF Extractive | ~10-20% | **No** | **Zero** | **Fast (local)** |
| Lead+Tail Truncation | ~80% | Optional | High | Fast |

TF-IDF extracts the most important sentences without any API call.

### Algorithm Overview

```
1. Split document into sentences
2. Tokenize each sentence into words
3. Remove stopwords (the, a, is, etc.)
4. Calculate TF (Term Frequency) per sentence
5. Calculate IDF (Inverse Document Frequency) across all sentences
6. Score each sentence: avg(TF-IDF of words in sentence)
7. Select top N sentences by score
8. Return sentences in original order
```

### Files to Create

1. **`include/tools/tfidf_summarizer.h`**
   ```c
   #ifndef TFIDF_SUMMARIZER_H
   #define TFIDF_SUMMARIZER_H

   // Configuration
   #define TFIDF_MAX_SENTENCES 500
   #define TFIDF_MAX_WORDS 10000
   #define TFIDF_DEFAULT_RATIO 0.2  // Keep top 20% of sentences

   // Main API
   int tfidf_summarize(const char *input_text,
                       char **output_summary,
                       float keep_ratio);  // 0.1 = 10%, 0.2 = 20%

   // For testing/debugging
   int tfidf_score_sentences(const char *input_text,
                             float *scores_out,
                             int max_sentences);

   #endif
   ```

2. **`src/tools/tfidf_summarizer.c`**
   - Sentence splitter (~50 lines)
   - Tokenizer with stopword filter (~80 lines)
   - Word frequency hash table (~100 lines)
   - TF-IDF calculation (~50 lines)
   - Sentence ranking and selection (~50 lines)
   - Main `tfidf_summarize()` function (~30 lines)
   - **Total: ~360 lines**

### Stopword List

Hardcoded English stopwords (130 common words):
```c
static const char *STOPWORDS[] = {
    "a", "an", "the", "and", "or", "but", "in", "on", "at", "to", "for",
    "of", "with", "by", "from", "as", "is", "was", "are", "were", "been",
    "be", "have", "has", "had", "do", "does", "did", "will", "would", "could",
    "should", "may", "might", "must", "shall", "can", "need", "dare", "ought",
    "used", "it", "its", "this", "that", "these", "those", "i", "you", "he",
    "she", "we", "they", "what", "which", "who", "whom", "whose", "where",
    "when", "why", "how", "all", "each", "every", "both", "few", "more",
    "most", "other", "some", "such", "no", "nor", "not", "only", "own",
    "same", "so", "than", "too", "very", "just", "also", "now", "here",
    "there", "then", "once", "always", "never", "often", "still", "already",
    "about", "after", "before", "between", "through", "during", "above",
    "below", "under", "over", "again", "further", "into", "out", "up", "down",
    NULL
};
```

### Hash Table for Word Counts

Simple open-addressing hash table:
```c
typedef struct {
    char word[64];
    int count;
    int doc_count;  // Number of sentences containing this word
} word_entry_t;

typedef struct {
    word_entry_t *entries;
    int capacity;
    int count;
} word_table_t;
```

### Integration with search_summarizer

Add new backend option in `dawn.toml`:
```toml
[search.summarizer]
backend = "tfidf"  # NEW option: "disabled", "local", "default", "tfidf"
threshold_bytes = 3072
target_ratio = 0.2  # For TF-IDF: keep top 20% of sentences
```

Modify `search_summarizer.c`:
```c
case BACKEND_TFIDF:
    return tfidf_summarize(content, out_summary, config->target_ratio);
```

---

## Part 3: Build Integration

### CMakeLists.txt Addition

```cmake
# In src/tools section
src/tools/tfidf_summarizer.c
```

### Config Changes

1. **`include/config/dawn_config.h`**
   - Add `float target_ratio` to `summarizer_file_config_t`

2. **`src/config/config_defaults.c`**
   - Set default `target_ratio = 0.2`

3. **`src/config/config_parser.c`**
   - Parse `target_ratio` field

---

## Implementation Order

1. [ ] Add `allow_fallback` parameter to `llm_chat_completion()`
2. [ ] Update all callers to pass `true`
3. [ ] Update `search_summarizer.c` to pass `false`
4. [ ] Create `tfidf_summarizer.h` header
5. [ ] Implement tokenizer and stopword filter
6. [ ] Implement hash table for word counts
7. [ ] Implement TF-IDF scoring
8. [ ] Implement sentence ranking and extraction
9. [ ] Add config support for `tfidf` backend and `target_ratio`
10. [ ] Integrate as new backend in `search_summarizer.c`
11. [ ] Add to CMakeLists.txt
12. [ ] Test with various content sizes
13. [ ] Update `dawn.toml.example` documentation

---

## Testing Plan

### Unit Tests
- Tokenizer splits sentences correctly
- Stopwords are filtered
- TF-IDF scores are calculated correctly
- Top sentences are selected

### Integration Tests
- Search with large results triggers TF-IDF
- URL fetch with large content uses TF-IDF
- Output is coherent (sentences in original order)
- Performance: should process 50KB in <100ms

### Comparison Test
- Same content through LLM summarizer vs TF-IDF
- Compare output quality and token usage

---

## Open Questions

1. Should TF-IDF be the default backend, or keep LLM as default?
2. Target ratio: 20% seems reasonable, but should it be configurable per-request?
3. Should we add sentence position weighting (first/last sentences often important)?

---

## References

- [TF-IDF Wikipedia](https://en.wikipedia.org/wiki/Tf–idf)
- [Extractive Summarization with TF-IDF](https://medium.com/@ashinsk/text-summarization-f2542bc6a167)
- [TextRank paper](https://aclanthology.org/W04-3252/) (alternative graph-based approach)
