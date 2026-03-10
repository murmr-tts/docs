# Language Support

murmr supports 10 languages with native-quality synthesis. VoiceDesign works in any language with automatic detection available.

## Supported Languages

Pass the full language name as a string in the `language` parameter.

Language Value

Regions

> **⚠️ Use full names, not ISO codes**
> The API expects full English names like `"English"`, `"Spanish"`, `"German"`. ISO 639-1 codes like `en`, `es`, `de` are not recognized and will fall back to auto-detection.

## Auto-Detection

When you set `language: "Auto"` (the default), murmr automatically detects the language from your input text.

```JSON
{
  "text": "Bonjour, comment allez-vous?",
  "voice_description": "A warm, friendly voice"
  // language defaults to "Auto" — will detect French
}
```

> **💡 When to use Auto**
> Auto-detection works well for single-language content. For mixed-language text or when you need precise control, explicitly set the language parameter.

## Explicit Language Setting

For precise control, specify the language explicitly:

```JSON
{
  "text": "Bienvenidos a nuestra plataforma.",
  "voice_description": "A professional narrator",
  "language": "Spanish"
}
```

This ensures the text is synthesized with Spanish phonetics and pronunciation, regardless of auto-detection.

## Cross-Lingual Synthesis

You can describe a voice in one language and generate speech in another. The voice description is always interpreted by the model, while the `language` parameter controls the output language.

```JSON
{
  "text": "東京タワーへようこそ。",
  "voice_description": "A warm, friendly female voice with a gentle tone",
  "language": "Japanese"
}
```

> **ℹ️ Voice Description + Language**
> The `voice_description` controls the speaker's character (age, gender, emotion, delivery) while the `language` parameter controls the output language. See the [Voice Crafting Guide](./voice-crafting.md) for effective description patterns.

## Accents and Regional Variations

The `language` parameter controls pronunciation and phonetics. The `voice_description` parameter controls the speaker's character (age, gender, emotion, delivery style) but **does not control accent or regional variation**.

> **⚠️ Accents are not controllable**
> Descriptions like "British accent", "Parisian accent", or "Southern drawl" are ignored by the model. The model was not trained on accent-specific data. To control the output language and its phonetics, use the `language` parameter.

### Language-Specific Tips

#### French

- ✓ Set `language: "French"` explicitly — auto-detection may confuse short French phrases with other Romance languages
- ✓ Focus `voice_description` on vocal qualities: "A warm, articulate female narrator, mid-30s, smooth and professional"
- ✗ Do not request accents like "Parisian" or "Québécois" — these are ignored. Use `language: "French"` and let the model handle pronunciation
- ✓ French liaison and elision are handled automatically when `language: "French"` is set

#### Spanish

- ✓ Set `language: "Spanish"` — covers both Castilian and Latin American phonetics
- ✗ Cannot distinguish between regional Spanish varieties (e.g., Mexican vs. Argentinian) via voice description

#### Chinese

- ✓ Supports Mandarin (Simplified and Traditional). Voice descriptions can be written in Chinese for best results
- ✓ Chinese descriptions score slightly higher on attribute perception benchmarks (82-85% APS)

#### English

- ✓ Best-supported language for voice descriptions — most examples in official documentation are in English
- ✗ Cannot distinguish between regional English varieties (US, UK, Australian) via voice description

### Complete French Example

Creating a French narrator voice — focus on vocal qualities, not accent:

```JSON
{
  "text": "Bienvenue dans notre podcast. Explorons ensemble le futur de l'intelligence artificielle.",
  "voice_description": "A warm, articulate female narrator, mid-30s, smooth and professional",
  "language": "French"
}
```

The `language: "French"` parameter ensures French phonetics (liaison, nasal vowels, uvular R). The `voice_description` controls the speaker's character — warm tone, professional delivery, mid-30s age range.

## Best Practices

Set language explicitly for production

Auto-detection adds a small amount of uncertainty. For production apps where the content language is known, always set it explicitly.

Use explicit language for proper nouns

Text with many proper nouns, brands, or technical terms benefits from explicit language setting to avoid auto-detection guessing incorrectly.

Test with your actual content

Different voice + language combinations produce varying results. Use the [Playground](https://murmr.dev/en/dashboard/playground) to find the best match for your content.

## API Examples

German content with explicit language:

```JSON
{
  "text": "Willkommen bei murmr. Wir freuen uns, Sie zu sehen.",
  "voice_description": "A professional German narrator",
  "language": "German"
}
```

Japanese content:

```JSON
{
  "text": "こんにちは、murmrへようこそ。",
  "voice_description": "A friendly Japanese woman",
  "language": "Japanese"
}
```

Saved voice with explicit language:

```JSON
{
  "text": "Bienvenidos a nuestro servicio.",
  "voice": "voice_abc123",
  "language": "Spanish"
}
```

## See Also

- [VoiceDesign API](./voicedesign.md) — Create voices with natural language descriptions
- [Voice Crafting Guide](./voice-crafting.md) — Write effective voice descriptions
