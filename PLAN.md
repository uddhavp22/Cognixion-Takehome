# Converse — Build Plan & Checklist

A Unity 6 desktop app for the Cognixion take-home. Hover-driven recipe explorer with an LLM that proposes options at every step. Mouse stands in for BCI input throughout — the whole UX is designed for dwell-to-commit, no clicks.

---

## 1. Concept (the one-paragraph version)

A world-map recipe explorer where the user navigates from constraints → country → cooking mood → a full recipe, choosing at each step from a small set of LLM-suggested options by hovering on a tile until a dwell ring fills. On the recipe screen, hovering over any ingredient surfaces three context-aware substitutions the LLM generates with the rest of the dish in mind. The whole app is designed around the constraint that every interaction must work without a click — same UX whether driven by mouse, gaze, or BCI.

**Why this for Cognixion:** demonstrates hover-driven exploration of a rich content space (the core BCI navigation problem) with the LLM doing real cognitive work — generating culturally-grounded options, reasoning about ingredient substitutions, and adapting to dietary constraints. The recipe domain is a delightful Trojan horse for the actual design thesis.

---

## 2. The flow

```
[1] Constraints  →  [2] World Map  →  [3] Mood  →  [4] Recipe + Substitutions
    (multi-          (hover-highlight    (5 LLM-          (full recipe; hover
     select tiles)    countries; floating  generated         any ingredient to
                      blurb; dwell to      tiles per         dwell-commit a sub)
                      commit)              country)
```

Persistent across all screens: a quiet "back" tile (dwell to go up one level), an "active constraints" indicator at top, and a settings toggle for dwell-time.

---

## 3. Architecture

```
┌────────────────────────────────────────────┐
│           Scene-per-screen flow            │
│   Constraints → Map → Mood → Recipe        │
└────────────────────────────────────────────┘
                      ↕
┌────────────────────────────────────────────┐
│           AppSession (singleton)           │
│  - selectedConstraints: List<string>       │
│  - selectedCountry: Country                │
│  - selectedMood: string                    │
│  - currentRecipe: Recipe                   │
└────────────────────────────────────────────┘
        ↕                          ↕
┌─────────────────────┐    ┌──────────────────────┐
│  ISelectionInput    │    │  IRecipeService      │
│  - MouseDwellInput  │    │  - ClaudeRecipe      │
│    (future: BCI)    │    │  - CachedFallback    │
└─────────────────────┘    └──────────────────────┘
                                  ↕
                         ┌──────────────────────┐
                         │  ClaudeClient (HTTP) │
                         │  - blurb cache       │
                         │  - rate limiting     │
                         └──────────────────────┘
```

**Two abstractions that matter for the interview story:**

- `ISelectionInput` — same interface for mouse-dwell today and BCI tomorrow. Zero UI changes when input source changes. This is *the* architectural argument that this app is BCI-ready.
- `IRecipeService` — Claude today, but the interface is `Task<Recipe> Generate(...)`. A `CachedFallback` implementation ships 10 pre-baked recipes so the demo never fails on network flakiness. In a real AAC/BCI product, the user must always be able to *function* when the LLM fails — this is the ethics piece.

---

## 4. LLM integration

**Provider:** Anthropic, `claude-haiku-4-5`. Fast (2x+ faster than Sonnet), cheap ($1/$5 per M tokens), strong enough for everything in this app. Endpoint: `https://api.anthropic.com/v1/messages`.

**Four endpoints, each with its own prompt:**

1. `generateCountryBlurb(country)` → 1-line flavor description for the floating map panel. Cached aggressively after first fetch.
2. `generateMoodTiles(country, constraints)` → 5 mood options specific to that cuisine. Returns structured JSON.
3. `generateRecipe(country, mood, constraints)` → full recipe JSON (title, servings, time, ingredients, steps). The big call (~2-4s).
4. `generateSubstitutions(ingredient, recipeContext)` → 3 alternatives reasoning about the rest of the dish.

**Prompt design notes:**
- All return strict JSON; no prose around it. Use `system` to enforce schema.
- Cultural anchor in every prompt: "use authentic, widely-recognized regional staples; if unsure, default to the most common version."
- For substitutions: include the *rest of the recipe* as context so suggestions account for flavor/technique compatibility.

**Auth:** API key from a local `appsettings.local.json` (gitignored) or env var. Never hardcoded, never committed. README will note that production would proxy through a backend so the user's machine never sees the key — important for any medical-adjacent product.

---

## 5. Dwell mechanic

