# Auric Arcanum - Original Behavior Analysis
**NeoForge 1.21.1 → Source: OutrightWings/ModJamSynergyMod**

## Overview
Auric Arcanum is a synergistic elemental magic mod where players select and combine up to 6 elements to cast spells with dynamic effects based on element combinations.

---

## 1. ELEMENTS SYSTEM

### 1.1 ElementType Enum (6 elements + NONE)
```
NONE (id=0) - no element
FIRE (id=1)
ICE (id=2)
ROCK (id=3)
LIFE (id=4)
DEATH (id=5)
WATER (id=6)
```

### 1.2 Element Selection
- **Max selected**: 6 elements
- **Storage**: int[] array in SelectedElementsComponent
- **Indexing**: 0 = NONE, 1 = FIRE, 2 = ICE, etc.
- **Persistence**: Stored on wand item via ItemComponent
- **Synchronization**: Bidirectional packet sync (client ↔ server)

### 1.3 Element Counting
- countElements() returns int[6] with counts for each element (excluding NONE)
- index 0 = FIRE count, index 1 = ICE count, etc.

---

## 2. CASTING FORMS

### 2.1 CastingForms Enum
```
PROJECTILE - launches projectile(s) that hit on impact
SELF - applies effects to caster immediately
```

### 2.2 Attack Forms (Determined by 2 most common elements)
```
NONE - no attack
SPRAY - multiple projectiles with spread
PROJECTILE - single/multiple fast projectiles
BEAM - single rapid projectile
```

### 2.3 Attack Form Mapping
Attack form selected by the 2 most abundant elements in selection:
- Fire+Fire → SPRAY
- Fire+Ice → SPRAY
- Fire+Rock → PROJECTILE
- Fire+Life → SPRAY
- Fire+Death → PROJECTILE
- Fire+Water → SPRAY
- Ice+Ice → SPRAY
- Ice+Rock → PROJECTILE
- Ice+Life → BEAM
- Ice+Death → BEAM
- Ice+Water → PROJECTILE
- Rock+Rock → PROJECTILE
- Rock+Life → PROJECTILE
- Rock+Death → PROJECTILE
- Rock+Water → PROJECTILE
- Life+Life → BEAM
- Life+Death → BEAM
- Life+Water → SPRAY
- Death+Death → BEAM
- Death+Water → SPRAY
- Water+Water → SPRAY

---

## 3. PARTICLE EFFECTS

Particles selected by same 2-element logic:
- Fire+Fire → FLAME
- Fire+Ice → CLOUD
- Fire+Rock → FALLING_LAVA
- Fire+Life → LAVA
- Fire+Death → ELECTRIC_SPARK
- Fire+Water → CLOUD
- Ice+Ice → SNOWFLAKE
- Ice+Rock → SNOWFLAKE
- Ice+Life → SNOWFLAKE
- Ice+Death → SNOWFLAKE
- Ice+Water → SNOWFLAKE
- Rock+Rock → WHITE_ASH
- Rock+Life → WHITE_ASH
- Rock+Death → DUST_PLUME
- Rock+Water → WHITE_ASH
- Life+Life → COMPOSTER
- Life+Death → ASH
- Life+Water → BUBBLE
- Death+Death → CRIT
- Death+Water → BUBBLE
- Water+Water → BUBBLE

---

## 4. PROPERTIES CALCULATION

### 4.1 Damage & Knockback
- **Base damage**: 0
- **Fire adds**: 1 damage per Fire element
- **Rock adds**: 1 damage per Rock + 0.2f knockback per Rock
- **Death adds**: 1 damage per Death

### 4.2 Knockback
- **Rock**: +0.2f per Rock
- **Water**: +0.5f per Water
- **Gravity**: Disabled if primary element is Fire or Fire+Ice

### 4.3 Potion Effects (60 ticks = 1 effect unit)
**Duration per element**: 60 ticks
**Stack rules**: Some effects consume Life element for amplification

#### Fire
- If paired with Life: FIRE_RESISTANCE (60*count)
- Else: Sets fireTicks = 20*count for entity burning

#### Ice
- Sets isCold=true, clears fireTicks
- If NOT paired with Fire:
  - MOVEMENT_SLOWDOWN (60*count)
  - DIG_SLOWDOWN (60*count)
  - If count > 1: freezeTicks = 20*(count-1)
- If paired with Fire:
  - MOVEMENT_SPEED (60*count)

#### Rock
- If paired with Life: DAMAGE_RESISTANCE (60*count)
- Else: Sets isRock=true, adds damage and knockback

#### Water
- If paired with Life: WATER_BREATHING (60*count)
- Else: Sets isWet=true, clears fireTicks, adds knockback
  - If cold: projectile = magic_ice
  - Else if rock: projectile = magic_clay

#### Death
- If paired with Water: POISON (60*water_count)
- If paired with Life: WITHER (60*death_count)
- Else: Adds damage, sets isDeath=true
  - If rock: projectile = AIR (empty)

