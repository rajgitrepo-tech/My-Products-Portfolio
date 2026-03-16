# Cojaisy — AI & Voice Pipeline

## AI Stack

| Component | Service | Model | Use Case |
|-----------|---------|-------|---------|
| Conversational AI | Amazon Bedrock | Nova Chat | Health Q&A, meal advice, context-aware chat |
| Recipe Generation | Amazon Bedrock | Nova Text | AI-generated Indian recipes from shelf items |
| Food Recognition | Amazon Bedrock | Nova Vision | Photo-based food identification + calorie estimation |
| Speech-to-Text | AWS Transcribe | — | Voice input in 6 Indian languages |
| Text-to-Speech | AWS Polly | Standard / Neural | Voice responses (tier-based quality) |

---

## Supported Languages

| Language | Code | Voice (Free) | Voice (Paid) |
|----------|------|-------------|-------------|
| English (Indian) | en-IN | Standard | Neural |
| Hindi | hi-IN | Standard | Neural |
| Tamil | ta-IN | Standard | Neural |
| Telugu | te-IN | Standard | Neural |
| Malayalam | ml-IN | Standard | Neural |
| Kannada | kn-IN | Standard | Neural |

Language is auto-detected by AWS Transcribe using `IdentifyLanguage` mode
with the 6 supported codes as hints — no user selection required.

---

## Bedrock Prompt Engineering

### System Prompt (Dynamic per User)

```
You are Cojaisy, an AI health and nutrition assistant specialised in Indian cuisine.

User Profile:
- Dietary preference: {vegetarian | egg | non-veg | vegan}
- Health goal: {weight_loss | muscle_gain | maintenance}
- Age: {age}, Gender: {gender}

Current shelf items: {shelf_items_list}
Recent meals (last 3 days): {meal_history}

Guidelines:
- Respond in the same language the user writes in
- Keep responses under 200 words for voice delivery
- Prioritise Indian food context (regional cuisines, local ingredients)
- Suggest recipes using available shelf items when relevant
- Provide calorie estimates for Indian food items
- Be conversational, warm, and encouraging
```

### Recipe Generation Prompt

```
Generate a {dietary_type} Indian recipe using these available ingredients: {shelf_items}.
Health goal: {health_goal}.
Format: Name, Ingredients (with quantities), Steps (numbered), Nutrition (calories, protein, carbs, fat).
Keep it practical for a home cook.
```

### Food Recognition Prompt

```
Identify this Indian food item in the image.
Provide: food name, estimated serving size, calories per serving,
macronutrients (protein, carbs, fat), and any common regional variations.
If multiple items are visible, list each separately.
```

---

## Voice Pipeline — Detailed Flow

### Input Processing

```
1. Flutter AudioRecorder captures audio
   - Format: m4a (Android default)
   - Sample rate: 16kHz (optimal for Transcribe)
   - Max duration: 30 seconds

2. Audio uploaded to S3
   - Bucket: kiro-food-photos-{env}
   - Key: voice-input/{user_id}/{timestamp}.m4a
   - Pre-signed URL used for upload (no public access)

3. voice-handler Lambda invoked via API Gateway
   - Receives: S3 key, user_id, language_hint (optional)
```

### Transcription

```
4. AWS Transcribe batch job started
   - MediaFormat: auto-detected (m4a/wav/webm/ogg)
   - IdentifyLanguage: true
   - LanguageOptions: [en-IN, hi-IN, ta-IN, te-IN, ml-IN, kn-IN]
   - OutputBucketName: same S3 bucket (transcripts/ prefix)

5. Lambda polls for job completion (max 25s, 500ms intervals)

6. Transcript extracted from JSON output
   - Confidence score logged to CloudWatch
   - Low confidence (<0.7) flagged in response
```

### AI Processing

```
7. bedrock-ai-handler invoked (Lambda-to-Lambda)
   - User context fetched from DynamoDB (shelf, profile, recent meals)
   - Dynamic system prompt constructed
   - Bedrock Nova Chat invoked
   - Response limited to 200 words (voice-optimised)
   - Intent detection: add_to_shelf | generate_recipe | meal_plan | general_query
```

### Audio Response

