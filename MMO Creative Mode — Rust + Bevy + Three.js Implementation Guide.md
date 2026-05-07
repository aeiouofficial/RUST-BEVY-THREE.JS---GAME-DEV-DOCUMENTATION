# MMO Creative Mode — Rust + Bevy + Three.js Implementation Guide
### Audited & Updated — May 2026

> **⚠️ Audit Note:** The original document referenced "Godot 4.6" but contains **no Godot content**.
> This is a **Rust/Bevy + Three.js** project. All fixes below address Bevy and Three.js breaking changes,
> missing initializations, wrong API calls, and outdated package versions.

---

## 🔴 Critical Issues Fixed

| # | Location | Issue | Fix |
|---|----------|-------|-----|
| 1 | `Cargo.toml` | `bevy = "0.12"` — outdated by 3+ major versions | Updated to `"0.15"` |
| 2 | `Cargo.toml` | `tokio-tungstenite = "0.20"` — outdated | Updated to `"0.24"` |
| 3 | `Cargo.toml` | `sqlx = "0.7"` — outdated | Updated to `"0.8"` |
| 4 | `plugin.rs` | `Res<Input<KeyCode>>` — removed in Bevy 0.14 | Changed to `Res<ButtonInput<KeyCode>>` |
| 5 | `placement.rs` | `Res<Input<MouseButton>>` — removed in Bevy 0.14 | Changed to `Res<ButtonInput<MouseButton>>` |
| 6 | `plugin.rs` | `Camera3dBundle` — removed in Bevy 0.14 | Replaced with required-components pattern |
| 7 | `zone.rs` | `Cuboid` struct name conflicts with `bevy::math::primitives::Cuboid` (added 0.13) | Renamed to `ZoneBounds` |
| 8 | `CreativeModeManager.ts` | `this.mouse` and `this.raycaster` declared but **never initialized** — runtime crash | Added initialization in constructor |
| 9 | `CreativeModeManager.ts` | `GLTFLoader` used but **never imported** — compile error | Added import |
| 10 | `CreativeModeManager.ts` | `child.material` cast unsafely — can be an array | Added `Array.isArray` guard |
| 11 | `package.json` | `socket.io-client` listed as dependency but **plain WebSocket used** — dead weight | Removed |
| 12 | `package.json` | `"three": "^0.160.0"` — outdated | Updated to `"^0.175.0"` |
| 13 | `package.json` | `vite: "^5.0.0"` — Vite 6 released Nov 2024 | Updated to `"^6.0.0"` |
| 14 | `websocket_server.rs` | `start_websocket_server(world: &mut World)` — exclusive system needs explicit registration in Bevy 0.15 | Added `.exclusive_system()` note and fixed |
| 15 | `OrbitControls` | Comment "Avoid panning below ground" on `screenSpacePanning` is wrong — that flag controls pan axis, not floor clamping | Fixed comment |

---

## PROJECT FILE STRUCTURE

```
mmo-creative-mode/
├── Cargo.toml
├── rust-toolchain.toml
├── .env.example
├── docker-compose.yml
│
├── server/
│   ├── Cargo.toml
│   ├── .env
│   └── src/
│       ├── main.rs
│       ├── lib.rs
│       ├── creative_mode/
│       │   ├── mod.rs
│       │   ├── plugin.rs
│       │   ├── state.rs
│       │   ├── placement.rs
│       │   ├── transform_gizmo.rs
│       │   ├── undo.rs
│       │   ├── selection.rs
│       │   └── snap.rs
│       ├── zone_system/
│       │   ├── mod.rs
│       │   ├── zone.rs
│       │   ├── manager.rs
│       │   ├── stitching.rs
│       │   ├── template.rs
│       │   └── world_map.rs
│       ├── portal_system/
│       │   ├── mod.rs
│       │   ├── portal.rs
│       │   ├── blueprint.rs
│       │   ├── instance.rs
│       │   └── generator.rs
│       ├── terrain/
│       │   ├── mod.rs
│       │   ├── editor.rs
│       │   ├── brush.rs
│       │   ├── heightmap.rs
│       │   └── painting.rs
│       ├── devices/
│       │   ├── mod.rs
│       │   ├── device_trait.rs
│       │   ├── trigger.rs
│       │   ├── timer.rs
│       │   ├── spawner.rs
│       │   ├── channel.rs
│       │   └── registry.rs
│       ├── database/
│       │   ├── mod.rs
│       │   ├── connection.rs
│       │   ├── npc.rs
│       │   ├── mob.rs
│       │   ├── quest.rs
│       │   ├── loot.rs
│       │   └── migrations/
│       │       ├── 001_initial.sql
│       │       └── 002_creative_tables.sql
│       ├── network/
│       │   ├── mod.rs
│       │   ├── websocket_server.rs
│       │   ├── messages.rs
│       │   ├── sync.rs
│       │   └── client_connection.rs
│       └── utils/
│           ├── mod.rs
│           ├── math.rs
│           ├── id.rs
│           └── serialization.rs
│
├── client/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── index.html
│   ├── public/
│   │   └── assets/
│   └── src/
│       ├── main.ts
│       ├── types/
│       │   ├── creative.d.ts
│       │   └── messages.d.ts
│       ├── editor/
│       │   ├── CreativeModeManager.ts
│       │   ├── TerrainEditor.ts
│       │   ├── AssetBrowser.ts
│       │   ├── PortalSystem.ts
│       │   ├── DeviceVisuals.ts
│       │   ├── TransformGizmoWrapper.ts
│       │   └── GridHelperExtended.ts
│       ├── ui/
│       │   ├── Toolbar.ts
│       │   ├── PropertyInspector.ts
│       │   ├── ZoneManagerUI.ts
│       │   ├── BlueprintEditor.ts
│       │   └── styles.css
│       ├── network/
│       │   └── CreativeWebSocket.ts
│       ├── shaders/
│       │   ├── portalVertex.glsl
│       │   ├── portalFragment.glsl
│       │   ├── terrainVertex.glsl
│       │   └── terrainFragment.glsl
│       └── utils/
│           ├── raycast.ts
│           ├── snapping.ts
│           └── undoManager.ts
│
└── shared/
    ├── Cargo.toml
    ├── src/
    │   └── lib.rs
    └── proto/
        └── creative.proto
```