#### Life
- REGENERATION (60*count)
- Sets isLife=true
- Consumes from Death effects

### 4.4 Projectile Type
- Fire+Rock → MAGMA_BLOCK
- Ice+Rock → ICE
- Rock alone → STONE
- Water+Ice → FROSTED_ICE (via special logic)
- Water alone → ICE
- Death alone → AIR (empty/no projectile)
- Default → AIR

### 4.5 Special Gravity Rules
- Fire (single or with Ice) → no gravity
- Default → gravity enabled

---

## 5. WAND MECHANICS

### 5.1 Wand Item
- 6 variants: BLUE_WAND, RED_WAND, GREEN_WAND, PURPLE_WAND, WHITE_WAND, LIGHT_BLUE_WAND
- Stack size: 1
- Attachment: SelectedElementsComponent + SelectedFormComponent

### 5.2 Wand Interactions
- **Left-click (server-side)**: Cast spell if elements selected
  - Calls `ILeftClickReact.onLeftClick()` via Moonlight interface
  - Creates MagicProps, calls cast()
  - Awards ITEM_USED stat

- **Right-click (server-side)**: 
  - No Shift: Opens ElementMenu GUI
  - With Shift: Clears selected elements

- **Tooltip**: Shows current form and selected elements with counts

---

## 6. CASTING MECHANICS

### 6.1 Cast Types by AttackForm

#### PROJECTILE
- lifetime = 1000 ticks
- velocity = 1.0
- inaccuracy = 0.5
- Single MagicBall spawned

#### BEAM
- lifetime = 200 ticks
- velocity = 1.0
- inaccuracy = 0.0
- No gravity
- Single MagicBall spawned

#### SPRAY
- lifetime = 20 ticks
- velocity = 0.5
- inaccuracy = 2.0
- Damage ÷ 3 (min 1)
- Multiple MagicBalls spawned:
  - Count = elements_selected * 2
  - Each with random pitch/yaw spread of ±22.5°

#### SELF
- No projectile
- Applies effects directly to caster
- Uses potion effects from calculation

---

## 7. MAGICBALL ENTITY

### 7.1 Registration
- Entity ID: "magic_ball"
- Category: MISC
- Size: 0.15F × 0.15F
- Update interval: 10 ticks
- Tracking range: 4 blocks

### 7.2 Properties
- Inherits from ImprovedProjectileEntity (Moonlight)
- Default item: AIR
- Contains all MagicProps state
- Can attach invisible potion entity

### 7.3 Particle Trail
- Spawns particle at center of projectile
- Offset by velocity × 0.25
- Uses MagicProps particle type

### 7.4 Behavior on Block Hit
- **Death + Killable block** (ModTags.DEATH_KILLABLE):
  - Sets block to AIR
  - Stops projectile

- **Death + Dirtable block** (ModTags.DEATH_DIRTABLE):
  - Sets block to DIRT

- **Wet + (not cold OR fire) + mud-convertible block**:
  - Sets block to magic_mud (ages away)

- **Wet + Death + Dirtable**:
  - Sets block to magic_mud

- **Fire + not wet + not cold + empty block**:
  - Sets adjacent block to fire

- **Fire + not wet + not cold + ice block**:
  - Sets block to water

- **Wet + Cold + Fire + magic_ice projectile**:
  - Sets adjacent block to water

- **BlockItem projectile**:
  - Places block at hit location

- **Life element**:
  - Applies bone meal at hit location and adjacent block

### 7.5 Behavior on Water During Flight
- **Cold + not Fire**:
  - Sets block to frosted_ice
  - Stops projectile

- **Fire + not wet**:
  - Sets block to sponge then air
  - Stops projectile

### 7.6 Behavior on Entity Hit
- Applies all MagicProps effects
- Calculates knockback direction from impact
- Removes projectile item (for block items)

### 7.7 Death Effects
- **Death + Rock**:
  - Explosion at hit location
  - Radius: 0.5f
  - No fire
  - BLOCK interaction type

- **Fire + Death**:
  - Spawns lightning bolt at hit location

---

## 8. INVISIBLEPOTION ENTITY

### 8.1 Registration
- Entity ID: "invisible_potion"
- Category: MISC
- Size: 0.1F × 0.1F
- Update interval: 10 ticks
- Tracking range: 4 blocks

### 8.2 Mechanics
- Attached to MagicBall as passenger
- Invisible to player
- Carries splash potion with effects
- Triggers on block/entity hit via onHit() callback

---

## 9. GUI/SCREEN

### 9.1 ElementScreen
- Texture: "textures/gui/container/element_gui.png" (256×256)
- Window size: appears to be standard (likely 176×166)

### 9.2 Element Selection UI
- 6 element buttons arranged in hexagon:
  - Fire (50, 17)
  - Ice (98, 17)
  - Rock (130, 65)
  - Life (98, 113)
  - Death (50, 113)
  - Water (18, 65)
