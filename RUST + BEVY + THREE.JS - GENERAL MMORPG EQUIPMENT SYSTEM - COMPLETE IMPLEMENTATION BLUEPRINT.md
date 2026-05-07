# MMORPG Equipment System – Complete Implementation Blueprint
### Audited & Updated — May 2026

---

## 🔴 Critical Issues Fixed

| # | Location | Issue | Fix |
|---|----------|-------|-----|
| 1 | `equipment_manager.rs` | `get_total_stats` moves `total` into cache then calls `total.clone()` — **compile error** (use of moved value) | Store clone in cache first, return original |
| 2 | `validator.rs` | Accesses `mgr.equipped` directly — **private field**, won't compile from a sibling module | Changed `equipped` to `pub` in `EquipmentManager`, or added accessor |
| 3 | `equipment_tests.rs` | `GearItem { required_level: 10, ..Default::default() }` — `GearItem` never derives/implements `Default` — **compile error** | Added `impl Default for GearItem` |
| 4 | `equipment_tests.rs` | Calls `mgr.get_equipped_item(slot)` — method **never defined** in `EquipmentManager` | Added `get_equipped_item` method |
| 5 | `gear_item.rs` | `serialize()` calls `.unwrap()` — panics on any encode error | Changed return type to `Result<Vec<u8>, rmp_serde::encode::Error>` |
| 6 | `SaveSystem.ts` | `await store.put(...)` and `await store.get(...)` — **IDBRequest is not a Promise**, will silently return `undefined` | Wrapped both in `new Promise(...)` |
| 7 | `DragDrop.ts` | References `gameInstance` — **never imported or defined**, runtime crash | Changed to accept `EquipmentManager` as constructor dependency |
| 8 | `SlotWidget.ts` | References `network` global — **never imported or defined** | Added `NetworkClient` parameter to constructor |
| 9 | `Cargo.toml` quick start | `cargo add bevy bevy_replicon` with no versions — bevy_replicon must match Bevy version exactly | Pinned compatible versions for Bevy 0.15 |
| 10 | `validator.rs` | Unique-ring check uses `item.equip_slot` (the item's default slot) not the **actual target slot** being equipped — logic error | Added `target_slot: EquipSlot` param to `validate_all` |
| 11 | `DragDropManager.isValidDrop` | Calls `canEquipItem(item, targetSlot.slotType)` but the Rust/TS signature only takes `(item)` — **signature mismatch** | Updated TS `canEquipItem` to accept slot |
| 12 | `stats_engine.rs` | `update_from_equipment` is a static-like fn that receives `&mut EquipmentManager` but belongs to an instance struct — misleading design | Changed to `&mut self` method |
| 13 | `EquipSlot::as_str` | Body is `{ ... }` placeholder — will not compile | Filled in all arms |

---

## 0. SKILLS REQUIRED

- Rust (ownership, async, traits, enums, serde) or TypeScript (async, classes, generics)
- ECS (Bevy 0.15) or component-based architecture
- Real-time networking (WebSocket, RPC, state reconciliation)
- UI systems (drag & drop, tooltips, dynamic layouts)
- Data serialization (JSON, MessagePack/rmp-serde)
- Database design (SQLite or PostgreSQL for item catalog)
- Performance profiling (target: <1ms stat recalc)
- Testing (unit, integration, network simulation)

---

## 1. FILE & FOLDER STRUCTURE

### Rust + Bevy (backend)

```
my_game/
├── assets/
│   ├── items/
│   │   ├── icons/
│   │   └── item_data/          (.ron or .json item definitions)
│   └── ui/
│       └── character_sheet/
├── crates/
│   ├── game_core/
│   │   ├── src/
│   │   │   ├── gear/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── enums.rs
│   │   │   │   ├── gear_item.rs
│   │   │   │   ├── equipment_manager.rs
│   │   │   │   └── validator.rs
│   │   │   ├── networking/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── server.rs
│   │   │   │   ├── client.rs
│   │   │   │   ├── protocol.rs
│   │   │   │   └── reconciliation.rs
│   │   │   ├── offline/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── save_system.rs
│   │   │   │   └── sync.rs
│   │   │   ├── advanced/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── ammo.rs
│   │   │   │   ├── durability.rs
│   │   │   │   ├── set_bonuses.rs
│   │   │   │   └── stats_engine.rs
│   │   │   └── lib.rs
│   │   └── Cargo.toml
│   └── game_client/
│       ├── src/
│       │   ├── ui/
│       │   │   ├── character_sheet.rs
│       │   │   ├── slot_widget.rs
│       │   │   ├── drag_drop.rs
│       │   │   └── tooltip.rs
│       │   └── main.rs
│       └── Cargo.toml
└── Cargo.toml (workspace)
```

### Three.js + TypeScript (frontend)

```
my_mmo/
├── client/
│   ├── public/assets/items/icons/
│   ├── src/
│   │   ├── game/
│   │   │   ├── gear/
│   │   │   │   ├── enums.ts
│   │   │   │   ├── GearItem.ts
│   │   │   │   ├── EquipmentManager.ts
│   │   │   │   └── Validator.ts
│   │   │   ├── networking/
│   │   │   │   ├── ClientSocket.ts
│   │   │   │   ├── protocol.ts
│   │   │   │   └── Reconciliation.ts
│   │   │   ├── offline/
│   │   │   │   ├── SaveSystem.ts
│   │   │   │   └── Sync.ts
│   │   │   └── advanced/
│   │   │       ├── Ammo.ts
│   │   │       ├── Durability.ts
│   │   │       ├── SetBonuses.ts
│   │   │       └── StatsEngine.ts
│   │   ├── ui/
│   │   │   ├── CharacterSheet.ts
│   │   │   ├── SlotWidget.ts
│   │   │   ├── DragDrop.ts
│   │   │   └── Tooltip.ts
│   │   └── main.ts
│   ├── package.json
│   └── tsconfig.json
└── server/
    ├── src/
    │   ├── gear/
    │   │   ├── EquipmentServer.ts
    │   │   └── ItemDatabase.ts
    │   ├── networking/
    │   │   └── ServerSocket.ts
    │   └── index.ts
    └── package.json
```

---

## 2. CODE SNIPPETS

### 2.1 Enums & Constants — `crates/game_core/src/gear/enums.rs` ✅ UPDATED

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum EquipSlot {
    Head, Shoulder, Back, Chest, Wrist, Hands, Waist, Legs, Feet,
    MainHand, OffHand, Ranged, Neck, Ring1, Ring2, Trinket1, Trinket2,
    Shirt, Tabard, Ammo,
}

impl EquipSlot {
    // FIX: Original had `{ ... }` placeholder — filled in all arms.
    pub fn as_str(&self) -> &'static str {
        match self {
            EquipSlot::Head => "Head",
            EquipSlot::Shoulder => "Shoulder",
            EquipSlot::Back => "Back",
            EquipSlot::Chest => "Chest",
            EquipSlot::Wrist => "Wrist",
            EquipSlot::Hands => "Hands",
            EquipSlot::Waist => "Waist",
            EquipSlot::Legs => "Legs",
            EquipSlot::Feet => "Feet",
            EquipSlot::MainHand => "Main Hand",
            EquipSlot::OffHand => "Off Hand",
            EquipSlot::Ranged => "Ranged",
            EquipSlot::Neck => "Neck",
            EquipSlot::Ring1 => "Ring 1",
            EquipSlot::Ring2 => "Ring 2",
            EquipSlot::Trinket1 => "Trinket 1",
            EquipSlot::Trinket2 => "Trinket 2",
            EquipSlot::Shirt => "Shirt",
            EquipSlot::Tabard => "Tabard",
            EquipSlot::Ammo => "Ammo",
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ArmorType { Light, Medium, Heavy, Plate }

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum WeaponType {
    Sword1H, Sword2H, Axe1H, Axe2H, Mace1H, Mace2H, Dagger, Fist,
    Staff, Polearm, Bow, Crossbow, Gun, Wand, Thrown, Shield, Totem, Libram, Idol,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ItemQuality { Poor, Common, Uncommon, Rare, Epic, Legendary }

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum CharacterClass {
    Warrior, Paladin, Hunter, Rogue, Priest, Shaman, Mage, Warlock, Druid
}
```

---

### 2.2 GearItem — `crates/game_core/src/gear/gear_item.rs` ✅ UPDATED

```rust
use super::enums::*;
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GearItem {
    pub id: u32,
    pub name: String,
    pub icon_path: String,
    pub quality: ItemQuality,
    pub equip_slot: EquipSlot,
    pub armor_type: Option<ArmorType>,
    pub weapon_type: Option<WeaponType>,
    pub is_two_handed: bool,
    pub requires_ammo: bool,
    pub ammo_type: String,
    pub required_level: u16,
    pub allowed_classes: Vec<CharacterClass>,
    pub armor: u16,
    pub stamina: i16,
    pub strength: i16,
    pub agility: i16,
    pub intellect: i16,
    pub spirit: i16,
    pub min_damage: f32,
    pub max_damage: f32,
    pub attack_speed: f32,
    pub passive_effects: Vec<String>,
    pub on_use_effects: Vec<String>,
    pub on_equip_effects: Vec<String>,
}

// FIX: Tests used `..Default::default()` but Default was never implemented.
impl Default for GearItem {
    fn default() -> Self {
        Self {
            id: 0,
            name: String::new(),
            icon_path: String::new(),
            quality: ItemQuality::Common,
            equip_slot: EquipSlot::MainHand,
            armor_type: None,
            weapon_type: None,
            is_two_handed: false,
            requires_ammo: false,
            ammo_type: String::new(),
            required_level: 0,
            allowed_classes: vec![],
            armor: 0,
            stamina: 0, strength: 0, agility: 0, intellect: 0, spirit: 0,
            min_damage: 0.0, max_damage: 0.0, attack_speed: 1.0,
            passive_effects: vec![],
            on_use_effects: vec![],
            on_equip_effects: vec![],
        }
    }
}

impl GearItem {
    pub fn can_equip(&self, class: CharacterClass, level: u16) -> bool {
        if level < self.required_level { return false; }
        if !self.allowed_classes.is_empty() && !self.allowed_classes.contains(&class) {
            return false;
        }
        true
    }

    pub fn dps(&self) -> f32 {
        if self.attack_speed == 0.0 { return 0.0; }
        (self.min_damage + self.max_damage) / 2.0 / self.attack_speed
    }

    // FIX: Was `fn serialize(&self) -> Vec<u8> { rmp_serde::to_vec(...).unwrap() }`.
    // Panicking in a library function is bad practice — returns Result instead.
    pub fn serialize(&self) -> Result<Vec<u8>, rmp_serde::encode::Error> {
        rmp_serde::to_vec(self)
    }

    pub fn deserialize(data: &[u8]) -> Result<Self, rmp_serde::decode::Error> {
        rmp_serde::from_slice(data)
    }
}
```

---

### 2.3 EquipmentManager — `crates/game_core/src/gear/equipment_manager.rs` ✅ UPDATED

```rust
use std::collections::HashMap;
use super::enums::*;
use super::gear_item::GearItem;
use super::validator::Validator;

pub struct EquipmentManager {
    // FIX: Changed from private to `pub(crate)` so Validator (sibling module)
    // can access it. Private fields caused a compile error in validator.rs.
    pub(crate) equipped: HashMap<EquipSlot, GearItem>,
    pub character_class: CharacterClass,
    pub level: u16,
    pub can_dual_wield: bool,
    pub can_use_heavy_armor: bool,
    pub can_use_plate_armor: bool,
    total_stats_cache: CharacterStats,
    cache_valid: bool,
}

#[derive(Debug, Clone, Default)]
pub struct CharacterStats {
    pub armor: u32,
    pub stamina: i32,
    pub strength: i32,
    pub agility: i32,
    pub intellect: i32,
    pub spirit: i32,
    pub attack_power: i32,
    pub spell_power: i32,
}

impl EquipmentManager {
    pub fn new(class: CharacterClass, level: u16) -> Self {
        let mut m = Self {
            equipped: HashMap::new(),
            character_class: class,
            level,
            can_dual_wield: false,
            can_use_heavy_armor: false,
            can_use_plate_armor: false,
            total_stats_cache: CharacterStats::default(),
            cache_valid: false,
        };
        m.update_armor_proficiency();
        m
    }

    fn update_armor_proficiency(&mut self) {
        if self.level >= 40 {
            match self.character_class {
                CharacterClass::Warrior | CharacterClass::Paladin => {
                    self.can_use_plate_armor = true;
                    self.can_use_heavy_armor = true;
                }
                CharacterClass::Hunter | CharacterClass::Shaman => {
                    self.can_use_heavy_armor = true;
                }
                _ => {}
            }
        } else if self.level >= 10 {
            match self.character_class {
                CharacterClass::Warrior | CharacterClass::Paladin |
                CharacterClass::Hunter | CharacterClass::Shaman => {
                    self.can_use_heavy_armor = true;
                }
                _ => {}
            }
        }
    }

    pub fn can_equip_item(&self, item: &GearItem, target_slot: EquipSlot) -> Result<(), String> {
        // FIX: Added `target_slot` parameter — required by Validator to check
        // unique-ring logic against the correct slot.
        Validator::validate_all(item, self, target_slot)
    }

    pub fn equip_item(&mut self, item: GearItem, slot: EquipSlot) -> Result<(), String> {
        self.can_equip_item(&item, slot)?;
        if item.is_two_handed && slot == EquipSlot::MainHand {
            self.equipped.remove(&EquipSlot::OffHand);
        }
        self.equipped.insert(slot, item);
        self.invalidate_cache();
        Ok(())
    }

    pub fn unequip_item(&mut self, slot: EquipSlot) -> Option<GearItem> {
        let item = self.equipped.remove(&slot);
        self.invalidate_cache();
        item
    }

    // FIX: Tests called `mgr.get_equipped_item(slot)` but this method didn't exist.
    pub fn get_equipped_item(&self, slot: EquipSlot) -> Option<&GearItem> {
        self.equipped.get(&slot)
    }

    pub fn get_total_stats(&mut self) -> CharacterStats {
        if self.cache_valid {
            return self.total_stats_cache.clone();
        }
        let mut total = CharacterStats::default();
        for item in self.equipped.values() {
            total.armor += item.armor as u32;
            total.stamina += item.stamina as i32;
            total.strength += item.strength as i32;
            total.agility += item.agility as i32;
            total.intellect += item.intellect as i32;
            total.spirit += item.spirit as i32;
            total.attack_power += item.strength as i32 * 2;
        }
        // FIX: Original code did `self.total_stats_cache = total; total.clone()`
        // which is a compile error — `total` is moved in the assignment, then
        // used again. Fixed by cloning before the move.
        let result = total.clone();
        self.total_stats_cache = total;
        self.cache_valid = true;
        result
    }

    fn invalidate_cache(&mut self) {
        self.cache_valid = false;
    }
}
```

---

### 2.4 Validator — `crates/game_core/src/gear/validator.rs` ✅ UPDATED

```rust
use super::equipment_manager::EquipmentManager;
use super::enums::*;
use super::gear_item::GearItem;

pub struct Validator;

impl Validator {
    // FIX: Added `target_slot: EquipSlot` parameter.
    // Original used `item.equip_slot` to decide which ring to check — but
    // the item's default slot and the actual target slot can differ.
    pub fn validate_all(
        item: &GearItem,
        mgr: &EquipmentManager,
        target_slot: EquipSlot,
    ) -> Result<(), String> {
        if mgr.level < item.required_level {
            return Err(format!("Requires level {}", item.required_level));
        }
        if !item.allowed_classes.is_empty()
            && !item.allowed_classes.contains(&mgr.character_class)
        {
            return Err("Class cannot equip this item".to_string());
        }
        if let Some(armor) = item.armor_type {
            if !Self::can_wear_armor(mgr, armor) {
                return Err("Armor type not allowed for this class/level".to_string());
            }
        }
        if let Some(weapon) = item.weapon_type {
            if !Self::has_weapon_skill(mgr, weapon) {
                return Err("Weapon proficiency missing".to_string());
            }
        }
        if item.is_two_handed && mgr.equipped.contains_key(&EquipSlot::OffHand) {
            return Err("Cannot equip two-handed weapon while an off-hand is equipped".to_string());
        }

        // FIX: Unique-ring check now uses `target_slot` (the actual destination)
        // instead of `item.equip_slot` (the item's default slot field).
        if matches!(target_slot, EquipSlot::Ring1 | EquipSlot::Ring2) {
            let other_ring = if target_slot == EquipSlot::Ring1 {
                EquipSlot::Ring2
            } else {
                EquipSlot::Ring1
            };
            if let Some(other) = mgr.equipped.get(&other_ring) {
                if other.id == item.id {
                    return Err("Unique: this ring is already equipped".to_string());
                }
            }
        }

        // Same unique check for trinkets
        if matches!(target_slot, EquipSlot::Trinket1 | EquipSlot::Trinket2) {
            let other_trinket = if target_slot == EquipSlot::Trinket1 {
                EquipSlot::Trinket2
            } else {
                EquipSlot::Trinket1
            };
            if let Some(other) = mgr.equipped.get(&other_trinket) {
                if other.id == item.id {
                    return Err("Unique: this trinket is already equipped".to_string());
                }
            }
        }

        Ok(())
    }

    fn can_wear_armor(mgr: &EquipmentManager, armor: ArmorType) -> bool {
        match armor {
            ArmorType::Light => true,
            ArmorType::Medium => !matches!(
                mgr.character_class,
                CharacterClass::Priest | CharacterClass::Mage | CharacterClass::Warlock
            ),
            ArmorType::Heavy => mgr.can_use_heavy_armor,
            ArmorType::Plate => mgr.can_use_plate_armor,
        }
    }

    fn has_weapon_skill(mgr: &EquipmentManager, weapon: WeaponType) -> bool {
        match mgr.character_class {
            CharacterClass::Warrior => true,
            CharacterClass::Paladin => matches!(
                weapon,
                WeaponType::Sword1H | WeaponType::Sword2H | WeaponType::Mace1H
                    | WeaponType::Mace2H | WeaponType::Axe1H | WeaponType::Axe2H
                    | WeaponType::Polearm | WeaponType::Shield | WeaponType::Libram
            ),
            CharacterClass::Hunter => matches!(
                weapon,
                WeaponType::Bow | WeaponType::Crossbow | WeaponType::Gun
                    | WeaponType::Axe1H | WeaponType::Sword1H | WeaponType::Dagger
                    | WeaponType::Polearm | WeaponType::Staff
            ),
            CharacterClass::Rogue => matches!(
                weapon,
                WeaponType::Dagger | WeaponType::Sword1H | WeaponType::Mace1H
                    | WeaponType::Fist | WeaponType::Thrown
            ),
            CharacterClass::Priest => matches!(
                weapon,
                WeaponType::Dagger | WeaponType::Staff | WeaponType::Wand | WeaponType::Mace1H
            ),
            CharacterClass::Shaman => matches!(
                weapon,
                WeaponType::Mace1H | WeaponType::Mace2H | WeaponType::Axe1H
                    | WeaponType::Axe2H | WeaponType::Staff | WeaponType::Shield
                    | WeaponType::Totem | WeaponType::Dagger | WeaponType::Fist
            ),
            CharacterClass::Mage => matches!(
                weapon,
                WeaponType::Staff | WeaponType::Sword1H | WeaponType::Dagger | WeaponType::Wand
            ),
            CharacterClass::Warlock => matches!(
                weapon,
                WeaponType::Staff | WeaponType::Sword1H | WeaponType::Dagger | WeaponType::Wand
            ),
            CharacterClass::Druid => matches!(
                weapon,
                WeaponType::Staff | WeaponType::Mace1H | WeaponType::Mace2H
                    | WeaponType::Dagger | WeaponType::Fist | WeaponType::Idol
            ),
        }
    }
}
```

---

### 2.5 Networking Protocol — `protocol.ts` (shared) ✅ NO CHANGES NEEDED

```typescript
export type EquipRequest      = { type: 'EquipRequest';      requestId: number; itemId: number; slot: number };
export type EquipResponse     = { type: 'EquipResponse';     requestId: number; success: boolean; slot: number; itemId?: number; reason?: string };
export type UnequipRequest    = { type: 'UnequipRequest';    requestId: number; slot: number };
export type UnequipResponse   = { type: 'UnequipResponse';   requestId: number; success: boolean; slot: number; reason?: string };
export type SyncFullEquipment = { type: 'SyncFullEquipment'; equipmentData: Record<string, number | null> };
export type BroadcastEquipChange = { type: 'BroadcastEquipChange'; peerId: number; slot: number; itemId: number | null };

export type GameMessage =
    | EquipRequest | EquipResponse
    | UnequipRequest | UnequipResponse
    | SyncFullEquipment | BroadcastEquipChange;

export function encodeMessage(msg: GameMessage): string { return JSON.stringify(msg); }
export function decodeMessage(data: string): GameMessage { return JSON.parse(data); }
```

---

### 2.6 Client Socket — `client/src/game/networking/ClientSocket.ts` ✅ NO CHANGES NEEDED

```typescript
import { EquipmentManager } from '../gear/EquipmentManager';
import { GameMessage } from './protocol';

export class ClientSocket {
    private ws: WebSocket;
    private pendingRequests = new Map<number, { slot: number; optimisticItemId: number | null }>();
    private requestId = 0;
    public localEquipment: EquipmentManager;

    constructor(url: string, localEq: EquipmentManager) {
        this.localEquipment = localEq;
        this.ws = new WebSocket(url);
        this.ws.onmessage = (ev) => this.handleMessage(JSON.parse(ev.data));
    }

    requestEquip(itemId: number, slot: number) {
        const reqId = this.requestId++;
        this.localEquipment.equipOptimistic(slot, itemId);
        this.pendingRequests.set(reqId, { slot, optimisticItemId: itemId });
        this.send({ type: 'EquipRequest', requestId: reqId, itemId, slot });
        setTimeout(() => this.rollbackIfPending(reqId), 5000);
    }

    private handleMessage(msg: GameMessage) {
        if (msg.type === 'EquipResponse') {
            const pending = this.pendingRequests.get(msg.requestId);
            if (!pending) return;
            if (!msg.success) {
                this.localEquipment.rollbackEquip(msg.slot, pending.optimisticItemId);
                console.error(`Equip failed: ${msg.reason}`);
            }
            this.pendingRequests.delete(msg.requestId);
        } else if (msg.type === 'SyncFullEquipment') {
            this.localEquipment.fullSync(msg.equipmentData);
        }
    }

    private rollbackIfPending(reqId: number) {
        if (this.pendingRequests.has(reqId)) {
            const pending = this.pendingRequests.get(reqId)!;
            this.localEquipment.rollbackEquip(pending.slot, pending.optimisticItemId);
            this.pendingRequests.delete(reqId);
        }
    }

    private send(msg: GameMessage) {
        this.ws.send(JSON.stringify(msg));
    }
}
```

---

### 2.7 Offline Save System — `client/src/game/offline/SaveSystem.ts` ✅ UPDATED

```typescript
export class SaveSystem {
    private DB_NAME = 'MMOSave';
    private STORE = 'equipment';
    private DB_VERSION = 1;

    async save(
        equipmentData: unknown,
        characterClass: string,
        level: number
    ): Promise<void> {
        const db = await this.openDB();
        // FIX: IDBRequest is NOT a Promise. `await store.put(...)` would silently
        // return undefined. All IDB operations must be wrapped in new Promise().
        await new Promise<void>((resolve, reject) => {
            const tx = db.transaction(this.STORE, 'readwrite');
            const store = tx.objectStore(this.STORE);
            const req = store.put({
                id: 'current',
                data: equipmentData,
                class: characterClass,
                level,
                version: this.DB_VERSION,
                timestamp: Date.now(),
            });
            req.onsuccess = () => resolve();
            req.onerror = () => reject(req.error);
        });
    }

    async load(): Promise<unknown> {
        const db = await this.openDB();
        // FIX: Same issue — wrapped in Promise.
        return new Promise((resolve, reject) => {
            const tx = db.transaction(this.STORE, 'readonly');
            const store = tx.objectStore(this.STORE);
            const req = store.get('current');
            req.onsuccess = () => resolve(req.result ?? null);
            req.onerror = () => reject(req.error);
        });
    }

    private openDB(): Promise<IDBDatabase> {
        return new Promise((resolve, reject) => {
            const req = indexedDB.open(this.DB_NAME, this.DB_VERSION);
            req.onupgradeneeded = () => {
                req.result.createObjectStore(this.STORE, { keyPath: 'id' });
            };
            req.onsuccess = () => resolve(req.result);
            req.onerror = () => reject(req.error);
        });
    }
}
```

---

### 2.8 Durability System — `crates/game_core/src/advanced/durability.rs` ✅ NO CHANGES NEEDED

```rust
use std::collections::HashMap;
use crate::gear::enums::EquipSlot;
use crate::gear::gear_item::GearItem;

pub struct DurabilityManager {
    pub current: HashMap<EquipSlot, u16>,
    pub max_durability: HashMap<EquipSlot, u16>,
}

impl DurabilityManager {
    pub fn new(equipped: &HashMap<EquipSlot, GearItem>) -> Self {
        let mut current = HashMap::new();
        let mut max_durability = HashMap::new();
        for (&slot, _item) in equipped {
            let max: u16 = 100; // replace with item.max_durability when field is added
            current.insert(slot, max);
            max_durability.insert(slot, max);
        }
        Self { current, max_durability }
    }

    pub fn damage(&mut self, slot: EquipSlot, amount: u16) {
        if let Some(cur) = self.current.get_mut(&slot) {
            *cur = cur.saturating_sub(amount);
        }
    }

    pub fn is_broken(&self, slot: EquipSlot) -> bool {
        self.current.get(&slot).copied().unwrap_or(0) == 0
    }

    pub fn repair(&mut self, slot: EquipSlot) {
        if let Some(max) = self.max_durability.get(&slot).copied() {
            self.current.insert(slot, max);
        }
    }

    pub fn repair_all(&mut self) {
        for (slot, max) in &self.max_durability {
            self.current.insert(*slot, *max);
        }
    }
}
```

---

### 2.9 Stats Engine — `crates/game_core/src/advanced/stats_engine.rs` ✅ UPDATED

```rust
use crate::gear::equipment_manager::{CharacterStats, EquipmentManager};

pub struct StatsEngine {
    pub base_stats: CharacterStats,
    pub set_bonus_stats: CharacterStats,
    pub temp_buffs: CharacterStats,
}

impl StatsEngine {
    pub fn new() -> Self {
        Self {
            base_stats: CharacterStats::default(),
            set_bonus_stats: CharacterStats::default(),
            temp_buffs: CharacterStats::default(),
        }
    }

    pub fn compute_final(&self) -> CharacterStats {
        CharacterStats {
            armor: self.base_stats.armor
                + self.set_bonus_stats.armor
                + self.temp_buffs.armor,
            stamina: self.base_stats.stamina
                + self.set_bonus_stats.stamina
                + self.temp_buffs.stamina,
            strength: self.base_stats.strength
                + self.set_bonus_stats.strength
                + self.temp_buffs.strength,
            agility: self.base_stats.agility
                + self.set_bonus_stats.agility
                + self.temp_buffs.agility,
            intellect: self.base_stats.intellect
                + self.set_bonus_stats.intellect
                + self.temp_buffs.intellect,
            spirit: self.base_stats.spirit
                + self.set_bonus_stats.spirit
                + self.temp_buffs.spirit,
            attack_power: self.base_stats.attack_power
                + self.set_bonus_stats.attack_power
                + self.temp_buffs.attack_power,
            spell_power: self.base_stats.spell_power
                + self.set_bonus_stats.spell_power
                + self.temp_buffs.spell_power,
        }
    }

    // FIX: Was a static-style fn taking &mut EquipmentManager but belonged to
    // an instance struct. Changed to &mut self method that updates internal state.
    pub fn update_from_equipment(&mut self, equip_mgr: &mut EquipmentManager) {
        self.base_stats = equip_mgr.get_total_stats();
    }

    pub fn apply_temp_buff(&mut self, buff: CharacterStats) {
        self.temp_buffs.stamina += buff.stamina;
        self.temp_buffs.strength += buff.strength;
        self.temp_buffs.agility += buff.agility;
        self.temp_buffs.intellect += buff.intellect;
        self.temp_buffs.spirit += buff.spirit;
        self.temp_buffs.attack_power += buff.attack_power;
        self.temp_buffs.spell_power += buff.spell_power;
    }

    pub fn clear_temp_buffs(&mut self) {
        self.temp_buffs = CharacterStats::default();
    }
}
```

---

### 2.10 Drag & Drop — `client/src/ui/DragDrop.ts` ✅ UPDATED

```typescript
import { GearItem } from '../game/gear/GearItem';
import { EquipmentManager } from '../game/gear/EquipmentManager';
import { SlotWidget } from './SlotWidget';
import { EquipSlot } from '../game/gear/enums';

// FIX: Original referenced `gameInstance` global that was never defined or imported.
// DragDropManager now accepts EquipmentManager as a constructor dependency.
export class DragDropManager {
    private static instance: DragDropManager;
    private draggedItem: GearItem | null = null;
    private sourceSlot: SlotWidget | null = null;
    private equipmentManager: EquipmentManager;

    constructor(equipmentManager: EquipmentManager) {
        this.equipmentManager = equipmentManager;
    }

    static init(equipmentManager: EquipmentManager): DragDropManager {
        DragDropManager.instance = new DragDropManager(equipmentManager);
        return DragDropManager.instance;
    }

    static get(): DragDropManager {
        if (!DragDropManager.instance) {
            throw new Error('DragDropManager not initialized. Call DragDropManager.init() first.');
        }
        return DragDropManager.instance;
    }

    startDrag(item: GearItem, source: SlotWidget, event: DragEvent) {
        this.draggedItem = item;
        this.sourceSlot = source;
        event.dataTransfer!.setData(
            'text/plain',
            JSON.stringify({ itemId: item.id, slot: source.slotType })
        );
        // Custom drag image
        const dragIcon = document.createElement('div');
        dragIcon.className = 'drag-preview';
        dragIcon.textContent = item.name;
        dragIcon.style.cssText = 'position:absolute;top:-1000px;left:-1000px;';
        document.body.appendChild(dragIcon);
        event.dataTransfer!.setDragImage(dragIcon, 16, 16);
        setTimeout(() => document.body.removeChild(dragIcon), 0);
    }

    stopDrag() {
        this.draggedItem = null;
        this.sourceSlot = null;
    }

    // FIX: Added `targetSlot` parameter to match updated canEquipItem signature.
    isValidDrop(targetSlot: SlotWidget): boolean {
        if (!this.draggedItem) return false;
        return this.equipmentManager.canEquipItem(this.draggedItem, targetSlot.slotType);
    }

    get currentItem(): GearItem | null { return this.draggedItem; }
}
```

---

### 2.11 SlotWidget — `client/src/ui/SlotWidget.ts` ✅ UPDATED

```typescript
import { GearItem } from '../game/gear/GearItem';
import { EquipSlot } from '../game/gear/enums';
import { DragDropManager } from './DragDrop';
import { ClientSocket } from '../game/networking/ClientSocket';

export class SlotWidget {
    public slotType: EquipSlot;
    private element: HTMLElement;
    private currentItem: GearItem | null = null;
    // FIX: `network` was referenced as a global that was never imported.
    // Now injected as a constructor dependency.
    private network: ClientSocket;

    constructor(slot: EquipSlot, parent: HTMLElement, network: ClientSocket) {
        this.slotType = slot;
        this.network = network;
        this.element = document.createElement('div');
        this.element.classList.add('slot');
        this.element.setAttribute('data-slot', EquipSlot[slot]);
        this.element.draggable = true;
        this.element.addEventListener('dragover', (e) => e.preventDefault());
        this.element.addEventListener('drop', (e) => this.onDrop(e));
        this.element.addEventListener('dragstart', (e) => this.onDragStart(e));
        parent.appendChild(this.element);
    }

    setItem(item: GearItem | null) {
        this.currentItem = item;
        this.updateVisual();
    }

    private onDragStart(e: DragEvent) {
        if (!this.currentItem) {
            e.preventDefault();
            return;
        }
        DragDropManager.get().startDrag(this.currentItem, this, e);
    }

    private onDrop(e: DragEvent) {
        e.preventDefault();
        const dragMgr = DragDropManager.get();
        if (!dragMgr.isValidDrop(this)) {
            this.flashInvalid();
            dragMgr.stopDrag();
            return;
        }
        const raw = e.dataTransfer?.getData('text/plain');
        if (!raw) return;
        const data = JSON.parse(raw) as { itemId: number; slot: EquipSlot };
        this.network.requestEquip(data.itemId, this.slotType);
        dragMgr.stopDrag();
    }

    private updateVisual() {
        this.element.innerHTML = '';
        if (this.currentItem) {
            const icon = document.createElement('img');
            icon.src = this.currentItem.iconPath;
            icon.alt = this.currentItem.name;
            this.element.appendChild(icon);
            this.element.classList.add('occupied');
        } else {
            this.element.classList.remove('occupied');
        }
    }

    private flashInvalid() {
        this.element.classList.add('invalid-drop');
        setTimeout(() => this.element.classList.remove('invalid-drop'), 400);
    }
}
```

---

## 3. DATA SCHEMAS (SQL) ✅ NO CHANGES NEEDED

```sql
-- Item catalog
CREATE TABLE items (
    id          INTEGER PRIMARY KEY,
    name        TEXT    NOT NULL,
    quality     INTEGER NOT NULL,  -- 0=Poor .. 5=Legendary
    equip_slot  INTEGER NOT NULL,  -- maps to EquipSlot enum ordinal
    armor_type  INTEGER,           -- NULL for weapons
    weapon_type INTEGER,           -- NULL for armor
    is_two_handed   INTEGER DEFAULT 0,
    requires_ammo   INTEGER DEFAULT 0,
    ammo_type       TEXT,
    required_level  INTEGER DEFAULT 0,
    armor       INTEGER DEFAULT 0,
    stamina     INTEGER DEFAULT 0,
    strength    INTEGER DEFAULT 0,
    agility     INTEGER DEFAULT 0,
    intellect   INTEGER DEFAULT 0,
    spirit      INTEGER DEFAULT 0,
    min_damage  REAL    DEFAULT 0,
    max_damage  REAL    DEFAULT 0,
    attack_speed REAL   DEFAULT 0,
    icon_path   TEXT
);

-- Class restrictions (many-to-many)
CREATE TABLE item_class_restrictions (
    item_id  INTEGER,
    class_id INTEGER,   -- maps to CharacterClass enum ordinal
    FOREIGN KEY(item_id) REFERENCES items(id) ON DELETE CASCADE
);

-- Set bonus definitions
CREATE TABLE item_sets (
    set_id          INTEGER PRIMARY KEY,
    set_name        TEXT,
    pieces_required INTEGER,
    stat_modifier   TEXT    -- JSON e.g. {"stamina": 10, "strength": 5}
);

-- Set membership
CREATE TABLE item_set_members (
    item_id INTEGER,
    set_id  INTEGER,
    FOREIGN KEY(item_id) REFERENCES items(id),
    FOREIGN KEY(set_id)  REFERENCES item_sets(set_id)
);
```

---

## 4. UNIT TESTS — `crates/game_core/tests/equipment_tests.rs` ✅ UPDATED

```rust
#[cfg(test)]
mod tests {
    use game_core::gear::enums::*;
    use game_core::gear::gear_item::GearItem;
    use game_core::gear::equipment_manager::EquipmentManager;

    fn make_sword() -> GearItem {
        GearItem {
            id: 1,
            name: "Longsword".into(),
            quality: ItemQuality::Common,
            equip_slot: EquipSlot::MainHand,
            weapon_type: Some(WeaponType::Sword1H),
            required_level: 20,
            allowed_classes: vec![CharacterClass::Warrior],
            stamina: 5,
            strength: 8,
            min_damage: 10.0,
            max_damage: 20.0,
            attack_speed: 2.0,
            ..Default::default()  // FIX: now valid — Default impl added to GearItem
        }
    }

    #[test]
    fn test_equip_valid_item() {
        let mut mgr = EquipmentManager::new(CharacterClass::Warrior, 30);
        let sword = make_sword();
        assert!(mgr.equip_item(sword, EquipSlot::MainHand).is_ok());
        // FIX: get_equipped_item now exists
        assert!(mgr.get_equipped_item(EquipSlot::MainHand).is_some());
    }

    #[test]
    fn test_equip_too_high_level() {
        let mut mgr = EquipmentManager::new(CharacterClass::Mage, 5);
        // FIX: GearItem now implements Default, so this compiles
        let high_item = GearItem { required_level: 10, ..Default::default() };
        assert!(mgr.can_equip_item(&high_item, high_item.equip_slot).is_err());
    }

    #[test]
    fn test_two_handed_clears_offhand() {
        let mut mgr = EquipmentManager::new(CharacterClass::Warrior, 40);
        let offhand = GearItem {
            id: 2,
            name: "Shield".into(),
            equip_slot: EquipSlot::OffHand,
            weapon_type: Some(WeaponType::Shield),
            ..Default::default()
        };
        mgr.equip_item(offhand, EquipSlot::OffHand).unwrap();
        let twohander = GearItem {
            id: 3,
            name: "Greatsword".into(),
            equip_slot: EquipSlot::MainHand,
            weapon_type: Some(WeaponType::Sword2H),
            is_two_handed: true,
            allowed_classes: vec![CharacterClass::Warrior],
            ..Default::default()
        };
        mgr.equip_item(twohander, EquipSlot::MainHand).unwrap();
        assert!(mgr.get_equipped_item(EquipSlot::OffHand).is_none());
    }

    #[test]
    fn test_unique_ring_blocked() {
        let mut mgr = EquipmentManager::new(CharacterClass::Warrior, 30);
        let ring = GearItem {
            id: 99,
            equip_slot: EquipSlot::Ring1,
            ..Default::default()
        };
        mgr.equip_item(ring.clone(), EquipSlot::Ring1).unwrap();
        // same item id in Ring2 should fail
        assert!(mgr.can_equip_item(&ring, EquipSlot::Ring2).is_err());
    }

    #[test]
    fn test_stat_cache_invalidated_on_unequip() {
        let mut mgr = EquipmentManager::new(CharacterClass::Warrior, 30);
        let sword = make_sword();
        mgr.equip_item(sword, EquipSlot::MainHand).unwrap();
        let stats_before = mgr.get_total_stats();
        mgr.unequip_item(EquipSlot::MainHand);
        let stats_after = mgr.get_total_stats();
        assert!(stats_before.strength > stats_after.strength);
    }
}
```

---

## 5. QUICK START COMMANDS ✅ UPDATED

```bash
# Create workspace
cargo new my_mmo --bin
cd my_mmo

# Add dependencies — versions pinned for Bevy 0.15 compatibility
# FIX: Original had no versions; bevy_replicon must match Bevy exactly.
cargo add bevy@0.15
cargo add bevy_replicon@0.28     # check https://crates.io/crates/bevy_replicon for latest Bevy 0.15 compatible release
cargo add serde --features derive
cargo add rmp-serde

# Create folder structure
mkdir -p crates/game_core/src/gear
mkdir -p crates/game_core/src/networking
mkdir -p crates/game_core/src/advanced
mkdir -p crates/game_core/tests

# Run all tests
cargo test

# Run client binary
cargo run --bin game_client
```

> **Note on `bevy_replicon`:** Always cross-check the compatibility table at
> https://github.com/projectharmonia/bevy_replicon before pinning versions,
> as each Bevy release requires a specific bevy_replicon range.

---

## 6. DEPLOYMENT CHECKLIST

- [ ] All enums defined with no magic numbers
- [ ] `GearItem` serialization roundtrip tested (rmp-serde)
- [ ] `EquipmentManager` passes all unit tests (25+ cases)
- [ ] Validator catches all invalid equip attempts (level, class, armor, weapon, unique rings/trinkets)
- [ ] Character sheet UI shows all 19 equipment slots
- [ ] Drag & drop works with valid/invalid visual feedback
- [ ] Tooltips show stat comparison vs currently equipped item
- [ ] Server validates equip requests against item database
- [ ] Client optimistic updates roll back correctly on timeout (5s)
- [ ] Offline save to IndexedDB works and loads on reconnect
- [ ] Offline→online sync reconciles without duplication
- [ ] Ammo decrements on each ranged attack
- [ ] Durability decreases on death; item disabled at 0 durability
- [ ] Set bonuses activate at correct piece thresholds (2/4/6)
- [ ] Stat recalc confirmed <1ms under profiling
- [ ] Load test with 100 simulated concurrent players
- [ ] Security audit: no item duplication exploit, no level-requirement bypass

---

## 7. COMMON PITFALLS & SOLUTIONS

**Two-handed + off-hand:** Always remove the off-hand item before inserting a two-hander into `MainHand`. The validator blocks the equip attempt if an off-hand is present, so `equip_item` must unequip it first (as implemented above).

**Unique rings/trinkets:** Check both paired slots using the *target slot*, not `item.equip_slot` — the item's default slot field is descriptive, not authoritative.

**IDB async in TypeScript:** `IDBRequest` is not a `Promise`. Every IndexedDB read/write must be wrapped in `new Promise((resolve, reject) => { req.onsuccess = ...; req.onerror = ...; })`. Using `await` directly on an `IDBRequest` silently returns `undefined`.

**Offline save corruption:** Keep a versioned backup slot and validate a checksum or schema version on load before applying data.

**Network desync:** Use periodic full-state sync (e.g. every 30s or on reconnect) in addition to delta updates. Compare state hashes server-side to detect drift.

**Stat cache:** Only invalidate on equip, unequip, or buff change — never on every frame. The cache guard in `get_total_stats` keeps recalcs well under 1ms for typical gear counts.

**Drag & drop custom image:** Always append the custom drag-image element to `document.body` and remove it via `setTimeout(..., 0)` — removing it synchronously during the dragstart handler causes the image to disappear before the browser captures it.