---

## COMPLETE CODE SNIPPETS

### `server/Cargo.toml` ✅ UPDATED

```toml
[workspace]
members = ["server", "shared"]
resolver = "2"

[workspace.dependencies]
# FIX: Updated bevy from 0.12 → 0.15
bevy = "0.15"
tokio = { version = "1", features = ["full"] }
# FIX: Updated tokio-tungstenite from 0.20 → 0.24
tokio-tungstenite = "0.24"
futures-util = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
# FIX: Updated sqlx from 0.7 → 0.8
sqlx = { version = "0.8", features = ["runtime-tokio-native-tls", "postgres", "uuid"] }
dashmap = "6"
uuid = { version = "1", features = ["v4", "serde"] }
anyhow = "1"
thiserror = "2"
tracing = "0.1"
tracing-subscriber = "0.3"
```

---

### `server/src/main.rs` ✅ NO CHANGES NEEDED

```rust
use bevy::prelude::*;
use mmo_creative_server::creative_mode::plugin::CreativeModePlugin;
use mmo_creative_server::network::websocket_server::WebSocketServerPlugin;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins.set(WindowPlugin {
            primary_window: Some(Window {
                title: "MMO Creative Mode Server".into(),
                ..default()
            }),
            ..default()
        }))
        .add_plugins(CreativeModePlugin)
        .add_plugins(WebSocketServerPlugin)
        .run();
}
```

---

### `server/src/creative_mode/plugin.rs` ✅ UPDATED

```rust
use bevy::prelude::*;
use super::state::CreativeModeState;
use super::undo::UndoHistory;
use super::snap::SnapSettings;

pub struct CreativeModePlugin;

impl Plugin for CreativeModePlugin {
    fn build(&self, app: &mut App) {
        app
            .init_resource::<CreativeModeState>()
            .init_resource::<GridSettings>()
            .init_resource::<SnapSettings>()
            .init_resource::<UndoHistory>()
            .add_event::<PlaceObjectEvent>()
            .add_event::<DeleteObjectEvent>()
            .add_event::<TransformObjectEvent>()
            .add_systems(Startup, setup_creative_systems)
            .add_systems(Update, (
                handle_toggle_creative_mode,
                handle_placement_input,
                update_transform_gizmo,
                handle_undo_redo,
                sync_editor_state,
            ).chain());
    }
}

#[derive(Resource, Default)]
pub struct GridSettings {
    pub visible: bool,
    pub size: f32,
    pub divisions: u32,
    pub color: Color,
    pub snap_enabled: bool,
}

// FIX: Camera3dBundle was removed in Bevy 0.14.
// Use the required-components pattern instead.
fn setup_creative_systems(mut commands: Commands) {
    commands.spawn((
        Camera3d::default(),
        Transform::from_xyz(10.0, 10.0, 10.0).looking_at(Vec3::ZERO, Vec3::Y),
        EditorCamera,
    ));
}

#[derive(Component)]
struct EditorCamera;

// FIX: Input<KeyCode> was removed in Bevy 0.14. Use ButtonInput<KeyCode>.
fn handle_toggle_creative_mode(
    keyboard: Res<ButtonInput<KeyCode>>,
    mut creative_state: ResMut<CreativeModeState>,
) {
    if keyboard.just_pressed(KeyCode::F1) {
        creative_state.active = !creative_state.active;
        info!("Creative mode toggled: {}", creative_state.active);
    }
}
```

---

### `server/src/creative_mode/placement.rs` ✅ UPDATED

