# Pocket Pal — Master Build Prompt for Roblox Studio

> Paste everything below the line into Roblox Studio Assistant (or Cursor / Claude with Rojo).
> Work **one phase at a time**: paste Sections 0–7 once as context, then say "Build Phase 1", test, then "Build Phase 2", and so on.
> Verified against create.roblox.com docs on 2026-11-02.

---

## 0. Role and ground rules

You are a senior Roblox game developer and Luau engineer. You ship clean, secure, server-authoritative games that are fair to players. You are building **Pocket Pal**, a cozy pet-companion simulator, in the **current release of Roblox Studio (November 2026)**, using only **current, non-deprecated Engine APIs** from the Creator Hub API reference (create.roblox.com/docs/reference/engine).

### Non-negotiable coding rules

1. **Luau, strictly typed.** Every ModuleScript and Script starts with `--!strict`. Define shared types in `ReplicatedStorage.Shared.Types`. No `any` unless it is unavoidable and commented.
2. **Modern task library only.** Use `task.wait`, `task.spawn`, `task.defer` and `task.delay`. Never use `wait()`, `spawn()`, `delay()` or `tick()`. Use `os.clock()` for durations and `os.time()` / `DateTime.now()` for timestamps.
3. **No deprecated APIs.** In particular:
   - `MarketplaceService:GetProductInfoAsync` — **not** `GetProductInfo` (deprecated).
   - `MarketplaceService:UserOwnsGamePassAsync` — **not** `PlayerOwnsAsset` (deprecated).
   - `TextService:FilterStringAsync`, then `TextFilterResult:GetNonChatStringForBroadcastAsync()` — **not** `FilterAndTranslateStringAsync` (deprecated).
   - Use `Instance:GetAttribute` / `SetAttribute` instead of `ValueBase` objects for simple replicated state.
   - Use the **constraint-based movers** (`AlignPosition`, `AlignOrientation`, `LinearVelocity`) — never `BodyPosition`, `BodyGyro` or `BodyVelocity`.
   - Use the **Input Action System** (`InputContext`, `InputAction`, `InputBinding`) for custom inputs — not `ContextActionService`.
4. **The server is authoritative.** The client only *requests* actions. The server validates everything: rate limits, cooldowns, ownership, distance, argument types and value ranges. Never trust a client-sent price, item ID, quantity, XP or position without checking it.
5. **Every network or yielding call is wrapped** in `pcall` with retry and exponential backoff where appropriate. This includes DataStore, MarketplaceService, TextService, PolicyService and `GetProductInfoAsync`.
6. **No magic numbers.** All tuning values (decay rates, XP curve, cooldowns, prices shown in Coins, product and pass IDs) live in `ReplicatedStorage.Shared.Config`.
7. **Small, single-responsibility modules.** Use a services-and-controllers pattern without a third-party framework: server `*Service` modules and client `*Controller` modules, each with an explicit `Init()` and `Start()`, booted by one bootstrap script per side.
8. **Clean up connections.** Track `RBXScriptConnection`s per player and per pet, and disconnect them on `PlayerRemoving` and pet despawn. Call `Destroy()` on instances you create.
9. **Mobile first.** Assume most players are on phones. Every interaction must work by touch, mouse/keyboard and gamepad.
10. **After each phase, stop.** List any manual Studio or Creator Dashboard steps I must do, such as creating passes, pasting IDs or enabling API access. Then give me the acceptance-test checklist for that phase. Do not start the next phase until I say so.

---

## 1. Game design spec

**Pitch:** Adopt a cute creature, name it, feed it, play with it, teach it tricks and decorate its home. As your friendship grows, you unlock new behaviours and looks. A shared park lets players show off their pals.

**Core loop:** Feed → Play → Walk → earn **Bond XP** → level up → unlock tricks, outfits and furniture → show off in the park and visit friends' houses.

**Tone and art direction** (matching the concept art): soft pastel palette (mint, coral, butter yellow, cream), rounded chunky shapes, a sunny park with blocky trees, lamp posts and a fountain. The UI uses big rounded pill buttons with white outlines and friendly icons.

### 1.1 Scope (v1 — start small)

| Area | v1 contents |
|---|---|
| Species | **1**: the "Sprout Pup" — a mint-green, puppy-like creature with small coral horns, a leafy tail and paw-print spots. |
| Activities | **3**: Feed, Play (fetch ball), Walk. |
| Locations | **1** private house per player (instanced plot) + **1** shared park. |
| Progression | Bond levels 1–20, 5 tricks, about 12 free cosmetics, about 12 paid cosmetics. |

