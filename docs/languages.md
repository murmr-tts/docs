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