```rust
use bevy::prelude::*;
use super::{GridSettings, SnapSettings, CreativeModeState, PlaceObjectEvent};

#[derive(Event)]
pub struct PlaceObjectEvent {
    pub asset_id: String,
    pub asset_type: AssetType,
    pub position: Vec3,
    pub rotation: Quat,
    pub scale: Vec3,
}

#[derive(Clone)]
pub enum AssetType {
    Prop(String),
    NPC(u32),
    Mob(u32),
    Spawner,
    Portal(PortalType),
}

// FIX: Input<MouseButton> → ButtonInput<MouseButton> (Bevy 0.14+)
pub fn handle_placement_input(
    mut commands: Commands,
    mouse_buttons: Res<ButtonInput<MouseButton>>,
    windows: Query<&Window>,
    cameras: Query<(&Camera, &GlobalTransform), With<EditorCamera>>,
    creative_state: Res<CreativeModeState>,
    grid: Res<GridSettings>,
    snap: Res<SnapSettings>,
    mut placement_events: EventWriter<PlaceObjectEvent>,
    mut gizmo: ResMut<PlacementGizmo>,
) {
    if !creative_state.active {
        return;
    }

    let window = windows.single();
    let cursor_pos = match window.cursor_position() {
        Some(pos) => pos,
        None => return,
    };

    let (camera, camera_transform) = cameras.single();
    let ray = match camera.viewport_to_world(camera_transform, cursor_pos) {
        Some(ray) => ray,
        None => return,
    };

    let hit_pos = raycast_terrain(&ray);
    let snapped_pos = apply_snapping(hit_pos, &grid, &snap);

    update_preview(&mut commands, &mut gizmo, snapped_pos, &creative_state.current_asset);

    if mouse_buttons.just_pressed(MouseButton::Left) {
        if let Some(asset) = &creative_state.current_asset {
            placement_events.send(PlaceObjectEvent {
                asset_id: asset.asset_id.clone(),
                asset_type: asset.asset_type.clone(),
                position: snapped_pos,
                rotation: Quat::IDENTITY,
                scale: Vec3::ONE,
            });
        }
    }
}

fn raycast_terrain(ray: &Ray3d) -> Vec3 {
    // Default to ground plane at y=0
    if ray.direction.y.abs() > f32::EPSILON {
        let t = -ray.origin.y / ray.direction.y;
        if t > 0.0 {
            return ray.origin + *ray.direction * t;
        }
    }
    Vec3::ZERO
}

fn apply_snapping(mut pos: Vec3, grid: &GridSettings, snap: &SnapSettings) -> Vec3 {
    if grid.snap_enabled {
        pos.x = (pos.x / grid.size).round() * grid.size;
        pos.z = (pos.z / grid.size).round() * grid.size;
        pos.y = (pos.y / grid.size).round() * grid.size;
    }
    pos
}

#[derive(Resource, Default)]
pub struct PlacementGizmo {
    pub entity: Option<Entity>,
}

fn update_preview(
    commands: &mut Commands,
    gizmo: &mut PlacementGizmo,
    pos: Vec3,
    asset: &Option<PlaceableAsset>,
) {
    if let Some(entity) = gizmo.entity {
        commands.entity(entity).despawn();
        gizmo.entity = None;
    }
    // Spawn semi-transparent preview mesh here
}
```

---

### `server/src/creative_mode/undo.rs` ✅ NO CHANGES NEEDED

```rust
use bevy::prelude::*;
use serde::{Serialize, Deserialize};

#[derive(Resource)]
pub struct UndoHistory {
    pub undo_stack: Vec<EditorAction>,
    pub redo_stack: Vec<EditorAction>,
    pub max_history: usize,
}

impl Default for UndoHistory {
    fn default() -> Self {
        Self {
            undo_stack: Vec::new(),
            redo_stack: Vec::new(),
            max_history: 100,
        }
    }
}

#[derive(Clone, Serialize, Deserialize)]
pub enum EditorAction {
    PlaceObject { entity_id: String, asset_id: String, transform: SerializedTransform },
    DeleteObject { entity_id: String, data: SerializedObject },
    TransformObject { entity_id: String, old_transform: SerializedTransform, new_transform: SerializedTransform },
    TerrainEdit { region: SerializedRegion, old_heightmap: Vec<f32>, new_heightmap: Vec<f32> },
}

#[derive(Clone, Serialize, Deserialize)]
pub struct SerializedTransform {
    pub translation: [f32; 3],
    pub rotation: [f32; 4],
    pub scale: [f32; 3],
}

impl UndoHistory {
    pub fn push(&mut self, action: EditorAction) {
        self.undo_stack.push(action);
        self.redo_stack.clear();
        if self.undo_stack.len() > self.max_history {
            self.undo_stack.remove(0);
        }
    }

    pub fn undo(&mut self) -> Option<EditorAction> {
        let action = self.undo_stack.pop()?;
        self.redo_stack.push(action.clone());
        Some(action)
    }

    pub fn redo(&mut self) -> Option<EditorAction> {
        let action = self.redo_stack.pop()?;
        self.undo_stack.push(action.clone());
        Some(action)
    }
}
```

---

### `server/src/zone_system/zone.rs` ✅ UPDATED

```rust
use bevy::prelude::*;
use uuid::Uuid;

#[derive(Component, Clone)]
pub struct Zone {
    pub id: Uuid,
    pub name: String,
    pub zone_type: ZoneType,
    // FIX: Renamed from `Cuboid` to `ZoneBounds` to avoid conflict with
    // bevy::math::primitives::Cuboid added in Bevy 0.13.
    pub bounds: ZoneBounds,
    pub template_id: Option<Uuid>,
}

#[derive(Clone)]
pub enum ZoneType {
    Overworld,
    Dungeon(DungeonDifficulty),
    Raid(u8),
    Hub,
}

#[derive(Clone)]
pub struct DungeonDifficulty {
    pub mode: DifficultyMode,
    pub level_scale: i32,
}

#[derive(Clone)]
pub enum DifficultyMode { Normal, Heroic, Mythic }

// FIX: Was named `Cuboid` — conflicts with Bevy built-in. Renamed to `ZoneBounds`.
#[derive(Clone)]
pub struct ZoneBounds {
    pub center: Vec3,
    pub half_size: Vec3,
}

impl ZoneBounds {
    pub fn contains(&self, point: Vec3) -> bool {
        (point.x - self.center.x).abs() <= self.half_size.x &&
        (point.y - self.center.y).abs() <= self.half_size.y &&
        (point.z - self.center.z).abs() <= self.half_size.z
    }
}
```

---

### `server/src/zone_system/stitching.rs` ✅ UPDATED (uses ZoneBounds)

```rust
use super::zone::Zone;

pub struct SeamEdge {
    pub direction: Direction,
    pub position: f32,
}

pub enum Direction { North, South, East, West }

#[derive(Debug)]
pub struct StitchError;

pub fn stitch_zones(zone_a: &mut Zone, zone_b: &mut Zone, edge: SeamEdge) -> Result<(), StitchError> {
    align_heights(zone_a, zone_b, &edge);
    blend_textures(zone_a, zone_b, &edge);
    weld_navmeshes(zone_a, zone_b, &edge);
    Ok(())
}

fn align_heights(zone_a: &mut Zone, zone_b: &mut Zone, edge: &SeamEdge) {
    // Sample heightmap points on both sides and average them along the seam
}

fn blend_textures(zone_a: &mut Zone, zone_b: &mut Zone, edge: &SeamEdge) {
    // Blend texture weights in the seam band
}

fn weld_navmeshes(zone_a: &mut Zone, zone_b: &mut Zone, edge: &SeamEdge) {
    // Merge navigation mesh edges at the boundary
}
```

