# Text Formatting

How you format input text directly affects the prosody (rhythm, pacing, and pauses) of the generated speech. murmr's model interprets whitespace and punctuation as cues for pausing and phrasing. Maximum text length is 4,096 characters per request.

## How Newlines Affect Prosody

| Input | Effect |
|-------|--------|
| No newline (continuous text) | Continuous speech, minimal pauses |
| `\n` (single newline) | Sentence-level pause |
| `\n\n` (double newline) | Paragraph-level pause (longer, more distinct) |

### Example: Before and After

**Without formatting (rushed):**
```
Welcome to the platform. We have three plans available. The free plan includes fifty thousand characters per month. The starter plan costs ten dollars.
```

**With formatting (natural pacing):**
```
Welcome to the platform.

We have three plans available.

The free plan includes fifty thousand characters per month.
The starter plan costs ten dollars.
```

The double newline between the welcome and the plan introduction creates a clear paragraph break. Single newlines between plan descriptions create natural sentence pauses.

## Punctuation Effects

| Character | Effect on Speech |
|-----------|-----------------|
| `.` | End of sentence, falling intonation, natural pause |
| `!` | End of sentence, emphasis |
| `?` | End of sentence, rising intonation |
| `,` | Brief pause within a sentence |
| `;` | Medium pause within a sentence |
| `:` | Medium pause, introduces what follows |
| `--` or `---` | Dramatic pause, parenthetical aside |
| `...` | Trailing off, hesitation |
| `\n` | Sentence-level pause |
| `\n\n` | Paragraph-level pause |

## Best Practices

| Do | Do Not |
|----|--------|
| Use `\n\n` between paragraphs | Run all text together in one line |
| Use `\n` between distinct sentences | Use `\n` for every line of a hard-wrapped document |
| Add punctuation at sentence endings | Omit periods, commas, and question marks |
| Write numbers as words ("fifty") | Use digits for spoken content ("50") |
| Clean text before sending | Send raw markdown, HTML, or formatting codes |
| Spell out abbreviations ("Doctor Smith") | Rely on abbreviation expansion ("Dr. Smith") |

## Common Pitfalls

### Hard-Wrapped Text

Text from PDFs, emails, or terminals often has hard line breaks at 72-80 characters. These cause unintended sentence pauses.

**Problem:**
```
The murmr text-to-speech API provides natural sounding speech
generation with support for multiple languages and custom voice
descriptions using natural language.
```

**Fix:**
```
The murmr text-to-speech API provides natural sounding speech generation with support for multiple languages and custom voice descriptions using natural language.
```

### Markdown Left in Text

