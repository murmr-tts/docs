# Languages

murmr supports 10 languages for text-to-speech generation. The `language` parameter is available on all generation endpoints and controls pronunciation, intonation, and prosodic patterns.

## Supported Languages

| Language | Parameter Value | Script |
|----------|----------------|--------|
| Chinese (Mandarin) | `Chinese` | Simplified/Traditional |
| English | `English` | Latin |
| French | `French` | Latin |
| German | `German` | Latin |
| Italian | `Italian` | Latin |
| Japanese | `Japanese` | Hiragana/Katakana/Kanji |
| Korean | `Korean` | Hangul |
| Portuguese | `Portuguese` | Latin |
| Russian | `Russian` | Cyrillic |
| Spanish | `Spanish` | Latin |

> Use full language names, not ISO codes. The `language` parameter is case-insensitive on the API but case-sensitive in the Python SDK. Always use the capitalized form (e.g., `"English"`, not `"english"` or `"en"`).

## Auto-Detection

When `language` is omitted or set to `Auto`, the API attempts to detect the language from the input text. The SDKs default to `English`.

> **When to set language explicitly:** Auto-detection works well for monolingual text but can misidentify short phrases or text with mixed-language content. Always set the language explicitly when you know it, especially for non-English text.

## Setting the Language

Set the `language` parameter to match the language of your input text. This controls pronunciation, intonation, and prosodic patterns:

**curl**
```bash
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Bonjour, comment allez-vous aujourd'\''hui?",
    "voice_description": "A warm French woman, mid-30s, Parisian accent",
    "language": "French"
  }' \
  --output french.wav
```

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const wav = await client.voices.design({
  input: 'Bonjour, comment allez-vous aujourd\'hui?',
  voice_description: 'A warm French woman, mid-30s, Parisian accent',
  language: 'French',
});

writeFileSync('french.wav', wav);
```

**Python**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = client.voices.design(
        input="Bonjour, comment allez-vous aujourd'hui?",
        voice_description="A warm French woman with a Parisian accent",
        language="French",
    )

    with open("french.wav", "wb") as f:
        f.write(wav)
```

## Language Examples

### Chinese

Generate Mandarin Chinese speech from simplified or traditional Chinese characters. The model handles tones natively:

```python
wav = client.voices.design(
    input="你好，欢迎来到我们的平台。今天的天气很好。",
    voice_description="A professional Chinese female news anchor",
    language="Chinese",
)
```

- Input should be in simplified or traditional Chinese characters
- Pinyin is not supported as input
- The model handles tone marks natively

### Japanese

Generate Japanese speech from kanji, hiragana, and katakana. No furigana needed -- the model reads kanji in context:

```python
wav = client.voices.design(
    input="こんにちは。本日のニュースをお伝えします。",
    voice_description="A polite Japanese male announcer",
    language="Japanese",
)
```

- Supports kanji, hiragana, and katakana
- Furigana is not needed; the model reads kanji correctly in context

### Korean

Generate Korean speech from standard hangul input. Use hangul equivalents instead of hanja (Chinese characters) for reliable pronunciation:

```python
wav = client.voices.design(
    input="안녕하세요. 오늘 좋은 하루 되세요.",
    voice_description="A cheerful Korean female voice",
    language="Korean",
)
```

- Standard hangul input
- Hanja (Chinese characters) may not be read correctly; use hangul equivalents

### European Languages

English, French, German, Italian, Portuguese, Russian, and Spanish all work with standard Unicode text. For best results, use proper diacritics and native punctuation conventions:

```python
# Portuguese with proper diacritics
wav = client.voices.design(
    input="Bom dia. Bem-vindo ao nosso servico.",
    voice_description="A warm Brazilian Portuguese female voice",
    language="Portuguese",
)

# German
wav = client.voices.design(
    input="Guten Morgen. Wie geht es Ihnen?",
    voice_description="A professional German male voice",
    language="German",
)

# Spanish
wav = client.voices.design(
    input="Hola, bienvenido a nuestra aplicacion.",
    voice_description="A friendly Spanish male voice",
    language="Spanish",
)
```

## Cross-Lingual Synthesis

murmr supports cross-lingual synthesis: use a voice designed with a description in one language to generate speech in another. The voice characteristics (tone, pace, timbre) transfer across languages.

Save a voice once and use the same voice ID across different languages by changing the `language` parameter:

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

// Design a voice with an English description
const refText = 'This is the reference audio for my multilingual voice.';
const wav = await client.voices.design({
  input: refText,
  voice_description: 'A professional male narrator, deep voice, calm delivery',
  language: 'English',
});

// Save it
const saved = await client.voices.save({
  name: 'Multilingual Narrator',
  description: 'Deep, calm male voice for multilingual content',
  audio: wav,
  ref_text: refText,
});

// Use the same voice in different languages
const languages = [
  { lang: 'Spanish', text: 'Bienvenidos a nuestra plataforma.' },
  { lang: 'German', text: 'Willkommen auf unserer Plattform.' },
  { lang: 'Japanese', text: 'プラットフォームへようこそ。' },
];

for (const { lang, text } of languages) {
  const stream = await client.speech.stream({
    input: text,
    voice: saved.id,
    language: lang,
  });
  for await (const chunk of stream) {
    // Process audio
  }
  console.log(`Generated ${lang} audio`);
}
```

**Python**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    description = "A warm, professional female narrator in her thirties"

    languages_and_texts = {
        "English": "Welcome to our global platform.",
        "French": "Bienvenue sur notre plateforme mondiale.",
        "German": "Willkommen auf unserer globalen Plattform.",
        "Spanish": "Bienvenido a nuestra plataforma global.",
        "Japanese": "私たちのグローバルプラットフォームへようこそ。",
    }

    for language, text in languages_and_texts.items():
        wav = client.voices.design(
            input=text,
            voice_description=description,
            language=language,
        )

        with open(f"welcome_{language.lower()}.wav", "wb") as f:
            f.write(wav)
        print(f"Generated: welcome_{language.lower()}.wav")
```

## Mixed-Language Text

For text containing multiple languages (e.g., English with occasional French words), set `language` to the dominant language. The model handles common loanwords and proper nouns naturally:

```python
wav = client.voices.design(
    input="The restaurant serves excellent creme brulee and pain au chocolat.",
    voice_description="An American woman with good French pronunciation",
    language="English",  # Dominant language
)
```

> For content that alternates significantly between languages, consider splitting into separate requests with the appropriate language set for each segment.

## Best Practices

| Practice | Reason |
|----------|--------|
| Always set `language` explicitly | Avoids auto-detection errors, especially for short text |
| Use native script | `Chinese` expects Chinese characters, not pinyin or romanization |
| Write numbers as words | "twenty-five" is more reliable than "25" across languages |
| Test voice descriptions in English | Voice Design descriptions work best in English, even for non-English output |
| Keep sentences natural length | Very short fragments (under 5 words) may have inconsistent prosody |

## See Also

- [Voice Design](./voicedesign.md) -- Voice descriptions and language
- [Speech Generation](./speech.md) -- The `language` parameter in API calls
- [Text Formatting](./text-formatting.md) -- How text structure affects prosody