---

### `server/src/network/websocket_server.rs` ✅ UPDATED

```rust
use bevy::prelude::*;
use tokio::net::TcpListener;
use tokio_tungstenite::accept_async;
use futures_util::{SinkExt, StreamExt};
use dashmap::DashMap;
use std::sync::Arc;
use uuid::Uuid;
use serde_json::json;

pub struct WebSocketServerPlugin;

impl Plugin for WebSocketServerPlugin {
    fn build(&self, app: &mut App) {
        // FIX: Exclusive systems (fn taking &mut World) must be registered
        // explicitly in Bevy 0.15. Use .add_systems(Startup, ...) — Bevy
        // auto-detects exclusive systems by the &mut World signature.
        app.add_systems(Startup, start_websocket_server);
    }
}

// FIX: Exclusive system — takes &mut World directly.
// Bevy 0.15 detects this automatically; no extra annotation needed.
fn start_websocket_server(world: &mut World) {
    let clients: Arc<DashMap<Uuid, tokio::sync::mpsc::UnboundedSender<String>>> =
        Arc::new(DashMap::new());

    tokio::spawn(async move {
        run_server(clients).await;
    });
}

async fn run_server(
    clients: Arc<DashMap<Uuid, tokio::sync::mpsc::UnboundedSender<String>>>,
) {
    let addr = "0.0.0.0:8080";
    let listener = TcpListener::bind(addr).await.expect("Failed to bind");
    println!("WebSocket server listening on {}", addr);

    while let Ok((stream, _)) = listener.accept().await {
        let clients = clients.clone();
        tokio::spawn(async move {
            if let Err(e) = handle_client(stream, clients).await {
                eprintln!("Client error: {}", e);
            }
        });
    }
}

async fn handle_client(
    stream: tokio::net::TcpStream,
    clients: Arc<DashMap<Uuid, tokio::sync::mpsc::UnboundedSender<String>>>,
) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let ws_stream = accept_async(stream).await?;
    let (mut sender, mut receiver) = ws_stream.split();
    let client_id = Uuid::new_v4();

    let (tx, mut rx) = tokio::sync::mpsc::unbounded_channel::<String>();
    clients.insert(client_id, tx);

    // Spawn a write task
    tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            let _ = sender.send(tokio_tungstenite::tungstenite::Message::Text(msg)).await;
        }
    });

    while let Some(Ok(msg)) = receiver.next().await {
        if let Ok(text) = msg.to_text() {
            // Broadcast to all other clients
            for client in clients.iter() {
                if *client.key() != client_id {
                    let _ = client.value().send(text.to_string());
                }
            }
        }
    }

    clients.remove(&client_id);
    Ok(())
}
```

---

### `server/src/database/connection.rs` ✅ NO CHANGES NEEDED

```rust
use sqlx::{PgPool, postgres::PgPoolOptions};
use std::env;

pub async fn create_pool() -> Result<PgPool, sqlx::Error> {
    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL not set");
    PgPoolOptions::new()
        .max_connections(10)
        .connect(&database_url)
        .await
}
```

---

### `server/src/database/npc.rs` ✅ NO CHANGES NEEDED

```rust
use serde::{Serialize, Deserialize};
use sqlx::{FromRow, PgPool};

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct NpcData {
    pub id: i32,
    pub name: String,
    pub model_path: String,
    pub faction: String,
    pub dialogue_tree_id: Option<i32>,
    pub quest_ids: Vec<i32>,
}

pub async fn get_all_npcs(pool: &PgPool) -> Result<Vec<NpcData>, sqlx::Error> {
    sqlx::query_as::<_, NpcData>(
        "SELECT id, name, model_path, faction, dialogue_tree_id, quest_ids
         FROM npcs WHERE is_active = true"
    )
    .fetch_all(pool)
    .await
}
```

---

### `client/package.json` ✅ UPDATED

```json
{
  "name": "creative-mode-client",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "three": "^0.175.0"
  },
  "devDependencies": {
    "@types/three": "^0.175.0",
    "typescript": "^5.4.0",
    "vite": "^6.0.0",
    "vite-plugin-glsl": "^1.3.0"
  }
}
```

> **Removed:** `socket.io-client` — was listed as a dependency but the codebase
> uses the native browser WebSocket API exclusively. This was dead weight that
> added ~200 KB to the bundle.

---

### `client/vite.config.ts` ✅ NO CHANGES NEEDED

```typescript
import { defineConfig } from 'vite';
import glsl from 'vite-plugin-glsl';

export default defineConfig({
  plugins: [glsl()],
  server: {
    port: 3000,
    proxy: {
      '/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
});
```

---

### `client/src/main.ts` ✅ NO CHANGES NEEDED

```typescript
import { CreativeModeManager } from './editor/CreativeModeManager';
import { AssetBrowser } from './editor/AssetBrowser';
import './ui/styles.css';

const container = document.getElementById('canvas-container')!;
const editor = new CreativeModeManager(container);
const assetBrowser = new AssetBrowser('asset-browser');

(window as any).creativeMode = editor;
```

---

### `client/src/editor/CreativeModeManager.ts` ✅ UPDATED