```
8. Response text hashed (SHA-256)
   - Cache key: voice-cache/{hash}.mp3
   - S3 HEAD request to check cache existence

9. Cache miss → AWS Polly invoked
   - Free tier: Standard voice (en-IN: Aditi, hi-IN: Aditi, etc.)
   - Paid tiers: Neural voice (more natural, expressive)
   - Output: MP3 audio stream

10. Audio stored in S3 cache
    - TTL: 24 hours (S3 lifecycle rule)
    - Identical responses reuse cached audio

11. Response returned to Flutter
    - transcript: what Cojaisy heard
    - response_text: AI answer
    - audio_url: pre-signed S3 URL (5-min expiry)
    - intent: detected action type
    - detected_language: what Transcribe identified
```

### Flutter Playback

```
12. Flutter receives response
    - Displays transcript text
    - Displays AI response text
    - audioplayers package streams audio from pre-signed URL
    - Playback controls: play/pause/stop
    - Language indicator shown if non-English detected
```

---

## Cost Optimisations

### Audio Caching
- Identical AI responses reuse cached Polly audio
- Cache hit rate: ~70% (common health questions repeat)
- Saving: ~$0.0011 per cached query (Polly cost avoided)

### Response Length Limiting
- AI responses capped at 200 words
- Reduces Polly character count by ~60%
- Saving: ~$0.0006 per query

### Batch Transcription
- Batch mode vs streaming: 50% cheaper
- Acceptable latency for voice assistant use case (2-4s total)
- Saving: ~$0.0006 per query

### Tier-Based Voice Quality
- Free tier: Standard voices ($4/1M chars)
- Paid tiers: Neural voices ($16/1M chars)
- Free users get good quality; paid users get premium
- Saving: ~$0.0012 per free-tier query

### Total per-query cost (optimised)
| Tier | Transcribe | Bedrock | Polly | Lambda + S3 | Total |
|------|-----------|---------|-------|-------------|-------|
| Free | $0.0012 | $0.0005 | $0.0004 | $0.0002 | **$0.0023** |
| Paid | $0.0012 | $0.0005 | $0.0016 | $0.0002 | **$0.0035** |

---

## Intent Detection

The voice AI detects user intent and returns a structured action alongside the verbal response:

| Intent | Example Utterance | Action |
|--------|------------------|--------|
| `add_to_shelf` | "Add onion to my shelf" | Returns item details for manual add |
| `generate_recipe` | "Make a recipe with paneer" | Triggers recipe generation flow |
| `meal_plan` | "Add idli to tomorrow's breakfast" | Returns meal plan entry for confirmation |
| `nutrition_query` | "How many calories in one roti?" | Returns nutritional data |
| `shelf_check` | "What's in my shelf?" | Returns shelf item list |
| `general_health` | "Is brown rice better for weight loss?" | Returns health advice |

Phase 3 (in progress) will auto-execute `add_to_shelf`, `generate_recipe`, and `meal_plan` intents
without requiring manual confirmation in the app.

---

## Food Photo Recognition Flow

```
User taps camera icon in food log screen
        ↓
Flutter image_picker → camera or gallery
        ↓
flutter_image_compress → resize to max 1024px, quality 85%
        ↓
Image uploaded to S3 (food-photos/{user_id}/{timestamp}.jpg)
        ↓
POST /nutrition-logs with S3 key
        ↓
nutrition-log-handler Lambda
        ↓
Bedrock Nova Vision invoked with image bytes
        ↓
Returns: food name, calories, macros, serving size
        ↓
Pre-populated nutrition log entry returned to Flutter
        ↓
User reviews and confirms (bulk review screen for multiple items)
```

---

## Conversation Context Management

Chat sessions maintain context across messages:

```
kiro-conversations table
  conversation_id (PK)
  user_id (GSI)
  created_at
  last_message_at
  message_count

kiro-messages table
  message_id (PK)
  conversation_id (GSI)
  user_id
  role: user | assistant
  content
  timestamp
```

On each new message:
1. Last N messages fetched from kiro-messages (N=10 for context window)
2. Passed to Bedrock as conversation history
3. New response appended to history
4. Context window keeps AI responses relevant to the ongoing conversation
