# Converse

A recipe explorer built in Unity 6 where every interaction with no clicks (hover the mouse). 

---

## What it is

You pick dietary constraints, hover over a country on a world map, choose a cooking mood from AI-generated options, and get a full recipe. Hover any ingredient in the recipe to get three substitution suggestions. 


I've always loved cooking different cultures food as well trying to make fusion food, so this project was also an excuse for me experiment with new food! This project also has enough depth to make the LLM do real work (cultural specificity, dietary rules, fusion cooking) while keeping each step's decision space small enough to work well with dwell input. It can be especially helpful because BCI users need tight option sets to be efficient.

**Scenes:**

| | |
|---|---|
| **01 Constraints** | Dietary tiles — vegetarian, gluten-free, etc. Toggle on/off by dwelling |
| **02 World Map** | Procedural country meshes from GeoJSON. Hover highlights a country and pulls an AI blurb; dwell commits it |
| **03 Mood** | 6 AI-generated mood cards for the cuisine. Right panel updates on hover with flavor profile info |
| **04 Recipe** | Full recipe. Hover any ingredient to get substitution options |

If you select two countries, the app switches into fusion mode — the prompts enforce that both cuisines actually contribute to the dish rather than one being an afterthought.

---

## How to run

1. Clone the repo (includes the `UnityProject` submodule — use `git clone --recurse-submodules`)
2. Open `UnityProject/` in Unity 6 LTS
3. Copy `Assets/StreamingAssets/appsettings.example.json` → `appsettings.local.json` in the same folder and add your Anthropic API key
4. Hit Play from the `01_Constraints` scene

If you don't have an API key it falls back to hardcoded recipes.
---

## LLM integration

**Model:** `claude-haiku-4-5` via the Anthropic API. Fast and cheap enough that latency isn't painful for a real-time UI.

**Fallback chain:** Claude (12s timeout) → Gemini `gemini-2.0-flash` (v1beta) → hardcoded local data. If Claude is slow or down, Gemini picks up. If Gemini returns a 429, the scene falls through to the cached service.

**Four calls:**

1. **Country blurb** — one sentence about the cuisine, fires on hover, cached per ISO code after the first fetch
2. **Mood tiles** — returns a `countryProfile` (summary, common bases, signature notes) plus 6 `MoodOption` objects. One call populates both the default panel and all the hover states
3. **Recipe** — full JSON: title, servings, time, ingredients, steps. Token budget is 1600 to avoid truncation on longer recipes
4. **Substitutions** — 3 alternatives for a given ingredient, with the rest of the recipe included as context so suggestions actually fit the dish

All prompts return strict JSON — no prose, no markdown wrapper. There's a `StripCodeFence()` fallback for when the model wraps output anyway. 

**Prefetch:** the app fires the next call before the user commits, using dwell time as a free head start. Mood tiles start loading when a country is committed on the map; the recipe starts loading when a mood is committed.

**`ISelectionInput`:** all interaction goes through one interface with `OnHoverEnter`, `OnHoverExit`, and `OnCommit`. The current implementation is mouse dwell. Swapping in eye tracking or a BCI SDK means writing one new class.
---

## What I'd do differently with more time

**Streaming** — the recipe call takes 2–4s on a cold request. The prefetch hides most of it but streaming tokens would let the recipe render progressively,from what I understand  `UnityWebRequest` doesn't support SSE so this would probably need a custom reader.

**API key handling** — right now the key lives in a gitignored local file. Which is fine for a prototype, but not fine for a clinical product. In production the LLM calls go through a backend proxy.

**Cultural accuracy** — the prompts push hard for regional specificity but the LLM can still hallucinate plausible-sounding dishes. A real version would pair the LLM with a curated recipe bank reviewed by food writers from each region or check online recipies, especially for cuisines that tend to get flattened or misrepresented.

**Persistence** — nothing survives an app restart. For a non-verbal user having to re-select dietary constraints every session is a real friction point. `PlayerPrefs` would be a quick fix.

**Fallback depth** — the offline data is detailed for Morocco and Japan and generic for everything else. A proper offline corpus would need real coverage.

---
## Structure

```
UnityProject/Assets/Scripts/
├── Claude/          ClaudeClient, GeminiClient, AppSettings
├── Input/           ISelectionInput, MouseDwellInput, DwellTile
├── Map/             GeoJsonParser, EarClipping, WorldMapController
├── Model/           AppSession, Country, Recipe, MoodOption
├── Scenes/          one bootstrap script per scene
└── Services/        IRecipeService, ClaudeRecipeService, CachedFallbackService, ScenePrefetch

Assets/Resources/Prompts/
    BlurbSystem.txt, MoodSystem.txt, RecipeSystem.txt, SubstitutionSystem.txt
```

---

## Demo

https://github.com/uddhavp22/Cognixion-Takehome/raw/main/media/demo.mov

<video src="media/demo.mov" controls width="100%"></video>