- Button size: 29×29 pixels
- Highlight texture at (227, 70) on texture atlas

### 9.3 Selected Elements Display
- Bar at (69, 67) showing selected elements
- Small icons: 11×11 pixels
- Gap between icons: 3 pixels
- Grid: 3 columns, 2 rows
- Highlight at (245, 239) on texture atlas

### 9.4 Interaction
- **Left-click** on element: Add to selection (first empty slot)
- **Right-click** anywhere: Remove last selected element
- **Middle-click** anywhere: Clear all selections
- **Hover**: Shows element tooltip

### 9.5 Sync to Server
- Uses PacketDistributor.sendToServer()
- Sends SelectedElementsComponent packet

---

## 10. COMPONENTS (NeoForge DataComponents)

### 10.1 SelectedElementsComponent
- Type: Record with int[] array
- Codec: Codec.INT.listOf()
- StreamCodec: Custom int array serialization
- Persistent: Yes (saved to NBT)
- Synchronized: Yes (sent to client on join/update)
- Handler: Updates wand NBT on packet receive

### 10.2 SelectedFormComponent
- Type: Record with int id
- Codec: Codec.INT.fieldOf("id")
- StreamCodec: ByteBufCodecs.INT
- Persistent: Yes
- Synchronized: Yes
- Handler: Updates wand NBT, shows action bar message

---

## 11. NETWORKING

### 11.1 Packets
1. **SelectedElementsComponent**
   - Direction: Bidirectional (PLAY)
   - Payload: int[] of selected element IDs
   - Handler: Updates main hand wand component

2. **SelectedFormComponent**
   - Direction: Bidirectional (PLAY)
   - Payload: int id of selected casting form
   - Handler: Updates main hand wand component, shows message

### 11.2 Registration
- Registrar version: "1"
- Uses DirectionalPayloadHandler for bi-directional handling
- Both packets use playBidirectional registration

---

## 12. BLOCKS

### 12.1 Temporary Magic Blocks
- **TempMagicBlocks** (abstract):
  - Extends Block
  - Has AGE_3 state property (0-3)
  - Ticks every 20 ticks when placed
  - Auto-removes at max age
  - Extends class can override placeAfter()

### 12.2 Concrete Blocks
1. **magic_stone**: TempMagicBlocks → COBBLESTONE base
2. **magic_mud**: TempMud → converts to DIRT
3. **magic_clay**: TempMagicBlocks → CLAY base
4. **magic_ice**: TempMagicBlocks → ICE base
5. **magic_magma**: TempMagicBlocks → MAGMA_BLOCK base

### 12.3 Block Lifespan
- Placed
- Waits 60 ticks
- Every 20 ticks increments age
- At age 3: Removed (or special handling in subclass)

---

## 13. TAGS

### 13.1 ModTags
- **DEATH_KILLABLE**: Blocks that Death can destroy
  - Default: Tall plants, flowers, crops
- **DEATH_DIRTABLE**: Blocks that Death converts to dirt
  - Default: Grass, etc.

---

## 14. KEYBINDS & INPUT

### 14.1 Scroll Listener (Client)
- Scroll wheel adjusts casting form (PROJECTILE ↔ SELF)
- Shift + scroll changes form
- Uses MouseScrollingEvent
- Event can be cancelled

---

## 15. RESOURCES

### 15.1 Translations
- Format: Translation key = "element.auric_arcanum.ELEMENT_NAME"
- Form keys: "form.auric_arcanum.PROJECTILE", "form.auric_arcanum.SELF"

### 15.2 Textures
- GUI: assets/auric_arcanum/textures/gui/container/element_gui.png

### 15.3 Entity Rendering
- MagicBall: ThrownItemRenderer (renders held item)
- InvisiblePotion: InvisiblePotionRenderer (custom, likely renders nothing)

---

## 16. DEPENDENCIES

### 16.1 Required
- **NeoForge**: [21.0.0-beta, ∞)
- **Minecraft**: [1.21, 1.21.1)

### 16.2 Optional
- **Moonlight Lib** (curse.maven:selene-499980:6254821)
  - Used for: ImprovedProjectileEntity, ILeftClickReact

---

## 17. KEY BEHAVIORS SUMMARY

1. **Element Selection**: Up to 6 elements, persisted on wand
2. **Dynamic Casting**: Properties calculated from element combination
3. **Attack Forms**: Determined by 2 most common elements
4. **Projectiles**: MagicBall entities with complex hit behaviors
5. **Block Interactions**: Create/destroy/transform blocks based on elements
6. **Potion Effects**: Applied to entities via splash potions or direct application
7. **Visual Feedback**: Particles, projectile rendering, element selection UI
8. **Sync**: Bidirectional component synchronization ensures server/client consistency
9. **Persistence**: All state saved to item NBT and synced on world load/join

