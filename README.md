# Converse — A Gaze-Ready Recipe Explorer

A hover-driven recipe explorer built in Unity 6 for the Cognixion take-home. Every interaction is designed to work without a click — users dwell on a tile until a fill ring completes, which stands in for BCI/gaze input throughout the app.

---

## What I built and why

**The flow:** Dietary constraints → interactive world map → AI-generated mood selection → full recipe with live ingredient substitutions.

The recipe domain is a deliberate choice: it has rich cultural depth, meaningful personalization axes (diet, spice, time, servings), and requires the LLM to do real reasoning — not just retrieval. More importantly, it maps cleanly onto the core BCI navigation problem: a user with limited motor control needs to move through a branching decision tree with small option sets, getting meaningful feedback at each step.

**Four scenes:**

| Scene | What happens |
|---|---|
| **01 Constraints** | Multi-select dietary tiles (vegetarian, gluten-free, etc.) |
| **02 World Map** | Procedural GeoJSON country meshes; hover to highlight; dwell to commit. Floating panel shows country name + AI blurb |
| **03 Mood** | 6 LLM-generated mood cards specific to the selected cuisine(s); right panel updates live on hover with flavor profile details |
| **04 Recipe** | Full generated recipe; hover any ingredient to surface 3 context-aware substitutions |

**Fusion mode:** selecting two countries on the map produces genuine cross-cultural fusion — the prompt enforces that core ingredients from one culture meet cooking techniques from the other, and the title must signal the fusion (e.g. "Miso-Harissa Braised Lamb").

---

## How to run

1. Clone the repo
2. Open `UnityProject/` in **Unity 6 LTS**
3. Copy `Assets/StreamingAssets/appsettings.example.json` → `appsettings.local.json` in the same folder
4. Paste your Anthropic API key (and optionally a Gemini key for the fallback) into `appsettings.local.json`
5. Press Play from the `01_Constraints` scene

> **No API key?** The app automatically falls back to a set of built-in recipes — the flow still works end-to-end.

---

## How the LLM is integrated

### Provider chain

```
Claude (claude-haiku-4-5, 12 s timeout)
    → Gemini (gemini-2.0-flash, v1beta) on timeout
        → CachedFallbackService (hardcoded offline data) on Gemini failure
```

`ClaudeRecipeService` races Claude against a 12-second `Task.Delay`. If Claude loses, it tries Gemini. A `429` from Gemini is caught and rethrown cleanly so the scene-level handler falls through to the cached service. The user always sees content — the fallback is not a dead end.

### Four LLM calls

**1. Country blurb** — fires as soon as a country is hovered on the map. Result is cached by ISO code so repeated hovers are instant.

```
System: you are a food writer; write one vivid sentence about this cuisine
User:   Country: Morocco
→       "Warm spices, slow-cooked tagines, and the sweet perfume of preserved lemon."
```

**2. Mood tiles** (`maxTokens: 1800`) — returns a structured JSON object with a `countryProfile` (summary, common bases, signature notes) and exactly 6 `MoodOption` objects (title, subtitle, panelTitle, panelBody). The country profile populates the right panel by default; each mood option populates it on hover.

```
System: [MoodSystem.txt — enforces fusion rules, single-country specificity, spice rules]
User:   Fusion countries: Japan + Morocco
        Dietary constraints: vegetarian
        Spice preference: mild
→       { "countryProfile": {...}, "moods": [{...}, ...] }
```

**3. Recipe** (`maxTokens: 1600`) — full recipe JSON (title, servings, timeMinutes, ingredients, steps). The fusion prompt requires the title to signal the blend and for both cuisines to contribute meaningfully across ingredients and steps.

**4. Substitutions** (`maxTokens: 256`) — given one ingredient and the rest of the recipe as context, returns 3 alternatives that make sense for the dish as a whole (not just generic swaps).

### Prompt design