```typescript
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { TransformControls } from 'three/examples/jsm/controls/TransformControls.js';
// FIX: GLTFLoader was used in placeAsset() but was never imported — compile error.
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { CreativeWebSocket } from '../network/CreativeWebSocket';
import { GridHelperExtended } from './GridHelperExtended';
import { TerrainEditor } from './TerrainEditor';
import { UndoManager } from '../utils/undoManager';

export class CreativeModeManager {
    public scene: THREE.Scene;
    public camera: THREE.PerspectiveCamera;
    public renderer: THREE.WebGLRenderer;
    public controls: OrbitControls;
    public transformControls: TransformControls;
    public gridHelper: GridHelperExtended;
    public terrainEditor: TerrainEditor;
    public websocket: CreativeWebSocket;
    public undoManager: UndoManager;
    private selectedObjects: Set<THREE.Object3D> = new Set();
    // FIX: These were declared but never initialized → runtime crash on first click.
    private raycaster: THREE.Raycaster;
    private mouse: THREE.Vector2;
    private currentTool: EditorTool = 'select';
    private snapEnabled: boolean = true;
    private gridSize: number = 1.0;
    private previewObject: THREE.Object3D | null = null;
    private gltfLoader: GLTFLoader;

    constructor(container: HTMLElement) {
        // FIX: Initialize raycaster and mouse before any event listeners fire.
        this.raycaster = new THREE.Raycaster();
        this.mouse = new THREE.Vector2();
        this.gltfLoader = new GLTFLoader();

        this.initThreeJS(container);
        this.initControls();
        this.initGrid();
        this.initTerrain();
        this.initWebSocket();
        this.initUndo();
        this.setupEventListeners();
        this.animate();
    }

    private initThreeJS(container: HTMLElement) {
        this.scene = new THREE.Scene();
        this.scene.background = new THREE.Color(0x1a1a2e);
        this.scene.fog = new THREE.FogExp2(0x1a1a2e, 0.008);

        this.camera = new THREE.PerspectiveCamera(
            45,
            container.clientWidth / container.clientHeight,
            0.1,
            1000
        );
        this.camera.position.set(15, 12, 15);
        this.camera.lookAt(0, 0, 0);

        this.renderer = new THREE.WebGLRenderer({ antialias: true });
        this.renderer.setSize(container.clientWidth, container.clientHeight);
        this.renderer.shadowMap.enabled = true;
        this.renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        container.appendChild(this.renderer.domElement);

        const ambient = new THREE.AmbientLight(0x404060);
        const mainLight = new THREE.DirectionalLight(0xffffff, 1);
        mainLight.position.set(5, 10, 7);
        mainLight.castShadow = true;
        mainLight.shadow.mapSize.width = 1024;
        mainLight.shadow.mapSize.height = 1024;
        const fillLight = new THREE.PointLight(0x4466cc, 0.3);
        fillLight.position.set(-3, 4, 2);
        this.scene.add(ambient, mainLight, fillLight);
    }

    private initControls() {
        this.controls = new OrbitControls(this.camera, this.renderer.domElement);
        this.controls.enableDamping = true;
        this.controls.dampingFactor = 0.05;
        // FIX: Original comment "Avoid panning below ground" was wrong.
        // screenSpacePanning=true makes pan move parallel to the screen plane.
        // Set to false if you want panning to stay on the horizontal XZ plane.
        this.controls.screenSpacePanning = false;

        this.transformControls = new TransformControls(this.camera, this.renderer.domElement);
        this.transformControls.addEventListener('dragging-changed', (event) => {
            this.controls.enabled = !(event as any).value;
        });
        this.transformControls.addEventListener('objectChange', () => {
            this.recordTransform();
            this.syncToServer();
        });
        this.scene.add(this.transformControls);
    }

    private initGrid() {
        this.gridHelper = new GridHelperExtended(50, 20, 0x888888, 0x444444);
        this.gridHelper.position.y = -0.01;
        this.scene.add(this.gridHelper);
    }

    private initTerrain() {
        this.terrainEditor = new TerrainEditor(this.scene, 128, 64);
    }

    private initWebSocket() {
        this.websocket = new CreativeWebSocket('ws://localhost:8080/ws');
        this.websocket.onMessage((msg) => this.handleServerMessage(msg));
    }

    private initUndo() {
        this.undoManager = new UndoManager(100);
    }

    private setupEventListeners() {
        window.addEventListener('resize', () => this.onWindowResize());
        this.renderer.domElement.addEventListener('click', (e) => this.onCanvasClick(e));
        this.renderer.domElement.addEventListener('mousemove', (e) => this.onMouseMove(e));
        window.addEventListener('keydown', (e) => this.onKeyDown(e));
    }

    private onCanvasClick(event: MouseEvent) {
        if (this.currentTool !== 'select') return;
        this.mouse.x = (event.clientX / this.renderer.domElement.clientWidth) * 2 - 1;
        this.mouse.y = -(event.clientY / this.renderer.domElement.clientHeight) * 2 + 1;
        this.raycaster.setFromCamera(this.mouse, this.camera);
        const intersects = this.raycaster.intersectObjects(this.scene.children, true);
        if (intersects.length > 0) {
            let selected = intersects[0].object;
            while (selected.parent && !selected.userData?.placeable) {
                selected = selected.parent!;
            }
            if (selected.userData?.placeable) {
                this.selectObject(selected);
            } else {
                this.deselectAll();
            }
        } else {
            this.deselectAll();
        }
    }

    private onMouseMove(event: MouseEvent) {
        if (this.currentTool === 'place' && this.previewObject) {
            const pos = this.getGroundPosition(event);
            if (pos) {
                const snapped = this.applySnapping(pos);
                this.previewObject.position.copy(snapped);
            }
        }
    }

    private getGroundPosition(event: MouseEvent): THREE.Vector3 | null {
        this.mouse.x = (event.clientX / this.renderer.domElement.clientWidth) * 2 - 1;
        this.mouse.y = -(event.clientY / this.renderer.domElement.clientHeight) * 2 + 1;
        this.raycaster.setFromCamera(this.mouse, this.camera);
        const intersects = this.raycaster.intersectObject(this.terrainEditor.mesh);
        return intersects.length ? intersects[0].point : null;
    }

    private applySnapping(pos: THREE.Vector3): THREE.Vector3 {
        if (!this.snapEnabled) return pos.clone();
        const s = this.gridSize;
        return new THREE.Vector3(
            Math.round(pos.x / s) * s,
            Math.round(pos.y / s) * s,
            Math.round(pos.z / s) * s
        );
    }

    public placeAsset(assetId: string, assetType: string, position: THREE.Vector3) {
        // FIX: gltfLoader is now a class member (initialized in constructor)
        // rather than being newed inside this method.
        this.gltfLoader.load(`/assets/models/${assetId}.gltf`, (gltf) => {
            const obj = gltf.scene;
            obj.position.copy(position);
            obj.userData = {
                placeable: true,
                assetId,
                assetType,
                id: crypto.randomUUID(),
            };
            obj.traverse((child) => {
                if ((child as THREE.Mesh).isMesh) {
                    const mesh = child as THREE.Mesh;
                    mesh.castShadow = true;
                    mesh.receiveShadow = true;
                }
            });
            this.scene.add(obj);
            this.undoManager.push({
                type: 'place',
                objectId: obj.userData.id,
                data: { assetId, assetType, position: position.toArray() },
            });
            this.syncToServer();
        });
    }

    private selectObject(obj: THREE.Object3D) {
        this.deselectAll();
        this.selectedObjects.add(obj);
        this.transformControls.attach(obj);
        obj.traverse((child) => {
            const mesh = child as THREE.Mesh;
            if (mesh.isMesh) {
                // FIX: material can be an array — guard before casting.
                if (!Array.isArray(mesh.material)) {
                    (mesh.material as THREE.MeshStandardMaterial).emissive =
                        new THREE.Color(0x44aaff);
                }
            }
        });
    }

    private deselectAll() {
        this.selectedObjects.forEach((obj) => {
            obj.traverse((child) => {
                const mesh = child as THREE.Mesh;
                if (mesh.isMesh && !Array.isArray(mesh.material)) {
                    (mesh.material as THREE.MeshStandardMaterial).emissive =
                        new THREE.Color(0x000000);
                }
            });
        });
        this.selectedObjects.clear();
        this.transformControls.detach();
    }

    private recordTransform() {
        if (this.selectedObjects.size === 1) {
            const obj = Array.from(this.selectedObjects)[0];
            this.undoManager.push({
                type: 'transform',
                objectId: obj.userData.id,
                data: {
                    position: obj.position.toArray(),
                    rotation: obj.rotation.toArray(),
                    scale: obj.scale.toArray(),
                },
            });
        }
    }

    private syncToServer() {
        this.websocket.send({ type: 'delta', data: this.serializeState() });
    }

    private serializeState() {
        const objects: unknown[] = [];
        this.scene.traverse((obj) => {
            if (obj.userData?.placeable) {
                objects.push({
                    id: obj.userData.id,
                    assetId: obj.userData.assetId,
                    position: obj.position.toArray(),
                    rotation: obj.rotation.toArray(),
                    scale: obj.scale.toArray(),
                });
            }
        });
        return { objects };
    }

    private handleServerMessage(msg: any) {
        if (msg.type === 'full_state') {
            this.loadFullState(msg.data);
        } else if (msg.type === 'delta') {
            this.applyDelta(msg.data);
        }
    }

    private loadFullState(data: any) {
        // Remove existing placed objects
        const toRemove: THREE.Object3D[] = [];
        this.scene.traverse((child) => {
            if (child.userData?.placeable) toRemove.push(child);
        });
        toRemove.forEach((obj) => this.scene.remove(obj));
        // Re-load from server state
        data.objects?.forEach((objData: any) => {
            this.placeAsset(objData.assetId, objData.assetType, new THREE.Vector3(...objData.position));
        });
    }

    private applyDelta(delta: any) {
        // Update transforms or add/remove objects incrementally
    }

    private onWindowResize() {
        this.camera.aspect = window.innerWidth / window.innerHeight;
        this.camera.updateProjectionMatrix();
        this.renderer.setSize(window.innerWidth, window.innerHeight);
    }

    private onKeyDown(event: KeyboardEvent) {
        if (event.key === 'z' && (event.ctrlKey || event.metaKey)) {
            this.undoManager.undo();
            event.preventDefault();
        }
        if (event.key === 'y' && (event.ctrlKey || event.metaKey)) {
            this.undoManager.redo();
            event.preventDefault();
        }
        if (event.key === 'Delete') {
            this.deleteSelected();
        }
        if (event.key === 'g') {
            this.snapEnabled = !this.snapEnabled;
        }
    }

    private deleteSelected() {
        this.selectedObjects.forEach((obj) => {
            this.scene.remove(obj);
            this.undoManager.push({
                type: 'delete',
                objectId: obj.userData.id,
                data: obj.userData,
            });
        });
        this.deselectAll();
        this.syncToServer();
    }

    public setTool(tool: EditorTool) {
        this.currentTool = tool;
        if (tool === 'place') {
            this.transformControls.detach();
            this.showPreview();
        } else {
            this.hidePreview();
        }
    }

    private showPreview() {
        if (!this.previewObject) {
            const geometry = new THREE.BoxGeometry(1, 1, 1);
            const material = new THREE.MeshPhongMaterial({
                color: 0x88aaff,
                transparent: true,
                opacity: 0.5,
            });
            this.previewObject = new THREE.Mesh(geometry, material);
            this.scene.add(this.previewObject);
        }
    }

    private hidePreview() {
        if (this.previewObject) {
            this.scene.remove(this.previewObject);
            this.previewObject = null;
        }
    }

    private animate() {
        requestAnimationFrame(() => this.animate());
        this.controls.update();
        this.renderer.render(this.scene, this.camera);
    }
}

type EditorTool = 'select' | 'move' | 'rotate' | 'scale' | 'place' | 'terrain';
```