Every interactive element is a `DwellTile`:
- On hover-enter, start a fill timer (default 800ms, configurable).
- Show a radial fill ring (`Image.type = Filled`, radial360) as it progresses.
- On hover-exit, reset to 0 (forgiving — accidental fly-bys don't commit).
- At ~60% fill: visible "point of no return" cue (color shift + slight scale). Real AAC design — user is *consciously holding* until commit.
- On full fill: fire `OnCommit(tileId)`, play subtle confirm sound, brief flash.

Dwell time per screen (tunable in settings):
- Constraints: 600ms (low-stakes toggles)
- Map: 1000ms (committing to a country is a real choice)
- Mood: 800ms
- Recipe substitutions: 800ms

---

## 6. Scope

### Must-have
- Unity 6 project setup with `.gitignore`
- `ISelectionInput` + `MouseDwellInput` + `DwellTile` prefab
- `SessionState` singleton
- 4 scenes: Constraints, Map (stubbed early as tile grid), Mood, Recipe
- Claude API client + 4 endpoints with JSON schemas
- `CachedFallback` recipe service with ~10 pre-baked recipes
- End-to-end flow working with one country (Morocco) before generalizing
- README + screen recording

### Should-have
- Real world map: GeoJSON parsing + procedural country meshes + hover highlighting + floating info panel
- Country blurb cache
- Substitution panel on recipe screen
- Settings: dwell-time slider, reset constraints
- Mac + Windows builds

### Stretch
- Pan/zoom on the map
- Saved-recipes screen
- BCI-sim mode (cursor jitter, variable dwell)
- Per-region ambient sound

---

## 7. Tradeoffs / things to call out in the README

1. **The LLM can hallucinate dishes/ingredients.** Mitigation: prompts strongly anchor to authentic, well-documented regional staples. In production: human-curated recipe banks.
2. **Cultural sensitivity.** "The cuisine of [country]" flattens what's actually regional/diasporic/contested. Acceptable in a prototype; in production, partner with food writers from each region.
3. **API key in a local file.** Production would proxy through a backend.
4. **Dwell time is one-size-fits-all.** Real motor-impairment apps calibrate per-user with onboarding. The settings slider is a token nod to this.
5. **No persistence.** Constraints don't survive app restart.
6. **The map is the biggest visual risk.** Mitigation: build it last, after the rest of the app works with a stub.

---

## 8. The checklist

### Step 1 — Repo & Unity project setup
- [x] Create GitHub repo `converse` (private to start, can flip public later)
- [x] Install Unity 6 LTS via Unity Hub
- [x] Create new Unity 6 project
- [x] Add Unity-specific `.gitignore`
- [ ] First commit: empty project + gitignore + README stub
- [ ] Push to GitHub

### Step 2 — Claude API client + secrets
- [ ] Create `Assets/Scripts/Claude/` folder
- [ ] Add `ClaudeClient.cs`: async HTTP client
  - POST to `https://api.anthropic.com/v1/messages`
  - Headers: `x-api-key`, `anthropic-version: 2023-06-01`, `content-type: application/json`
  - Body: `{ "model": "claude-haiku-4-5", "max_tokens": 1024, "system": "...", "messages": [{"role": "user", "content": "..."}] }`
  - Parse response, return `content[0].text`
- [ ] Add `AppSettings.cs`: loads API key from `appsettings.local.json` at `StreamingAssets/` (gitignored)
- [ ] Add `appsettings.local.json` to `.gitignore` and to `StreamingAssets/`
- [ ] Add `appsettings.example.json` (committed, shows the schema with a placeholder key)
- [ ] Smoke test: temp scene with one button that fires `client.SendMessage("say hi")` and prints to console
- [ ] Commit + push

### Step 3 — Selection input abstraction + DwellTile
- [ ] Create `Assets/Scripts/Input/` folder
- [ ] `ISelectionInput.cs` interface:
  ```csharp
  public interface ISelectionInput {
    event Action<string> OnHoverEnter;
    event Action<string> OnHoverExit;
    event Action<string> OnCommit;
    void Tick(float deltaTime);
    float DwellSeconds { get; set; }
  }
  ```
- [ ] `MouseDwellInput.cs` implementation
- [ ] `DwellTile.cs` MonoBehaviour with radial fill ring, 60% color shift, commit flash
- [ ] Prefab: `Assets/Prefabs/DwellTile.prefab`
- [ ] Test scene: 3 dwell tiles, each logs "committed: {id}" on full fill
- [ ] Commit + push

### Step 4 — SessionState + IRecipeService + CachedFallback
- [ ] `Assets/Scripts/Model/`:
  - `Country.cs` — `{ string name; string iso; }`
  - `Recipe.cs` — `{ string title; int servings; int timeMinutes; List<Ingredient> ingredients; List<string> steps; }`
  - `Ingredient.cs` — `{ string name; string amount; string substitutionNotes; }`
  - `AppSession.cs` — singleton holding constraints/country/mood/recipe
- [ ] `Assets/Scripts/Services/`:
  - `IRecipeService.cs` — methods for the 4 LLM calls (return `Task<T>`)
  - `ClaudeRecipeService.cs` — calls `ClaudeClient`, parses JSON
  - `CachedFallbackService.cs` — hardcoded recipes for ~10 countries
- [ ] Fallback: try Claude first, fall back to cache on error/timeout
- [ ] 4 prompts as `.txt` files in `Resources/Prompts/`
- [ ] Commit + push

### Step 5 — Constraints scene
- [ ] New scene `Scenes/01_Constraints.unity`
- [ ] Grid of 9 DwellTiles: vegetarian, vegan, pescatarian, gluten-free, dairy-free, nut allergy, shellfish allergy, low-carb, no restrictions
- [ ] Each tile toggles a constraint in `SessionState` on commit (filled = active)
- [ ] "Continue" tile → loads Map scene
- [ ] Commit + push

### Step 6 — Map stub (temporary)
- [ ] New scene `Scenes/02_Map.unity`
- [ ] 6 DwellTiles: Morocco / Japan / Mexico / India / Italy / Peru
- [ ] Commit country to `SessionState` → load Mood scene
- [ ] (Replaced in step 9)
- [ ] Commit + push

### Step 7 — Mood scene
- [ ] New scene `Scenes/03_Mood.unity`
- [ ] On load: call `IRecipeService.GenerateMoodTiles(country, constraints)`
- [ ] Loading spinner while waiting
- [ ] Render 5 mood DwellTiles from response
- [ ] Commit → save mood to state, load Recipe scene
- [ ] Commit + push

### Step 8 — Recipe scene
- [ ] New scene `Scenes/04_Recipe.unity`
- [ ] On load: call `IRecipeService.GenerateRecipe(country, mood, constraints)`
- [ ] Loading state while waiting
- [ ] Layout: title + serving info top, ingredients left, steps right
- [ ] Each ingredient row is a DwellTile
- [ ] On ingredient commit: call `GenerateSubstitutions`, show panel with 3 sub DwellTiles
- [ ] Commit sub → swap into recipe, re-render
- [ ] "Back to Map" tile, "New recipe (same settings)" tile
- [ ] Commit + push

### Step 9 — Real world map (replaces step 6 stub)
- [ ] Download Natural Earth countries GeoJSON (admin-0, low-res) → `Assets/StreamingAssets/world.geojson`
- [ ] `MapLoader.cs`: parse GeoJSON, generate procedural mesh per country
  - `MeshCollider` for raycasting
  - `CountryHover` component with ISO code as tile id
- [ ] Orthographic top-down camera fit to map bounds
- [ ] Hover: raycast mouse, highlight country mesh
- [ ] Floating panel: country name + LLM blurb (cached)
- [ ] Pre-warm blurbs for top ~20 countries on start
- [ ] Dwell on country → commits + loads Mood scene
- [ ] Commit + push

### Step 10 — Settings + polish
- [ ] Settings overlay: dwell-time slider (400ms–2000ms), reset-constraints button
- [ ] Accessible via "gear" DwellTile on each screen
- [ ] Audio: confirm sound on commit, ambient background loop
- [ ] Visual polish: typography, color, scene transitions
- [ ] Commit + push

### Step 11 — Builds
- [ ] Mac build → `Builds/Mac/`
- [ ] Windows build → `Builds/Windows/`
- [ ] Both gitignored; zip for README release links

### Step 12 — README
- [ ] What you built and why
- [ ] How to run (clone, add `appsettings.local.json`, open in Unity 6, play)
- [ ] LLM integration details (4 endpoints, structured JSON, fallback service, prompt design)
- [ ] The BCI angle (`ISelectionInput` abstraction, dwell mechanics, low option count)
- [ ] Tradeoffs / what I'd do with more time
- [ ] EEG / future BCI integration paragraph

### Step 13 — Recording
- [ ] 60–90 second screen recording: constraints → map → country → mood → recipe → substitution swap
- [ ] Upload to YouTube unlisted or Loom; link from README

### Step 14 — Submit
- [ ] Final commit + push
- [ ] Make repo public (or invite collaborator)
- [ ] Reply email with repo link, recording link, binary links
- [ ] EOD Monday 2026-05-18

---

## 9. Time budget

| Day | Focus |
|---|---|
| Fri night | Steps 1–3: setup, API client, dwell mechanic in test scene |
| Saturday | Steps 4–7: state/services, Constraints, map stub, Mood |
| Sunday | Step 8: Recipe scene with substitutions |
| Monday morning | Step 9: real world map |
| Monday afternoon | Steps 10–13: settings, polish, builds, README, recording |
| Monday EOD | Step 14: submit |

**Fallback:** if the real map is going sideways Monday morning, ship the stub and polish everything else. Working app with stubbed map > half-working app with beautiful map.

---

## 10. Open questions

- TTS / read-aloud of the final recipe? (Skip from v1; mention in README as next step.)
- Saved recipes / favorites? (Stretch — needs persistence.)
- Re-roll mood tiles if none feel right? (Cheap to add — "show me different moods" tile.)
- Edit constraints mid-flow without going back to start? (Probably yes — inline editor via dwell on constraints indicator.)