Markdown syntax (`#`, `**`, `*`, `` ` ``, `[]()`) is read aloud or causes garbled output.

**Problem:**
```
## Getting Started

Visit [murmr.dev](https://murmr.dev) to **create** an account.
```

**Fix:**
```
Getting Started.

Visit murmr.dev to create an account.
```

### Missing Punctuation

Without sentence-ending punctuation, the model does not know where to pause.

**Problem:**
```
Welcome to the app Click the button to continue Then enter your name
```

**Fix:**
```
Welcome to the app. Click the button to continue. Then enter your name.
```

### Excessive Newlines

Too many blank lines create awkwardly long pauses. Use at most two consecutive newlines.

## Preprocessing Function

Strip markdown formatting, collapse excessive newlines, and unwrap hard-wrapped lines before sending text to the TTS API. This function handles the most common sources of garbled output:

**TypeScript**
```typescript
function preprocessText(raw: string): string {
  return raw
    // Remove markdown headers
    .replace(/^#{1,6}\s+/gm, '')
    // Remove markdown bold/italic
    .replace(/\*{1,3}([^*]+)\*{1,3}/g, '$1')
    // Remove markdown links, keep text
    .replace(/\[([^\]]+)\]\([^)]+\)/g, '$1')
    // Remove inline code backticks
    .replace(/`([^`]+)`/g, '$1')
    // Remove code blocks
    .replace(/```[\s\S]*?```/g, '')
    // Remove HTML tags
    .replace(/<[^>]+>/g, '')
    // Collapse 3+ newlines to 2
    .replace(/\n{3,}/g, '\n\n')
    // Unwrap hard-wrapped lines (single \n not preceded by sentence-ending punctuation)
    .replace(/([^.!?\n])\n([^\n])/g, '$1 $2')
    // Trim whitespace
    .trim();
}
```

Same preprocessing in Python using regex. Removes markdown syntax, HTML tags, and normalizes whitespace:

**Python**
```python
import re

def preprocess_for_tts(text: str) -> str:
    """Clean and format text for optimal TTS output."""
    # Remove markdown headers
    result = re.sub(r"^#{1,6}\s+", "", text, flags=re.MULTILINE)
    # Remove markdown bold/italic
    result = re.sub(r"\*{1,3}([^*]+)\*{1,3}", r"\1", result)
    # Remove markdown links, keep text
    result = re.sub(r"\[([^\]]+)\]\([^)]+\)", r"\1", result)
    # Remove inline code backticks
    result = re.sub(r"`([^`]+)`", r"\1", result)
    # Remove code blocks
    result = re.sub(r"```[\s\S]*?```", "", result)
    # Remove HTML tags
    result = re.sub(r"<[^>]+>", "", result)
    # Normalize whitespace within lines (preserve newlines)
    lines = result.split("\n")
    cleaned_lines = [" ".join(line.split()) for line in lines]
    result = "\n".join(cleaned_lines)
    # Collapse 3+ newlines to 2
    result = re.sub(r"\n{3,}", "\n\n", result)
    # Strip leading/trailing whitespace
    return result.strip()
```

## Long Text Considerations

For text longer than 4,096 characters, use the SDK's long-form generation which handles chunking automatically. The chunker splits at these boundaries in order of preference:

1. Paragraph breaks (`\n\n`)
2. Sentence endings (`.` `!` `?`)
3. Clause boundaries (`,` `;` `:`)
4. Word boundaries (spaces)
5. CJK character boundaries
6. Hard character limit (last resort)

Generate audio from text of any length. The SDK splits, generates, and concatenates automatically:

**TypeScript**
```typescript
const result = await client.speech.createLongForm({
  input: longArticle,
  voice: 'voice_abc123',
});
```

Same in Python, with optional control over chunk size and silence duration between chunks:

**Python**
```python
result = client.speech.create_long_form(
    input=book_chapter,
    voice="voice_abc123def456",
    chunk_size=3500,
    silence_between_chunks_ms=400,
)
```

## Language-Specific Tips

### Chinese / Japanese / Korean

CJK text does not use spaces between words. The model handles this natively. Use standard CJK punctuation for pacing control:

```python
text = "今天天气很好。我们去公园散步吧。\n\n公园里有很多花，非常漂亮。"
```

### German

German compound words (like Geschwindigkeitsbegrenzung) are pronounced correctly without manual splitting:

```python
text = "Die Geschwindigkeitsbegrenzung auf der Autobahn betragt manchmal zweihundert Kilometer pro Stunde."
```

### Dialogue

Use newlines to separate speaker turns. Double newlines create longer pauses between speakers:

```python
text = """"Good morning," said the receptionist. "How can I help you?"

"I have an appointment at ten," the visitor replied.

"Of course. Please take a seat and I'll let them know you're here."
"""
```

## Quick Reference

| Goal | Technique |
|------|-----------|
| Sentence pause | End with `.` `?` `!` |
| Short pause within sentence | Use `,` `;` `:` |
| Paragraph pause | Double newline `\n\n` |
| Dramatic pause | Use `...` or `--` |
| Emphasis | Use `!` or rephrase |
| Natural numbers | Spell them out |
| Abbreviations | Spell them out |
| Dialogue turns | Separate with newlines |
| Long text | Use `createLongForm()` / `create_long_form()` |

## See Also

- [Speech Generation](./speech.md) -- Sending text to the API
- [Voice Design](./voicedesign.md) -- Voice description best practices
- [Languages](./languages.md) -- Language-specific text considerations