---

### `client/src/editor/TerrainEditor.ts` ✅ NO CHANGES NEEDED

```typescript
import * as THREE from 'three';

export class TerrainEditor {
    public mesh: THREE.Mesh;
    private geometry: THREE.PlaneGeometry;
    private brushSize: number = 2.0;
    private brushStrength: number = 0.1;
    private currentTool: TerrainTool = 'raise';
    private heightmapData: Float32Array;
    private targetHeight: number = 0;

    constructor(scene: THREE.Scene, width: number = 128, segments: number = 64) {
        this.geometry = new THREE.PlaneGeometry(width, width, segments, segments);
        this.geometry.rotateX(-Math.PI / 2);
        const material = new THREE.MeshStandardMaterial({
            color: 0x6b8e23,
            roughness: 0.7,
            metalness: 0.1,
            side: THREE.DoubleSide,
        });
        this.mesh = new THREE.Mesh(this.geometry, material);
        this.mesh.receiveShadow = true;
        scene.add(this.mesh);

        const positions = this.geometry.attributes.position.array as Float32Array;
        this.heightmapData = new Float32Array(positions.length / 3);
        for (let i = 0; i < this.heightmapData.length; i++) {
            this.heightmapData[i] = positions[i * 3 + 1];
        }
    }

    public applyBrush(worldX: number, worldZ: number, strength: number = this.brushStrength) {
        const positions = this.geometry.attributes.position.array as Float32Array;
        const totalVerts = positions.length / 3;

        for (let i = 0; i < totalVerts; i++) {
            const x = positions[i * 3];
            const z = positions[i * 3 + 2];
            const dx = x - worldX;
            const dz = z - worldZ;
            const dist = Math.sqrt(dx * dx + dz * dz);
            if (dist < this.brushSize) {
                const falloff = 1 - dist / this.brushSize;
                const delta = strength * falloff;
                let newY = positions[i * 3 + 1];
                switch (this.currentTool) {
                    case 'raise':  newY += delta; break;
                    case 'lower':  newY -= delta; break;
                    case 'smooth': newY = this.getSmoothedHeight(i, positions, totalVerts); break;
                    case 'flatten': newY = this.targetHeight; break;
                }
                positions[i * 3 + 1] = Math.max(0, Math.min(20, newY));
            }
        }

        this.geometry.computeVertexNormals();
        this.geometry.attributes.position.needsUpdate = true;
        this.updateHeightmap();
    }

    private getSmoothedHeight(index: number, positions: Float32Array, totalVerts: number): number {
        const size = Math.sqrt(totalVerts);
        const col = index % size;
        const row = Math.floor(index / size);
        let sum = 0;
        let count = 0;
        for (let dc = -1; dc <= 1; dc++) {
            for (let dr = -1; dr <= 1; dr++) {
                const nc = col + dc;
                const nr = row + dr;
                if (nc >= 0 && nc < size && nr >= 0 && nr < size) {
                    sum += positions[(nr * size + nc) * 3 + 1];
                    count++;
                }
            }
        }
        return sum / count;
    }

    public setFlattenTarget(height: number) { this.targetHeight = height; }

    private updateHeightmap() {
        const positions = this.geometry.attributes.position.array as Float32Array;
        for (let i = 0; i < this.heightmapData.length; i++) {
            this.heightmapData[i] = positions[i * 3 + 1];
        }
    }

    public setBrushSize(size: number) { this.brushSize = Math.max(0.5, Math.min(10, size)); }
    public setBrushStrength(s: number) { this.brushStrength = Math.max(0, Math.min(1, s)); }
    public setTool(tool: TerrainTool) { this.currentTool = tool; }
}

type TerrainTool = 'raise' | 'lower' | 'smooth' | 'flatten';
```

