# CONTEXT.md — lyric-lab (for LLMs)

## Product (1 line)
Japanese song lyrics learning PWA. Search songs via LLM API → get lyrics + romaji + Chinese translation + word breakdown → study pronunciation with browser TTS → save vocabulary.

## Core Design Decisions

1. **No audio files**: App does NOT handle MP3 import/playback/recording. User listens to songs on QQ Music. App is purely for lyrics + language study.
2. **Two-step API flow**: (1) Search songs → candidate list, (2) Select → parse lyrics. Separate calls to keep JSON stable and avoid wasted tokens on wrong matches.
3. **LLM does everything**: Single DeepSeek API call per song returns jp, romaji, zh, and word breakdown (surface, reading, meaning, wordType). No external lyrics APIs or scraping.
4. **Browser TTS**: SpeechSynthesis API with ja-JP voice. Free, works offline. Per-sentence and per-word pronunciation.
5. **Word classification**: Each word tagged as `daily-use` (common Japanese) or `lyric-only` (poetic/song-specific). Visual distinction with green/orange tags.
6. **IndexedDB persistence**: songs, lyrics, vocab stores. Cache-first: same song never re-parsed.
7. **Single-file HTML**: All HTML + CSS + JS in index.html. Zero dependencies. Follows repo convention.
8. **PWA**: manifest.json + Service Worker (stale-while-revalidate for static assets, no API interception).
9. **Dark theme only**: CSS variables matching text-reader dark mode (`--ll-*` prefix).

## Data Model (IndexedDB)

```
Database: lyric-lab-db (v1)

songs (keyPath: id)
  - id (string, generated)
  - title, artist, searchQuery, createdAt (timestamp)

lyrics (keyPath: id, same as songId)
  - id, songId
  - lines: [{jp, romaji, zh, words: [{surface, reading, meaning, wordType}]}]
  - createdAt

vocab (keyPath: id)
  - id, word, reading, meaning, wordType
  - sourceSongId, sourceSongTitle, addedAt
  - Index: sourceSongId
```

## localStorage Keys
- `ll-api-key` (btoa encoded)
- `ll-api-endpoint` (default: https://api.deepseek.com)
- `ll-api-model` (default: deepseek-chat)
- `ll-show-romaji` (default: true)
- `ll-show-translation` (default: false)

## API Prompts

### Search
System: "你是日语歌曲搜索引擎。返回JSON：{songs:[{title,artist,confidence}]}。最多5首。"
User: raw search query

### Parse
System: Detailed instructions for lyrics + romaji + zh translation + word breakdown. wordType = "daily-use" or "lyric-only".
User: "歌曲：{title} — {artist}"
Response format: `{title, artist, lines: [{jp, romaji, zh, words: [{surface, reading, meaning, wordType}]}]}`

## Views & Navigation
- **search** (default): Search input + candidate list + "My Songs" list
- **lyrics**: 3-line display (JP/romaji/ZH) + per-sentence TTS + word tap → bottom sheet
- **vocab**: Words grouped by song, TTS on tap, delete support
- **settings**: API endpoint + key + model configuration

Bottom nav: search / lyrics / vocab. Settings via top-bar gear icon.

## Key Functions
- `LLStorage`: IndexedDB wrapper (open, put, get, getAll, getAllByIndex, delete)
- `LLAPI`: API calls (getConfig, isConfigured, searchSongs, parseLyrics)
- `LLTTS`: SpeechSynthesis wrapper (speak, stop, isAvailable)
- `toast.show(msg, type?)`: ephemeral notification

## Current State (V1)
- ✅ All P1 features implemented
- ✅ Song search + parse via DeepSeek API
- ✅ 3-line lyric view with show/hide toggles
- ✅ Per-sentence TTS pronunciation
- ✅ Word tap → bottom sheet → add to vocab
- ✅ Vocab list with TTS, grouped by song
- ✅ Settings page (API config)
- ✅ PWA (manifest + SW)
- ✅ Dark theme, mobile-first responsive (max-width 480px)
- ✅ Offline: cached songs viewable offline, TTS works offline

## Next Steps (P2)
- Flashcard review mode for vocab
- Manual lyric paste when API can't find a song
- SM-2 spaced repetition (reuse text-reader implementation)
- Better word interaction (highlight current word during TTS)

## Constraints
- Max-width 480px centered layout
- No build step, no external libraries
- SpeechSynthesis quality varies by browser/OS (Chrome Android best)
- LLM JSON parsing has try-catch + retry, but malformed responses still possible
- Single DeepSeek API key shared with text-reader (manually)