### 1.2 Pet needs (gentle, never punishing)

- Three needs from 0–100: **Hunger**, **Fun** and **Energy**.
- Needs decay slowly while online (see `Config.Needs`). Offline decay is **capped**: a pet never drops below 40 while the player is away.
- A pet **never dies, runs away or gets sick.** At low needs it shows a sad emote and earns 50% Bond XP. That is the only penalty.
- Show needs as three small rounded bars above the action buttons.

### 1.3 Activities

| Activity | Player action | Effect | Cooldown | Bond XP |
|---|---|---|---|---|
| **Feed** | Tap Feed → choose a food from the inventory → the pet eats (animation plus heart particles). | Hunger +35 | 20 s | +10 |
| **Play** | Tap Play → a ball spawns → the player taps or clicks a target spot (or presses the throw action) → the pet fetches it → 3 throws per session. | Fun +30, Energy −10 | 30 s | +5 per catch, +5 bonus for 3/3 |
| **Walk** | Tap Walk → the pet goes on a leash → walk through glowing paw checkpoints (5 per route, 2 routes in the park). | Energy +20 (a happy-tired rest afterwards), Fun +10 | per route, 120 s | +4 per checkpoint, +10 on completion |

- Basic food (Kibble) is **free and unlimited**. Better food (Berries, Treat Cake) is bought with **Coins** (soft currency), which are earned through play only.
- **Coins:** earned from activities (+5 to +15) and a daily login gift. Coins are **never sold for Robux** in v1. This keeps the economy transparent for a young audience.

### 1.4 Bond progression

- XP to reach level `n`: `Config.Bond.xpForLevel(n) = math.floor(60 * n ^ 1.45)`.
- A soft daily cap: after 600 Bond XP per day, XP gains fall to 50% (not zero). Reset at 00:00 UTC.
- Unlocks:

| Level | Unlock |
|---|---|
| 2 | Trick: **Sit** |
| 3 | Free outfit: Yellow Paw Bandana (as in the concept art) |
| 4 | Trick: **Spin** |
| 5 | Pet bed recolours (free) |
| 6 | Trick: **Wave** |
| 8 | Trick: **Roll Over** |
| 10 | Behaviour: the pet follows with a happy bounce and greets friends' pets |
| 12 | Trick: **Dance** (synced with the owner's `/e dance`) |
| 15 | Free outfit: Flower Sun Hat |
| 20 | "Best Friends" title and a sparkle trail (free) |

- On level-up, show a full-screen celebratory banner for 2 seconds, plus a confetti particle burst and a sound. Pause cooldowns briefly.

### 1.5 Tricks

- Unlocked tricks appear in a **Tricks wheel** (a radial menu). Selecting one plays the trick animation on the pet, visible to everyone nearby.
- Training: the first 3 performances of a newly unlocked trick each give +15 Bond XP (a "learning" bonus).
- Server-side cooldown: 4 seconds between tricks.

### 1.6 Naming

- On first join, an adoption cutscene shows the pet hatching from a leafy pod, then a name prompt (max 16 characters).
- Filter the name with `TextService:FilterStringAsync(name, userId, Enum.TextFilterContext.PublicChat)`, then `result:GetNonChatStringForBroadcastAsync()`. Store the **filtered** string. If it comes back fully hashed (`####`), reject it and ask again. Renaming is free once every 24 hours.

### 1.7 The house (private plot)

- Each player gets an instanced plot (pre-built "Cottage" template cloned from `ServerStorage.Templates.House`), placed on one of 8 plot pads around the park, or in a reserved-server space. Use plot pads for v1 so friends can walk over.
- **Decorate mode:** grid placement (1-stud grid, 90° rotation) inside the house's bounding region only. Place, move, rotate and store furniture. The client previews the placement; the server validates the bounds, ownership, the item limit (60) and overlap, then commits.
- House privacy setting: **Open / Friends only / Closed**. The server enforces it with a door barrier and kicks visitors back to the park if the setting changes.

### 1.8 The shared park

- A central hub with a fountain, benches, lamp posts, two walk routes and a fetch lawn.
- **Show-off features:**
  - "Pal Spotlight" stage: stand on it to show your pet's name, Bond level and outfit on a floating card (`BillboardGui`).
  - Pet greetings: when two pets above level 10 come within 8 studs, they play a greet animation.
  - A "Visit" button on any player's card: teleport to the door of their house, subject to their privacy setting.
- Up to 24 players per server.

---

## 2. Monetisation (fair, cosmetic-first)

### 2.1 Principles

- **Never paywall care or progression.** Feeding, playing, walking, Bond XP, tricks and the base house are 100% free.
- **No paid random items** (no paid eggs or loot boxes). Every purchase shows exactly what you get.
- **No pay-to-win and no Robux-for-Coins** in v1.
- No dark patterns: no fake timers, no "only 2 left" claims, no pop-ups during activities. The shop opens only when the player taps Shop.
- Paid items look great on the pet, **which follows the player everywhere**. That visibility is the main reason to buy.
- Revenue share: Roblox gives the creator **70%** of the Robux spent on the game's own passes and developer products.

### 2.2 Product types and how to use each

| What | Roblox type | Why |
|---|---|---|
| **Extra Outfit Slots** (from 3 saved looks to 10) | **Pass** | A one-time permanent benefit. |
| **Themed Bedroom packs** (Cloud Dream, Candy Cottage, Starry Night: walls, floor, window, rug, lamp) | **Pass** (one per pack) | A permanent bundle, which suits a pass. |
| **Individual pet costumes** (Bee Hoodie, Strawberry Hat, Dino Onesie, Wizard Cloak, and so on) | **Developer product**, **granted once** and stored in the inventory | There are many SKUs, and passes don't scale to dozens of items. A dev product plus your own ownership record is the standard pattern. The server must block re-purchase of an owned item. |
| **Fancy beds** (Cloud Bed, Mushroom Bed, Royal Cushion) | **Developer product**, granted once | Same reasoning as costumes. |
| **Supporter Pack** (optional, later) | Pass | A badge, a name colour and an exclusive scarf. |

### 2.3 Server implementation requirements

**Game passes (`MonetizationService`, server):**
- On `PlayerAdded`, check `MarketplaceService:UserOwnsGamePassAsync(userId, passId)` for every pass in `Config.Passes`, concurrently, with `pcall` and retry. Cache the results in the player's session.
- Listen to `MarketplaceService.PromptGamePassPurchaseFinished`. Grant the benefit only if `wasPurchased == true` and the pass ID is in `Config.Passes`.
- Re-verify ownership with `UserOwnsGamePassAsync` before applying any pass benefit, rather than trusting the event alone.
- Record pass ownership in the player profile for fast loading, but treat `UserOwnsGamePassAsync` as the source of truth.

**Developer products (`ProcessReceipt`):**
- Assign `MarketplaceService.ProcessReceipt` **exactly once**, from one server script.
- The handler must be **idempotent**. Roblox's own sample does not guard against duplicate grants, so you must:
  1. Look up the player via `Players:GetPlayerByUserId(receiptInfo.PlayerId)`. If they're not in the server, return `Enum.ProductPurchaseDecision.NotProcessedYet`.
  2. Make sure their profile is loaded and session-locked by this server. If not, return `NotProcessedYet`.
  3. If `receiptInfo.PurchaseId` is already in `profile.PurchaseLog`, return `PurchaseGranted` immediately without granting again.
  4. Map `receiptInfo.ProductId` to a handler in `Config.Products`. Unknown ID: log it and return `NotProcessedYet`.
  5. Grant the item and append `PurchaseId` to the log **in the same profile mutation**. Force an immediate save. Only return `PurchaseGranted` after the save succeeds. Otherwise return `NotProcessedYet`.
  6. Keep `PurchaseLog` bounded to the last 100 IDs (oldest dropped).
- Log `receiptInfo.CurrencySpent` for analytics. It reflects the actual price paid, including during price tests.
- **Block re-buying owned costumes and beds:** before calling `MarketplaceService:PromptProductPurchase`, the server checks the inventory. The client shop shows owned items as "Owned ✓" with a disabled buy button.

**Prices in the UI:**
- Never hard-code Robux prices. Fetch them with `MarketplaceService:GetProductInfoAsync(id, Enum.InfoType.Product)` (or `Enum.InfoType.GamePass`) and cache them per server for 10 minutes. This keeps the shop correct when **Regional Pricing** or **Price Optimization** changes prices.

**Policy compliance:**
- On join, call `PolicyService:GetPolicyInfoForPlayerAsync(player)` with `pcall`. Respect `ArePaidRandomItemsRestricted`; v1 has none, but wire the check for future-proofing. Hide any external-link references the policy disallows.

### 2.4 Shop UI

- Tabs: **Costumes · Beds · Bedrooms · Perks**.
- Each card shows a 3D preview (`ViewportFrame` with the player's own pet wearing the item, slowly rotating), the item name, the price fetched live, and **Buy** or **Owned ✓** / **Equip**.
- **Try on:** tapping a card previews the costume on the real pet for 10 seconds, visible only to the local player.
- Purchase flow: tap Buy → the native Roblox prompt → on success, show a "New look unlocked!" toast and auto-equip.

### 2.5 Suggested launch prices (tune with Price Optimization later)

| Item | Robux |
|---|---|
| Individual costume | 49–99 |
| Fancy bed | 79–149 |
| Themed bedroom pack (pass) | 199–299 |
| Extra Outfit Slots (pass) | 149 |
| Supporter Pack (pass) | 399 |

---

## 3. Architecture

### 3.1 Explorer layout

```
ReplicatedStorage
  Shared/
    Config            (ModuleScript — all tuning, IDs, item catalogue)
    Types             (ModuleScript — exported Luau types)
    ItemCatalog       (ModuleScript — every cosmetic/food/furniture item def)
    Signal            (ModuleScript — tiny typed signal)
    Net               (ModuleScript — creates/gets RemoteEvents & RemoteFunctions by name)
    Util/             (Retry, Maid/Trove-style cleanup, Math helpers)
  Inputs/
    PlayContext       (InputContext)  → ThrowBall, OpenTricks, Interact
    MenuContext       (InputContext)  → Close, TabLeft, TabRight
  Assets/
    Pets/SproutPup    (Model with Animator + animation IDs as attributes)
    Cosmetics/        (Accessory-style models with an attachment name attribute)
    Furniture/
    VFX/

ServerScriptService
  Bootstrap.server    (requires and Init()/Start() all services)
  Services/
    DataService       (profiles, session locking, save/autosave, migration)
    PetService        (spawn/despawn, follow, needs tick, animations)
    ActivityService   (Feed/Play/Walk validation and rewards)
    BondService       (XP, levels, unlocks, daily cap)
    TrickService
    InventoryService  (Coins, items, equip, outfits)
    HouseService      (plot assignment, decorate validation, privacy)
    ParkService       (spotlight, greetings, visit teleports)
    MonetizationService (passes, ProcessReceipt, prices cache)
    PolicyServiceWrap
    NameService       (filtering)
    RateLimiter       (per-player, per-remote token bucket)

ServerStorage
  Templates/House
  Templates/Pets

StarterPlayer/StarterPlayerScripts
  Bootstrap.client
  Controllers/
    UIController, PetController (client-side cosmetic smoothing only),
    ActivityController, DecorateController, ShopController, TricksController,
    InputController, SoundController, CameraController (adoption cutscene)

StarterGui
  HUD (ScreenGui), Shop, Wardrobe, Tricks, Decorate, Settings, Adoption, Toasts
```

### 3.2 Networking contract

Every remote lives in `ReplicatedStorage.Remotes` and is created by the server through `Net`. The client sends **intents only**. The server validates types with explicit `typeof` checks plus range and ownership checks, and enforces `RateLimiter` (default: 8 requests per second per remote, with tighter per-action cooldowns from Config).

| Remote | Direction | Payload | Server validation |
|---|---|---|---|
| `RequestFeed` | C→S | `foodId: string` | food in catalogue, owned or free, cooldown, pet spawned |
| `RequestPlayStart` / `RequestThrow` | C→S | `target: Vector3` | session active, target within 40 studs of player, ≤3 throws |
| `RequestWalkStart` | C→S | `routeId: string` | route exists, cooldown |
| `ReachedCheckpoint` | C→S | `index: number` | server checks distance from player HRP to checkpoint ≤ 8 studs and order |
| `RequestTrick` | C→S | `trickId: string` | unlocked, cooldown |
| `SetPetName` | C→S | `name: string` | length, rename cooldown, filter |
| `EquipItem` / `SaveOutfit` / `LoadOutfit` | C→S | ids/slot | owned, slot count ≤ entitlement |
| `PlaceFurniture` / `MoveFurniture` / `StoreFurniture` | C→S | `itemId, cframe` / `uid, cframe` | owned, within plot bounds, grid-snapped, no overlap, ≤60 items |
| `SetHousePrivacy` | C→S | enum string | one of 3 values |
| `RequestVisit` | C→S | `targetUserId: number` | target in server, privacy allows (friends via `Player:IsFriendsWith` with pcall) |
| `RequestPurchase` | C→S | `productKey: string` | known key, not already owned → server calls the prompt |
| `StateUpdate` | S→C | partial profile diff | — |
| `Notify` | S→C | toast payload | — |

Replicate per-pet visible state (name, level, needs, equipped cosmetics) as **attributes on the pet model**, so all clients see it without extra remotes.

### 3.3 Data (DataService)

- Use `DataStoreService:GetDataStore("PocketPal_v1")` with key `"player_" .. userId`.
- **Session locking:** use `UpdateAsync` to write `session = { jobId = game.JobId, lockedAt = os.time() }`. If a different, recent lock exists (less than 30 minutes old), retry up to 5 times with backoff, then load anyway and steal the lock, logging a warning. Release the lock on leave. All writes use `UpdateAsync` (never `SetAsync`) and verify the lock is still ours inside the transform function. The transform function **must not yield**.
- Pass `{ userId }` as the `UserIds` argument (via `DataStoreSetOptions` or the `UpdateAsync` return) so data-deletion requests can be honoured.
- Autosave every 120 s (jittered ±15 s), on `PlayerRemoving`, and in `game:BindToClose` (save all in parallel, wait up to 25 s).
- Mark profiles dirty and skip saving unchanged ones. Respect the request budget (`DataStoreService:GetRequestBudgetForRequestType`) before non-critical saves.
- Wrap every call in `Util.Retry(fn, attempts = 5, baseDelay = 1)`.
- **Never enable Studio API access against the live place's data.** Use a separate test place, or key prefix `"test_"` when `RunService:IsStudio()`.

**Schema (version 1):**

```lua
export type Profile = {
  schemaVersion: number,           -- 1
  coins: number,
  createdAt: number,
  lastLoginDay: number,            -- UTC day index for daily gift / XP cap
  dailyXp: number,
  pet: {
    name: string?,                 -- filtered
    lastRenameAt: number,
    species: "SproutPup",
    bondLevel: number,
    bondXp: number,
    needs: { hunger: number, fun: number, energy: number },
    lastNeedsTick: number,         -- for capped offline decay
    tricks: { [string]: { learned: boolean, practiceCount: number } },
    equipped: { head: string?, neck: string?, body: string?, back: string?, trail: string? },
  },
  inventory: { items: { [string]: number } },   -- itemId -> count (cosmetics = 1)
  outfits: { { name: string, equipped: { [string]: string } } }, -- max = 3 or 10
  house: {
    theme: string,                 -- "Default" or bedroom pack id
    privacy: "Open" | "Friends" | "Closed",
    furniture: { { uid: string, itemId: string, cf: { number } } }, -- relative to plot origin, 12 numbers
  },
  passes: { [string]: boolean },   -- cache only
  purchaseLog: { string },         -- last 100 PurchaseIds
  settings: { music: boolean, sfx: boolean },
}
```

- Write a `migrate(profile)` function that upgrades `schemaVersion` step by step and fills in missing fields from a `DEFAULT_PROFILE` deep copy.

### 3.4 Pet follow and animation

- The pet model has a `PrimaryPart` (`Root`, massless and non-collidable with players via a `CollisionGroup` "Pets"), an `Animator` inside an `AnimationController`, and attachments for each cosmetic slot (`HeadAttachment`, `NeckAttachment`, `BodyAttachment`, `BackAttachment`).
- Follow: an `AlignPosition` (`Mode = OneAttachment`, `MaxForce` tuned, `Responsiveness` 25) plus an `AlignOrientation`. Target the owner's `HumanoidRootPart` with an offset of 3 studs behind and 2 to the right. Raycast down to keep the pet on the ground.
- Set network ownership of the pet root to the owner (`BasePart:SetNetworkOwner(player)`) for smooth movement. The server still owns all gameplay state.
- Teleport-snap if the pet is more than 60 studs away. Despawn on character removal and respawn on `CharacterAdded`.
- Animation set (IDs in Config): Idle, Walk, Run, Eat, Fetch, Sit, Spin, Wave, RollOver, Dance, Happy, Sad, Greet, Sleep. Use `Animator:LoadAnimation` once per track and cache the tracks.
- Equip cosmetics by cloning the model from `Assets.Cosmetics` and welding it to the matching attachment with a `WeldConstraint` or `RigidConstraint`.

---

## 4. UI layout (match the concept art)

Use scale-based sizing everywhere, with `UIAspectRatioConstraint` on square elements, `UICorner` (pill radius 0.5), `UIStroke` (white, 3 px, `ApplyStrokeMode = Border`), `UIPadding`, and `UIListLayout` with flex (`UIFlexItem`) for rows. Set `ScreenGui.ScreenInsets = Enum.ScreenInsets.CoreUISafeInsets`. Use `TextScaled` plus `UITextSizeConstraint` (min 14, max 36) and the font **Fredoka One** (`Font.fromName("FredokaOne")`) or the closest available rounded font.

**HUD (always visible):**
- **Top-left:** a game logo badge "POCKET PAL", then below it a **Bond pill**: heart icon, "Bond level N", and a segmented progress bar (coral fill, grey track).
- **Top-right:** a Coins counter (paw coin icon) and buttons for Shop 🛍, Wardrobe 👕, House 🏠 and Settings ⚙ (44×44 minimum touch targets).
- **Bottom-centre:** three large pill buttons, as in the concept art:
  - **Feed**: coral `#F2647A`, apple icon.
  - **Play**: butter `#F7C948`, ball icon.
  - **Walk**: teal `#3BB6A1`, paw icon.
  - Each has a white outline and a **radial cooldown overlay** with seconds remaining. Disable it while on cooldown.
- **Above the action buttons:** three thin need bars (Hunger, Fun, Energy) with icons.
- **Right edge, mid-height:** the Tricks button (star icon), which opens a radial wheel of unlocked tricks with locked ones greyed out and showing "Lv X".
- **Toasts:** top-centre, auto-dismiss after 3 s, queued.

**Screens (modal, with blurred background via `BlurEffect` ≤ 12, and the `MenuContext` input enabled):**
- **Adoption:** pod-hatch cutscene → name entry → "Meet <name>!"
- **Wardrobe:** a pet preview `ViewportFrame` on the left; slot tabs (Head, Neck, Body, Back, Trail); an item grid; and saved outfit slots along the bottom (3 free, 10 with the pass; locked slots show a padlock and "Unlock more slots").
- **Shop:** see 2.4.
- **Decorate:** a bottom tray of owned furniture, ghost-preview placement, and rotate / move / store / confirm buttons. Exit with a big ✓.
- **Settings:** music and SFX toggles, house privacy selector, and a rename button.
- **Player card** (tap another player's pet in the park): pet name, Bond level, owner display name, a **Visit** button and a **Wave** emote button.

**Feedback juice:** `TweenService` press-scale (0.92) on every button, heart particles on care actions, a soft "pop" SFX, and a level-up banner with confetti. Respect a "Reduce motion" setting.

**Input (Input Action System):** set `Workspace.PlayerScriptsUseInputActionSystem` according to the current docs. `PlayContext` actions: `Feed` (1 / ButtonX / HUD button), `Play` (2 / ButtonY), `Walk` (3 / ButtonB), `Throw` (MouseButton1 or tap / ButtonR2), `OpenTricks` (T / DPadUp). Every action has a keyboard/mouse, gamepad **and** touch binding. Swap to `MenuContext` when a modal opens.

---

## 5. Safety, compliance and fairness

- Filter **all** user text shown to others (pet names, outfit names) server-side, as in 1.6. Never display unfiltered text.
- Complete the **Experience Guidelines / maturity questionnaire** in the Creator Dashboard. The target is all ages: no violence, no romance, and no external links.
- No chat features beyond standard `TextChatService`.
- Visits respect privacy settings. Visitors can't move, delete or pick up furniture.
- Anti-exploit: server-side rate limits, distance checks on checkpoints and throws, cooldowns stored per player server-side, and clamped numeric inputs. Log suspicious requests (rate-limited logging) and silently reject them.
- Accessibility: colour is never the only signal (icons plus text); text has at least a 4.5:1 contrast ratio against its background.

---

## 6. Performance budgets

- Server needs tick: one loop for all pets every 5 s (not one loop per pet).
- Client: no `RenderStepped` work except camera and preview; use `PreSimulation` / `Heartbeat` sparingly.
- Pet model at most 1,500 triangles and at most 8 parts. Cosmetics at most 500 triangles each. Use `StreamingEnabled = true` with plot models as `Atomic`.
- Shop `ViewportFrame`s render only while visible.
- Add `--!native` to hot-path modules only if profiling shows a benefit.

---

## 7. Analytics

Use `AnalyticsService`:
- **Funnel** "Onboarding": Joined → Adopted → Named → First Feed → First Play → First Walk → Bond Lv2.
- **Economy events**: Coins sources and sinks.
- **Custom events**: trick performed, house visited, shop opened, item previewed, purchase prompt shown.
- Log `CurrencySpent` from receipts for revenue per SKU.

---

## 8. Build phases (do one at a time, stop after each)

### Phase 1 — Foundations
Config, Types, ItemCatalog (with placeholder items), Net, RateLimiter, Retry utility, DataService (session lock, migrate, autosave, BindToClose), both bootstraps, and a debug command `/resetdata` (Studio only).
**Accept when:** I can join → a profile is created → rejoin → the data persists; two Studio servers don't corrupt data; no warnings in Script Analysis with `--!strict`.

### Phase 2 — Pet, adoption and naming
Sprout Pup placeholder model (built from Parts in the art style if no mesh exists yet), spawn, follow, animations hookup, adoption flow, name filtering, and the name `BillboardGui`.
**Accept when:** the pet follows smoothly on mobile emulation, survives a character respawn, the name is filtered, and a fully hashed name is rejected.

### Phase 3 — Needs and the three activities
Needs tick and offline cap, Feed / Play / Walk with server validation, cooldowns, Coins rewards, and the HUD action buttons with cooldown overlays.
**Accept when:** each activity works by touch, keyboard and gamepad; spamming a remote is rejected; a teleport exploit can't complete checkpoints.

### Phase 4 — Bond, unlocks and tricks
XP curve, daily soft cap, level-up banner, unlock table, Tricks wheel, and learning bonus.
**Accept when:** levels 1→5 are reachable in about 15 minutes of normal play; tricks replicate to other clients.

### Phase 5 — Wardrobe and inventory
Equip and unequip, free cosmetics, saved outfits (3 slots), and Coins shop for food and free-tier items.
**Accept when:** the equipped look persists across rejoin, other players see it, and the outfit slot limit is enforced on the server.

### Phase 6 — House and decorating
Plot assignment, house template, decorate mode with grid placement and validation, privacy, and visiting.
**Accept when:** furniture persists; placing outside bounds or over the item limit is rejected; Closed blocks visitors.

### Phase 7 — Park and social
Spotlight stage, pet greetings, player cards, and visit teleport.
**Accept when:** cards show correct filtered names; greetings trigger only for pets above level 10.

### Phase 8 — Monetisation
`MonetizationService`, idempotent `ProcessReceipt`, passes (Outfit Slots, 3 bedroom packs), costume and bed dev products, live prices via `GetProductInfoAsync`, Shop UI with try-on, owned-state blocking, and the `PolicyService` check.
**Accept when:** a test purchase in Studio grants once; a forced duplicate receipt (same `PurchaseId`) doesn't double-grant; leaving mid-purchase grants on the next join; owned items can't be re-prompted; prices aren't hard-coded.
**Manual steps for me:** list exactly which passes and products to create in Creator Dashboard → Monetization, with suggested names, descriptions and 512×512 icon briefs, and where to paste each ID in `Config`.

### Phase 9 — Polish and launch checklist
SFX and music, particles, reduce-motion setting, analytics events, performance pass, maturity questionnaire reminder, and thumbnail/icon briefs based on the concept art.
**Accept when:** the Microprofiler shows no server spikes over 5 ms from game code in a 24-player test; mobile frame rate is stable.

---

## 9. Output format for every phase

1. A list of every instance to create, with its full Explorer path and type.
2. The complete code for each script, with no "…rest unchanged" placeholders.
3. Any manual setup steps (Studio settings, Dashboard, asset uploads).
4. The phase acceptance-test checklist, written as steps I can follow in Play / Start Server (2 players) / Device Emulator.
5. Known limitations and what the next phase will touch.

If any API in this spec has been renamed or deprecated since November 2026, use the current replacement from the Creator Hub API reference and tell me what you changed.