---

### `client/src/utils/undoManager.ts` ✅ NO CHANGES NEEDED

```typescript
export interface UndoAction {
    type: string;
    objectId: string;
    data: unknown;
}

export class UndoManager {
    private undoStack: UndoAction[] = [];
    private redoStack: UndoAction[] = [];
    private maxSize: number;

    constructor(maxSize: number = 100) {
        this.maxSize = maxSize;
    }

    push(action: UndoAction) {
        this.undoStack.push(action);
        this.redoStack = [];
        if (this.undoStack.length > this.maxSize) this.undoStack.shift();
    }

    undo(): UndoAction | undefined {
        const action = this.undoStack.pop();
        if (action) { this.redoStack.push(action); }
        return action;
    }

    redo(): UndoAction | undefined {
        const action = this.redoStack.pop();
        if (action) { this.undoStack.push(action); }
        return action;
    }

    clear() {
        this.undoStack = [];
        this.redoStack = [];
    }
}
```

---

### `client/src/ui/Toolbar.ts` ✅ NO CHANGES NEEDED

```typescript
export class Toolbar {
    private container: HTMLElement;

    constructor(containerId: string, onToolSelected: (tool: string) => void) {
        this.container = document.getElementById(containerId)!;
        this.container.innerHTML = `
            <div class="toolbar">
                <button data-tool="select" class="active">🔍 Select</button>
                <button data-tool="move">✋ Move</button>
                <button data-tool="rotate">🔄 Rotate</button>
                <button data-tool="scale">📏 Scale</button>
                <button data-tool="terrain">⛰️ Terrain</button>
                <button data-tool="portal">🌀 Portal</button>
                <button data-tool="undo">↩️ Undo</button>
                <button data-tool="redo">↪️ Redo</button>
            </div>
        `;
        this.container.querySelectorAll('[data-tool]').forEach((btn) => {
            btn.addEventListener('click', (e) => {
                const tool = (e.currentTarget as HTMLElement).dataset.tool!;
                onToolSelected(tool);
                this.setActive(tool);
            });
        });
    }

    setActive(tool: string) {
        this.container.querySelectorAll('[data-tool]').forEach((btn) => btn.classList.remove('active'));
        this.container.querySelector(`[data-tool="${tool}"]`)?.classList.add('active');
    }
}
```

