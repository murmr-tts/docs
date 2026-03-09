# Crafting Voice Descriptions

VoiceDesign creates voices from natural language descriptions. This guide shows you what works, using examples directly from the [Qwen3-TTS documentation](https://github.com/QwenLM/Qwen3-TTS).

## What You Can Control

VoiceDesign supports "speech generation driven by natural language instructions for flexible control over timbre, emotion, and prosody" ([Qwen3-TTS technical report](https://arxiv.org/abs/2601.15621)).

### Demographics

- Age (elderly, young, 17 years old)
- Gender (male, female)

### Vocal Qualities

- Pitch (high-pitched, deep)
- Timbre (warm, bright, mystical)
- Vocal range (tenor, bass)

### Emotion and Mood

- Emotional states (excited, incredulous, joyful)
- Layered emotions (panic + incredulity)

### Delivery Style

- Speaking pace (slowly, quickly, measured)
- Energy level (enthusiastic, calm)
- Personality (confident, gentle, playful)

### Context and Purpose

- Use case hints (for bedtime stories)
- Character archetypes (wizard, CEO)

## Official Qwen3-TTS Examples

These examples come directly from the [Qwen3-TTS GitHub repository](https://github.com/QwenLM/Qwen3-TTS):

| Description | Use Case |
|-------------|----------|
| "A wise elderly wizard with a deep, mystical voice. Speaks slowly and deliberately with gravitas." | Fantasy narrator, audiobook |
| "Excited teenage girl, high-pitched voice with lots of energy and enthusiasm. Speaking quickly." | Energetic character, animation |
| "Professional male CEO voice, confident and authoritative, measured pace" | Business content, corporate |
| "Warm grandmother voice, gentle and soothing, perfect for bedtime stories" | Children's content, storytelling |
| "Speak in an incredulous tone, but with a hint of panic beginning to creep into your voice." | Emotional acting, drama |
| "Male, 17 years old, tenor range, gaining confidence - deeper breath support now, though vowels still tighten when nervous" | Age-specific character |

### Chinese Example

> "&#x4F53;&#x73B0;&#x6492;&#x5A07;&#x7A1A;&#x5AE9;&#x7684;&#x841D;&#x8389;&#x5973;&#x58F0;&#xFF0C;&#x97F3;&#x8C03;&#x504F;&#x9AD8;&#x4E14;&#x8D77;&#x4F0F;&#x660E;&#x663E;&#xFF0C;&#x8425;&#x9020;&#x51FA;&#x9ECF;&#x4EBA;&#x3001;&#x505A;&#x4F5C;&#x53C8;&#x523B;&#x610F;&#x5356;&#x840C;&#x7684;&#x542C;&#x89C9;&#x6548;&#x679C;&#x3002;"

Translation: A coquettish, immature young female voice with high pitch and obvious fluctuations, creating a clingy, affected, and deliberately cute auditory effect.

## Description Building Formula

Combine character + age + emotion + delivery:

```
[Character/Role] + [Age/Demographics] + [Vocal Quality] + [Emotional State] + [Delivery Style]

Examples:
"Professional male CEO voice" + "confident and authoritative" + "measured pace"
"Wise elderly wizard" + "deep, mystical voice" + "speaks slowly" + "with gravitas"
"Excited teenage girl" + "high-pitched" + "lots of energy" + "speaking quickly"
```

You don't need all elements. "Wizard" already implies age and mystical qualities. Let the model infer what you don't specify.

## Good vs Bad Examples

### Good

- Use character archetypes when appropriate
- Be specific about age when it matters
- Layer emotions for nuanced performances (e.g., "incredulous + panic")
- Include purpose hints for context (e.g., "for bedtime stories")
- Describe pace and energy level

### Bad

- Celebrity references ("like Morgan Freeman") -- not supported
- Specific accent requests ("British accent") -- use the `language` parameter instead
- Technical audio specifications ("16kHz sample rate")
- Contradictory traits ("deep high-pitched voice")
- Overly long descriptions (keep under 500 chars)

## Using Descriptions with the SDKs

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'fs';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const wav = await client.voices.design({
  input: 'Once upon a time, in a land far away...',
  voice_description: 'A wise elderly wizard with a deep, mystical voice. Speaks slowly and deliberately with gravitas.',
  language: 'English',
});

writeFileSync('wizard.wav', wav);
```

**Python (sync)**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = client.voices.design(
        input="Once upon a time, in a land far away...",
        voice_description="A wise elderly wizard with a deep, mystical voice. Speaks slowly and deliberately with gravitas.",
        language="English",
    )

    with open("wizard.wav", "wb") as f:
        f.write(wav)
```

**Python (async)**
```python
import asyncio
import os
from murmr import AsyncMurmrClient

async def main():
    async with AsyncMurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
        wav = await client.voices.design(
            input="Once upon a time, in a land far away...",
            voice_description="A wise elderly wizard with a deep, mystical voice. Speaks slowly and deliberately with gravitas.",
            language="English",
        )

        with open("wizard.wav", "wb") as f:
            f.write(wav)

asyncio.run(main())
```

## Supported Languages

VoiceDesign supports 10 languages for both the description and the output speech:

Chinese, English, Japanese, Korean, German, French, Russian, Portuguese, Spanish, Italian

## Sources

- [arXiv Technical Report](https://arxiv.org/abs/2601.15621) -- Performance metrics, training details
- [GitHub Repository](https://github.com/QwenLM/Qwen3-TTS) -- Official examples, API documentation
- [HuggingFace Model Card](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-VoiceDesign) -- Supported languages, model specifications

## See Also

- [Style Instructions](./style-instructions.md) -- Control delivery through voice descriptions
- [OpenAI Migration](./openai-migration.md) -- Migrating from OpenAI's fixed voices