- All four prompts return **strict JSON only** — no markdown, no prose wrapper. `StripCodeFence()` in `ClaudeRecipeService` handles the rare case where the model wraps output in ` ```json ` anyway.
- Cultural specificity is enforced in the prompt text: generic labels ("comforting stew") are called out as bad examples; region-specific ones ("ras el hanout braise", "charmoula street skewers") are shown as targets.
- `panelBody` is capped at 30 words in the mood prompt to keep responses well within the token budget.

### Prefetch chain

The app starts the next LLM call before the user has committed — using dwell time as a head start:

```
Map hover (t=0)     → start GenerateCountryBlurb (128 tokens, ~0.5 s)
Map commit (t=1 s)  → start GenerateMoodTiles in background
Mood scene loads    → consumes MoodTilesTask (usually already done)
Mood commit         → start GenerateRecipe in background
Recipe scene loads  → consumes RecipeTask (usually already done)
```

In practice, by the time the user dwells and confirms on a mood tile, the recipe is already fetched.

### BCI-readiness: `ISelectionInput`

All interaction routes through a single interface:

```csharp
public interface ISelectionInput
{
    event Action<string> OnHoverEnter;
    event Action<string> OnHoverExit;
    event Action<string> OnCommit;
}
```

Today's implementation is `MouseDwellInput`. Swapping in a BCI/eye-tracker implementation requires zero changes to any scene or UI code — the interface is the only contract. This is the architectural argument that this app is genuinely gaze-ready, not just gaze-themed.

`DwellTile` handles the visual side: radial fill ring, a deliberate color shift at ~60% fill (the "point of no return" cue used in real AAC design so users know they're about to commit), and a brief flash on confirmation.

---

## Tradeoffs and things I'd improve with more time

**Latency** — Claude Haiku is fast (~1–2 s for short calls) but the recipe call can take 3–4 s on a cold request. The prefetch chain masks most of this, but a production app would stream tokens and render the recipe progressively as they arrive. Unity's `UnityWebRequest` doesn't support streaming, so this would need a custom SSE reader.

**API key on device** — the key lives in a local JSON file that is gitignored. In a medical/clinical product this is unacceptable; the LLM calls would go through a backend proxy so the key never leaves the server. Noted in the architecture as a known gap.

**Hallucination and cultural accuracy** — the prompts anchor strongly to authentic regional staples, but the LLM can still invent plausible-sounding dishes. In production I'd layer a human-curated recipe bank: LLM generates creative options, a food-writer-reviewed bank validates and supplements them. Particularly important given how much cuisine representation flattens regional and diasporic variation.

**Dwell calibration** — dwell time is app-wide with one slider. Real motor-impairment apps run a short onboarding calibration sequence to find each user's comfortable dwell window, then adapt over time. The current slider is a minimal nod at this; a proper implementation would treat dwell time as a per-user profile stored persistently.

**No session persistence** — constraints and preferences reset on app quit. For a real BCI user who may be non-verbal, re-entering constraints every session is a real burden. A `PlayerPrefs`-backed session store would be a straightforward addition.

**The fallback data is thin** — `CachedFallbackService` has rich offline data for Morocco and Japan, and generic fallback data for all other countries. A production offline corpus would cover at least the 50 most commonly selected countries with manually authored content.

**Map pan/zoom** — the world map is fixed-view orthographic. Adding pan and pinch-zoom (via dwell on edge zones for BCI compatibility) would make the map much more usable for less prominent countries.

**Re-roll** — there's no "show me different moods" option if none of the 6 feel right. A single re-roll tile would be cheap to add and meaningfully improves the experience.

---

## Project structure

```
UnityProject/Assets/Scripts/
├── Claude/
│   ├── AppSettings.cs          # loads API keys from StreamingAssets
│   ├── ClaudeClient.cs         # Anthropic REST client (UnityWebRequest + coroutine)
│   └── GeminiClient.cs         # Gemini fallback client (v1beta, gemini-2.0-flash)
├── Input/
│   ├── ISelectionInput.cs      # BCI-swap abstraction
│   ├── MouseDwellInput.cs      # mouse implementation
│   └── DwellTile.cs            # fill ring, color cues, commit event
├── Map/
│   ├── GeoJsonParser.cs        # parses Natural Earth countries GeoJSON
│   ├── EarClipping.cs          # triangulates country polygons → meshes
│   ├── WorldMapController.cs   # hover highlight, floating panel, dwell-to-commit
│   └── CountryGeoData.cs       # country data model
├── Model/
│   ├── AppSession.cs           # singleton: constraints, country, mood, recipe
│   ├── Country.cs
│   ├── Recipe.cs / Ingredient.cs
│   └── MoodOption.cs           # MoodOption, CountryProfile, MoodTilesResult
├── Scenes/
│   ├── ConstraintsSceneBootstrap.cs
│   ├── MapSceneBootstrap.cs
│   ├── MoodSceneBootstrap.cs
│   └── RecipeSceneBootstrap.cs
└── Services/
    ├── IRecipeService.cs
    ├── ClaudeRecipeService.cs  # Claude→Gemini→cache fallback chain
    ├── CachedFallbackService.cs
    └── ScenePrefetch.cs        # holds in-flight Tasks between scenes

Assets/Resources/Prompts/
├── BlurbSystem.txt
├── MoodSystem.txt
├── RecipeSystem.txt
└── SubstitutionSystem.txt
```

---

## Demo

*(Screen recording coming — will be linked here once uploaded)*

---

## License

Private take-home project for Cognixion. Not for redistribution.