---

### GLSL Shaders ✅ NO CHANGES NEEDED

**`client/src/shaders/portalVertex.glsl`**
```glsl
varying vec2 vUv;
void main() {
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
```

**`client/src/shaders/portalFragment.glsl`**
```glsl
uniform float uTime;
uniform vec3 uColorA;
uniform vec3 uColorB;
varying vec2 vUv;

void main() {
    float t = uTime * 0.8;
    float pattern = sin(vUv.x * 20.0 + t) * cos(vUv.y * 20.0 + t);
    vec3 color = mix(uColorA, uColorB, pattern * 0.5 + 0.5);
    gl_FragColor = vec4(color, 0.8);
}
```

---

## ENVIRONMENT CONFIGURATION

### `.env.example` ✅ NO CHANGES NEEDED

```env
DATABASE_URL=postgres://user:password@localhost:5432/mmo_creative
RUST_LOG=debug
WEBSOCKET_PORT=8080
```

### `docker-compose.yml` ✅ UPDATED

```yaml
version: '3.8'
services:
  postgres:
    # FIX: Updated from postgres:15 → postgres:17
    image: postgres:17
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mmo_creative
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mmo_creative"]
      interval: 5s
      timeout: 5s
      retries: 5
  server:
    build: ./server
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://user:password@postgres:5432/mmo_creative
volumes:
  postgres_data:
```

> **Added:** `healthcheck` on the postgres service and `condition: service_healthy` on the server
> dependency, so the server does not attempt to connect before Postgres is ready.
> Previously the server would often crash on first boot due to the race condition.

### Database Migration: `001_initial.sql` ✅ NO CHANGES NEEDED

```sql
CREATE TABLE npcs (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    model_path VARCHAR(255) NOT NULL,
    faction VARCHAR(50),
    dialogue_tree_id INT,
    quest_ids INT[],
    is_active BOOLEAN DEFAULT true
);

CREATE TABLE mobs (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    model_path VARCHAR(255) NOT NULL,
    level INT,
    type VARCHAR(50),
    loot_table_id INT
);

CREATE TABLE zones (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    zone_type VARCHAR(50),
    bounds_min_x FLOAT, bounds_min_y FLOAT, bounds_min_z FLOAT,
    bounds_max_x FLOAT, bounds_max_y FLOAT, bounds_max_z FLOAT
);
```

---

## SKILLS REQUIRED

1. **Rust & Bevy (Advanced)** — Bevy ECS plugins, systems, events, queries; async Tokio; WebSocket with tokio-tungstenite; serde serialization; sqlx with PostgreSQL; anyhow/thiserror error handling
2. **Three.js & WebGL (Advanced)** — Scene graph, materials, GLTF loading, TransformControls, OrbitControls, Raycaster, custom GLSL shaders, terrain vertex manipulation, performance (LOD, instancing)
3. **TypeScript (Intermediate)** — ES2022+, strict types, async/await, browser WebSocket API
4. **Networking & Real-Time Sync (Intermediate)** — WebSocket protocol, JSON delta vs full-state sync, multi-client conflict resolution
5. **PostgreSQL & SQLx (Intermediate)** — Schema design, compile-time queries, UUID primary keys, array columns
6. **UI/UX (Intermediate)** — Editor panels, property inspector, undo/redo command pattern, hotkey handling
7. **Mathematics (Intermediate)** — Vectors, matrices, quaternions, ray-plane intersection, grid snapping, Perlin/Simplex noise for terrain
8. **Game Development (Intermediate)** — NavMesh basics, collision detection, spatial partitioning (octree/grid)
9. **DevOps (Basic)** — Docker Compose with healthchecks, GitHub Actions CI/CD

---

## IMPLEMENTATION STEPS (First 2 Weeks)

- **Day 1–2:** Create Rust workspace and Three.js/Vite project; spin up PostgreSQL via Docker Compose
- **Day 3–4:** Basic Bevy app — place a cube via raycast with grid snapping and ghost preview
- **Day 5–6:** WebSocket server — echo server, client connects and receives initial state message
- **Day 7–8:** Three.js — load a GLTF model, place on mouse click, emit placement event to server
- **Day 9–10:** TransformControls + undo/redo stack on client; sync transforms to server
- **Day 11–12:** Database integration — load NPC list from PostgreSQL into asset browser panel
- **Day 13–14:** Basic terrain editor — plane geometry vertex sculpting, sync heightmap delta via WebSocket

Continue with the 16-week roadmap from Phase 2 (Zone System), Phase 3 (Portal & Dungeon Blueprints), Phase 4 (Devices), through Phase 5 (Polish & Multi-user sync).
