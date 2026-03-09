# Crafting Voice Descriptions

VoiceDesign creates voices from natural language descriptions. This guide shows you what works, using examples directly from the [Qwen3-TTS documentation](https://github.com/QwenLM/Qwen3-TTS).

> **💡 Try as You Read**
> Open the [Voice Playground](https://murmr.dev/en/dashboard/playground) in another tab. Copy any example from this guide and hear the result instantly.

## What You Can Control

According to the [Qwen3-TTS technical report](https://arxiv.org/abs/2601.15621), VoiceDesign supports “speech generation driven by natural language instructions for flexible control over timbre, emotion, and prosody.”

#### Benchmark Performance

From the arXiv paper, aggregate metrics on how well the model follows descriptions:

82-85%

APS (Attribute Perception)

81-82%

DSD (Description-Speech Consistency)

Chinese scores slightly higher than English. These are aggregate metrics.

## Official Examples

These examples come directly from the [Qwen3-TTS GitHub repository](https://github.com/QwenLM/Qwen3-TTS). They demonstrate the style and level of detail that works well.

Chinese

`“体现撒娇稚嫩的萝莉女声，音调偏高且起伏明显，营造出黏人、做作又刻意卖萌的听觉效果。”`

Translation: A coquettish, immature young female voice with high pitch and obvious fluctuations, creating a clingy, affected, and deliberately cute auditory effect.

Source: GitHub README

## Patterns That Work

Analyzing the official examples reveals consistent patterns for effective descriptions:

### ✓ What Official Examples Include

- **Character archetypes** — wizard, CEO, grandmother
- **Specific ages** — elderly, teenage, 17 years old
- **Emotional layers** — incredulous + panic
- **Pace descriptors** — slowly, quickly, measured
- **Purpose hints** — for bedtime stories
- **Vocal range** — tenor, high-pitched

### ✗ What to Avoid

- Celebrity references (“like Morgan Freeman”)
- Specific accent requests (“British accent”)
- Technical audio specifications (“16kHz sample rate”)
- Contradictory traits (“deep high-pitched voice”)
- Overly long descriptions (keep under 500 chars)

The model does not support accent or nationality control via voice descriptions. Use the `language` parameter instead to control the output language.

## Building Effective Descriptions

The official examples suggest a pattern: combine character + age + emotion + delivery. Here's how to construct your own:

```Pattern
[Character/Role] + [Age/Demographics] + [Vocal Quality] + [Emotional State] + [Delivery Style]

Examples:
"Professional male CEO voice" + "confident and authoritative" + "measured pace"
"Wise elderly wizard" + "deep, mystical voice" + "speaks slowly" + "with gravitas"
"Excited teenage girl" + "high-pitched" + "lots of energy" + "speaking quickly"
```

> You don't need all elements. The wizard example works because “wizard” already implies age and mystical qualities. Let the model infer what you don't specify.

## Supported Languages

VoiceDesign supports 10 languages for both the description and the output speech:

Source: [HuggingFace Model Card](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-VoiceDesign)

## Quick Reference

### Do

- • Use character archetypes when appropriate
- • Be specific about age when it matters
- • Layer emotions for nuanced performances
- • Include purpose hints for context
- • Describe pace and energy level

### Avoid

- • Celebrity impersonation requests
- • Contradictory traits
- • Technical audio specifications
- • Descriptions over 500 characters

## Sources

All claims in this guide are backed by official Qwen3-TTS documentation:

- [arXiv Technical Report](https://arxiv.org/abs/2601.15621) — Performance metrics, training details
- [GitHub Repository](https://github.com/QwenLM/Qwen3-TTS) — Official examples, API documentation
- [HuggingFace Model Card](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-VoiceDesign) — Supported languages, model specifications

## See Also

- [VoiceDesign API](./voicedesign.md) — Complete API reference
- [Style Instructions](./style-instructions.md) — Control delivery through voice descriptions
- [Voice Playground](https://murmr.dev/en/dashboard/playground) — Test descriptions before coding
