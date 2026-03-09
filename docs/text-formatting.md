# Text Formatting for Natural Speech

The way you format your input text directly affects how the voice sounds. Newline characters act as breathing and pause cues — use them intentionally for natural, human-like prosody.

## How Newlines Affect Prosody

The TTS model interprets newline characters as pause cues in the speech output. This mirrors how a human reader would naturally pause at line breaks in a script or printed text.

`\n`

Sentence-level pause

A single newline creates a short breath pause — like the natural gap between sentences when reading aloud. The voice briefly pauses, then continues with the same energy and tone.

`\n\n`

Paragraph-level pause

A double newline creates a longer pause with a prosodic reset — like the breath a reader takes between paragraphs. The voice resets its cadence, as if starting a new thought.

none

No newlines

A long block of text with no newlines sounds rushed. The model races through without natural breathing points, producing flat, unnatural delivery — especially noticeable on text longer than a few sentences.

## Before & After

The same text, formatted two ways. The difference in audio quality is immediately noticeable.

Raw text — rushed, no breathing

```text
The future of voice technology is here. Developers can now create any voice from a text description. No recording studios, no voice actors, no complicated setup. Just describe what you want and start generating. The API handles everything from streaming to batch processing, supporting ten languages out of the box.
```

Formatted text — natural pacing

```text
The future of voice technology is here.
Developers can now create any voice from a text description.
No recording studios, no voice actors, no complicated setup.

Just describe what you want and start generating.
The API handles everything from streaming to batch processing, supporting ten languages out of the box.
```

> **💡 Listen to the difference**
> Try both versions in the [Voice Playground](https://murmr.dev/en/dashboard/playground). The formatted version will have noticeably more natural pauses and emphasis.

## Best Practices

| Do | Don't |
| --- | --- |
| Insert \n between sentences (after . ! ?) | Send long text blocks with no newlines |
| Use \n\n between paragraphs or topic changes | Hard-wrap text at a fixed column width (e.g., 75 chars) |
| Let punctuation and newlines work together | Add \n after every clause or phrase |
| Pre-process copy-pasted text to fix line breaks | Paste raw text from PDFs, emails, or editors without cleanup |

## Common Pitfalls

### Copy-pasted text with hard line wraps

Text copied from PDFs, terminals, or plain-text emails often has newlines every 60-80 characters. The model treats each line break as a pause cue, producing choppy, flat delivery. Strip these artificial line breaks before sending to the API.

### Markdown or HTML formatting left in

Headings (`## Title`), bullet points (`- item`), and HTML tags will be spoken literally. Strip formatting to plain text before synthesis.

### Excessive newlines in dialogue

When generating dialogue, avoid adding `\n` after every short line. Short exchanges ("Yes." "I agree.") with newlines between each become stilted. Let natural sentence flow guide the pacing instead.

## Preprocessing Text

For the best results, normalize your input text before sending it to the API. Here's a simple preprocessing function:

**JavaScript**

```javascript
function formatForSpeech(text) {
  return text
    // Collapse hard line wraps (single newlines not after sentence endings)
    .replace(/([^.!?\n])\n(?!\n)/g, '$1 ')
    // Normalize multiple spaces
    .replace(/ {2,}/g, ' ')
    // Ensure sentence endings have a newline for natural pauses
    .replace(/([.!?])\s+(?=[A-Z])/g, '$1\n')
    // Preserve paragraph breaks (double newlines)
    .replace(/\n{3,}/g, '\n\n')
    .trim();
}
```

**Python**

```python
import re

def format_for_speech(text: str) -> str:
    # Collapse hard line wraps (single newlines not after sentence endings)
    text = re.sub(r'([^.!?\n])\n(?!\n)', r'\1 ', text)
    # Normalize multiple spaces
    text = re.sub(r' {2,}', ' ', text)
    # Ensure sentence endings have a newline for natural pauses
    text = re.sub(r'([.!?])\s+(?=[A-Z])', r'\1\n', text)
    # Preserve paragraph breaks (double newlines)
    text = re.sub(r'\n{3,}', '\n\n', text)
    return text.strip()
```

> **ℹ️ SDK chunking handles long text**
> The [Long-Form Audio](./long-form.md) feature in the SDK already splits text at sentence boundaries for requests over 4,096 characters. This guide is about formatting *within* each chunk for better prosody — they work together.

## Quick Reference

| Pattern | Effect | When to Use |
| --- | --- | --- |
| sentence.\nSentence. | Short breath pause | Between sentences in the same paragraph |
| paragraph.\n\nNew topic. | Long pause + prosodic reset | Between paragraphs or topic changes |
| No newlines at all | Rushed, flat delivery | Avoid for text longer than 1-2 sentences |
| \n every 60-80 chars | Choppy, robotic delivery | Never — strip hard wraps from pasted text |

## See Also

- [Style Instructions](./style-instructions.md) — Control voice personality and delivery via voice_description
- [Long-Form Audio](./long-form.md) — Automatic chunking for text over 4,096 characters
- [Crafting Voices](./voice-crafting.md) — Write effective voice descriptions
