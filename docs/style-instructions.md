# Style Instructions

Control how voices deliver your content using natural language descriptions. With VoiceDesign, you describe the emotion, pace, and tone directly in the `voice_description` field.

## How Style Works

Style control happens through the `voice_description` parameter in VoiceDesign. Write what you want the voice to sound like -- including personality, emotion, pace, and delivery -- and the model will adjust accordingly.

Generate audio with style instructions by embedding emotion, pace, and delivery cues in the `voice_description` field:

**curl**
```bash
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Welcome to the future of AI.",
    "voice_description": "An excited male narrator, dramatic delivery, building anticipation",
    "language": "English"
  }' --output welcome.wav
```

Same request using the Node.js SDK. The `voice_description` controls both voice identity and delivery style:

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const wav = await client.voices.design({
  input: 'Welcome to the future of AI.',
  voice_description: 'An excited male narrator, dramatic delivery, building anticipation',
  language: 'English',
});
```

**Python**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = client.voices.design(
        input="Welcome to the future of AI.",
        voice_description="An excited male narrator, dramatic delivery, building anticipation",
        language="English",
    )

    with open("welcome.wav", "wb") as f:
        f.write(wav)
```

> **VoiceDesign vs Saved Voices:** Style instructions are part of VoiceDesign's `voice_description`. When you find a style you like, save the voice to reuse it with a stable `voice_xxx` ID in the `/v1/audio/speech` endpoint.

## Style Categories

### Professional

For business and formal content:

- `calm and professional`
- `confident and authoritative`
- `clear and measured`
- `formal and precise`

### Emotional

Express feelings and energy:

- `excited and enthusiastic`
- `sympathetic and caring`
- `urgent and intense`
- `joyful and upbeat`

### Pace

Control speaking speed:

- `speak slowly and deliberately`
- `rapid and energetic delivery`
- `natural conversational pace`
- `thoughtful pauses between phrases`

### Tone

Set the overall character:

- `warm and friendly`
- `serious and formal`
- `playful and light`
- `mysterious and intriguing`

## Combining Styles

For more nuanced delivery, combine multiple descriptors. The model understands complex, layered instructions:

| Description | Effect |
|-------------|--------|
| "A warm, friendly woman speaking at a relaxed pace, with occasional pauses for emphasis" | Combines character, tone (warm), pace (relaxed), and technique (pauses) |
| "Professional male narrator, approachable, clear enunciation, confident delivery" | Balances formality with accessibility |
| "A mysterious storyteller with a deep voice, slow build-up, dramatic pauses" | Creates narrative arc through character + delivery style |
| "Excited news anchor announcing breaking news, urgent but clear" | Uses role-play style instruction |

## Use Case Examples

These `voice_description` values combine character and style for common use cases:

- **Product Demo:** "A clear, professional male voice, enthusiastic but measured, highlighting key features"
- **Meditation Guide:** "A warm, soothing female voice, calm and gentle, very slow pace, whisper-like quality"
- **Movie Trailer:** "A deep, powerful male voice with gravitas, epic and dramatic, building intensity"
- **Kids Educational:** "A bright, energetic young woman, playful and animated, speaking clearly, fun and engaging"
- **Podcast Intro:** "A friendly, conversational male voice, natural delivery, welcoming host energy"

## Save and Reuse Voices

Once you find a voice description that captures the right style, save it for consistent production use. The saved voice preserves the style from your description, so every generation with that voice ID has the same character and delivery:

**TypeScript**
```typescript
// 1. Design with style descriptors
const wav = await client.voices.design({
  input: 'This is my reference audio.',
  voice_description: 'A calm, professional narrator, measured pace, clear enunciation',
  language: 'English',
});

// 2. Save the voice
const saved = await client.voices.save({
  name: 'Professional Narrator',
  description: 'Calm male narrator, measured pace',
  audio: wav,
  ref_text: 'This is my reference audio.',
});

// 3. Use in production
const stream = await client.speech.stream({
  input: 'Your content here.',
  voice: saved.id,
});
```

**Python**
```python
# 1. Design a voice with style descriptors
description = "A calm, professional narrator, measured pace, clear enunciation"
text = "This is a reference sample for voice saving."

wav = client.voices.design(
    input=text,
    voice_description=description,
)

# 2. Save the voice
saved = client.voices.save(
    name="Professional Narrator",
    audio=wav,
    description=description,
    ref_text=text,
)

# 3. Use in production with the stable voice ID
result = client.speech.create_and_wait(
    input="Your content here.",
    voice=saved.id,
)
```

Saved voices capture the style from your description -- every generation with that voice ID will have the same character and delivery.

## Tips for Effective Descriptions

- **Be descriptive, not prescriptive.** Describe the feeling you want ("warm and inviting") rather than technical instructions ("lower pitch by 10%").
- **Include both character and delivery.** The best descriptions combine who the speaker is ("elderly wizard") with how they speak ("slowly and deliberately"). See the [Voice Crafting Guide](./voice-crafting.md) for more patterns.
- **Keep it under 500 characters.** 2-3 descriptors usually work better than long, complex instructions. Focus on the most important characteristics.

## See Also

- [Voice Crafting](./voice-crafting.md) -- Detailed patterns for effective voice descriptions
- [OpenAI Migration](./openai-migration.md) -- Replacing OpenAI's fixed voices with VoiceDesign
