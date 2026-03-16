# AI Intelligence — Deep Dive

> Three Bedrock models. Six languages. A context pipeline that makes every response personal.

[![Bedrock](https://img.shields.io/badge/Amazon_Bedrock-Nova_Chat_·_Text_·_Vision-7B2D8B?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/bedrock)
[![Transcribe](https://img.shields.io/badge/AWS_Transcribe-6_Indian_Languages-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/transcribe)
[![Polly](https://img.shields.io/badge/AWS_Polly-Neural_TTS-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/polly)

---

## AI Stack Overview

| Capability | AWS Service | Model | What It Does |
|-----------|------------|-------|-------------|
| Health coaching chat | Amazon Bedrock | Nova Chat | Context-aware Q&A, meal advice, proactive insights |
| Recipe generation | Amazon Bedrock | Nova Text | Personalised Indian recipes from pantry inventory |
| Food photo recognition | Amazon Bedrock | Nova Vision | Identify food items, estimate calories and macros |
| Voice input | AWS Transcribe | — | Speech-to-text in 6 Indian languages, auto-detected |
| Voice output | AWS Polly | Standard / Neural | Natural text-to-speech responses |

---

## 1. Context-Aware AI Chat

### Why Context Changes Everything

Most AI health apps send a user message to an LLM and get a generic response.
This product assembles a complete, personalised health context before every single AI call.

### The Context Assembly Pipeline

```
User sends message
        ↓
bedrock-ai-handler Lambda — parallel DynamoDB reads:

  ┌─────────────────────────────────────────────────────┐
  │ Read 1: User profile                                 │
  │   dietary_preference, health_goal, age, BMI         │
  │   allergies, locality, gender                       │
  ├─────────────────────────────────────────────────────┤
  │ Read 2: Shelf inventory                              │
  │   current items, quantities, expiry dates           │
  ├─────────────────────────────────────────────────────┤
  │ Read 3: Expiring items (next 7 days)                 │
  │   items with expiry_date < today + 7                │
  ├─────────────────────────────────────────────────────┤
  │ Read 4: Today's nutrition logs                       │
  │   calories consumed, protein, carbs, fat vs targets │
  ├─────────────────────────────────────────────────────┤
  │ Read 5: Recent meal history (3 days)                 │
  │   what was eaten, when, nutritional totals          │
  ├─────────────────────────────────────────────────────┤
  │ Read 6: Conversation history (last 10 messages)      │
  │   maintains continuity across the session           │
  └─────────────────────────────────────────────────────┘
        ↓
Dynamic system prompt constructed with all context
        ↓
Amazon Bedrock Nova Chat invoked
        ↓
Personalised, actionable, contextually relevant response
```

### Dynamic System Prompt Structure

```
You are an AI health and nutrition assistant specialised in Indian cuisine.

USER PROFILE:
- Dietary preference: {vegetarian | egg | non-veg | vegan}
- Health goal: {weight_loss | muscle_gain | maintenance}
- Age: {age} | Gender: {gender} | BMI: {bmi}
- Allergies/restrictions: {allergies}

CURRENT PANTRY ({item_count} items):
{shelf_items_with_quantities}

⚠ EXPIRING SOON (next 7 days):
{expiring_items_with_dates}

TODAY'S NUTRITION ({date}):
- Calories: {consumed} / {target} kcal
- Protein: {consumed}g / {target}g  {gap_indicator}
- Carbs: {consumed}g / {target}g
- Fat: {consumed}g / {target}g

RECENT MEALS (last 3 days):
{meal_history_summary}

GUIDELINES:
- Respond in the user's language
- Reference their actual shelf items and nutrition data
- Proactively flag expiring items and nutrition gaps
- Give specific, actionable advice — not generic tips
- Keep responses concise and practical
```

### What This Enables

| User asks | Generic AI says | This product says |
|-----------|----------------|------------------|
| "What should I eat for dinner?" | "Try a balanced meal with protein and vegetables" | "You have paneer and spinach expiring tomorrow — palak paneer gives you 28g protein, closing your gap for today. Puts you at 1,580 cal, right on target." |
| "How am I doing this week?" | "Track your meals regularly for best results" | "You're averaging 1,200 cal/day against a 1,600 target. Protein is consistently low — 28g vs 55g goal. Your dal rice lunches are good but add a protein source at breakfast." |
| "What can I cook tonight?" | "Here are some healthy recipe ideas..." | "You have tomatoes, onions, and chicken that need using. I can generate a chicken curry recipe — want me to?" |

---

## 2. AI Recipe Generation

### How It Works

```
User requests recipe (text or voice)
        ↓
recipe-handler Lambda
        ↓
Fetches user's shelf items from DynamoDB
        ↓
Constructs recipe generation prompt:
  - Available ingredients (from shelf)
  - Dietary type (veg/egg/non-veg/vegan)
  - Health goal (weight loss → lower calorie options)
  - Cuisine preference (regional Indian)
        ↓
Amazon Bedrock Nova Text invoked
        ↓
Structured recipe returned:
  - Name + description
  - Ingredients with quantities
  - Step-by-step instructions
  - Nutrition breakdown (calories, protein, carbs, fat)
  - Serving size
        ↓
Saved to kiro-recipes table
        ↓
Displayed in Flutter recipe detail screen
```

### Recipe Generation Prompt

```
Generate a {dietary_type} Indian recipe.

Available ingredients: {shelf_items}
Health goal: {health_goal}
Cuisine: Indian ({regional_preference})

Return as structured JSON:
{
  "name": "...",
  "description": "...",
  "prep_time": "...",
  "cook_time": "...",
  "servings": N,
  "ingredients": [{"item": "...", "quantity": "..."}],
  "instructions": ["step 1...", "step 2..."],
  "nutrition_per_serving": {
    "calories": N, "protein": Ng, "carbs": Ng, "fat": Ng
  }
}

Prioritise ingredients that are expiring soon: {expiring_items}
Keep it practical for a home cook. No exotic ingredients.
```

### Pre-Seeded Recipe Database

Beyond AI generation, the app ships with a curated Indian recipe database:
- Vegetarian: 100+ recipes (North and South Indian)
- Egg-based: 40+ recipes
- Non-vegetarian: 120+ recipes (chicken, mutton, fish)
- Vegan: 60+ recipes

All recipes include full nutrition data, regional tags, and difficulty levels.

---

## 3. Food Photo Recognition

### Vision Pipeline

```
User photographs a meal
        ↓
Flutter: image_picker → camera or gallery
        ↓
flutter_image_compress:
  - Max dimension: 1024px
  - Quality: 85%
  - Format: JPEG
        ↓
Image uploaded to S3:
  food-photos/{user_id}/{timestamp}.jpg
        ↓
nutrition-log-handler Lambda
        ↓
Image bytes passed to Bedrock Nova Vision:

  Prompt: "Identify all food items visible in this image.
           For each item provide:
           - Food name (use Indian names where applicable)
           - Estimated serving size
           - Calories per serving
           - Protein, carbohydrates, fat (grams)
           - Any common regional variations
           Return as structured JSON array."
        ↓
Nova Vision response parsed
        ↓
Bulk review screen — user confirms each item
        ↓
Confirmed items logged to kiro-nutrition-logs
```

### Why Nova Vision Works Well for Indian Food

Indian food is visually complex — a thali has 8+ items, curries look similar, regional variations are significant. Nova Vision's multimodal capability handles:
- Mixed-dish identification (biryani, pulao, khichdi)
- Thali decomposition (multiple items in one image)
- Regional dish recognition (idli vs uttapam, dosa variants)
- Portion size estimation from visual cues

---

## 4. Multilingual Voice Pipeline

### Language Coverage

| Language | Code | Speakers | Region |
|----------|------|---------|--------|
| Hindi | hi-IN | 528M | North India |
| English (Indian) | en-IN | 125M | Pan-India |
| Tamil | ta-IN | 75M | Tamil Nadu |
| Telugu | te-IN | 82M | Andhra Pradesh, Telangana |
| Malayalam | ml-IN | 38M | Kerala |
| Kannada | kn-IN | 44M | Karnataka |

**Total reach: ~892 million people — 65% of India's population**

Language is auto-detected by AWS Transcribe — users speak naturally, no selection needed.

### Voice Pipeline — End to End

```
Flutter AudioRecorder
  - Format: m4a (Android native)
  - Sample rate: 16kHz (Transcribe optimal)
  - Max duration: 30 seconds
        ↓
Audio uploaded to S3 via pre-signed URL
  voice-input/{user_id}/{timestamp}.m4a
        ↓
voice-handler Lambda (512MB, 30s timeout)
        ↓
AWS Transcribe batch job:
  - IdentifyLanguage: true
  - LanguageOptions: [en-IN, hi-IN, ta-IN, te-IN, ml-IN, kn-IN]
  - MediaFormat: auto-detected
  - OutputBucketName: S3 (transcripts/ prefix)
  Lambda polls every 500ms, max 25s
        ↓
Transcript + confidence score extracted
        ↓
bedrock-ai-handler invoked (full context assembly)
  - Response capped at 200 words (voice-optimised)
  - Intent detected: add_to_shelf | generate_recipe |
                     meal_plan | nutrition_query | general
        ↓
Response text hashed (SHA-256)
  Cache key: voice-cache/{hash}.mp3
  S3 HEAD check → cache hit or miss
        ↓
Cache miss → AWS Polly:
  - Free tier: Standard voice
  - Paid tiers: Neural voice (more natural, expressive)
  - Output: MP3 stream
  Audio stored in S3 cache (24h TTL)
        ↓
Response returned to Flutter:
  {transcript, response_text, audio_url, intent, detected_language}
        ↓
Flutter audioplayers streams audio from pre-signed URL
```

### Intent Detection

Every voice query returns a detected intent alongside the verbal response:

| Intent | Example | Next Action |
|--------|---------|------------|
| `add_to_shelf` | "Add onion to my shelf" | Returns item for shelf confirmation |
| `generate_recipe` | "Make a recipe with paneer" | Triggers recipe generation |
| `meal_plan` | "Add idli to tomorrow's breakfast" | Returns meal plan entry |
| `nutrition_query` | "How many calories in one roti?" | Returns nutrition data |
| `shelf_check` | "What's in my shelf?" | Returns inventory list |
| `general_health` | "Is brown rice better for weight loss?" | Returns health advice |

Full intent execution (auto-acting on voice commands without manual confirmation) is the next phase of development.

---

## 5. Cost Engineering for AI

Every AI call has a cost. The architecture is designed to minimise it without degrading experience.

### Voice Pipeline Cost Optimisations

| Optimisation | Mechanism | Saving |
|-------------|-----------|--------|
| Audio response caching | SHA-256 hash → S3 cache, 24h TTL | ~70% Polly cost reduction |
| Response length cap | 200 words max for voice responses | ~60% Polly character reduction |
| Batch transcription | Batch mode vs streaming | ~50% Transcribe cost reduction |
| Tier-based voice quality | Standard (free) vs Neural (paid) | Balances cost and experience |

### Bedrock Cost Optimisations

| Optimisation | Mechanism | Saving |
|-------------|-----------|--------|
| Context window limit | Last 10 messages only | Prevents token bloat |
| Response token cap | maxTokens: 500 | Bounded per-call cost |
| Reserved concurrency | Max 10 concurrent Bedrock calls | Prevents runaway costs |
| Usage limits per tier | Monthly AI query caps | Predictable cost per user |

### Per-Query Cost (Optimised)

| Component | Cost per query |
|-----------|--------------|
| AWS Transcribe (batch, 30s audio) | $0.0012 |
| Amazon Bedrock Nova Chat | $0.0005 |
| AWS Polly (cached 70% of time) | $0.0001–$0.0016 |
| Lambda + S3 | $0.0002 |
| **Total (free tier)** | **~$0.0023** |
| **Total (paid tier, Neural)** | **~$0.0035** |

At 1,000 active users with mixed usage: **~$130/month total infrastructure cost.**
