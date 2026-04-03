# データ指向設計 (DoD) — SoA vs AoS

**データ指向設計（Data-Oriented Design, DoD）**は、CPU キャッシュの効率を最大化するためにデータのメモリレイアウトを最適化する設計手法です。
オブジェクト指向設計がコードの構造（振る舞い）を中心に考えるのに対し、DoD はデータの並び方と CPU の動作原理を中心に考えます。

---

## AoS と SoA の比較

### AoS（Array of Structs）— 構造体の配列

```
メモリレイアウト（構造体がメモリ上に連続）:

struct Particle { x: f32, y: f32, vx: f32, vy: f32, lifetime: f32, active: bool }

[Particle0                        ][Particle1                        ] ...
 x0  y0  vx0 vy0 life0 act0 pad   x1  y1  vx1 vy1 life1 act1 pad
[f32][f32][f32][f32][f32 ][bool][  ][f32][f32][f32][f32][f32 ][bool][  ]
  0    4    8   12   16   20  21..24 25 ...

「x 座標だけ更新」する処理でのキャッシュライン利用率:
キャッシュライン（64 bytes = 16 × f32）に必要なデータ（x0, x1, ...）は 4 bytes ごとに
他フィールドを挟んで散在 → キャッシュライン 1 本あたり x 値は 2〜3 個だけ
```

### SoA（Struct of Arrays）— 配列の構造体

```
メモリレイアウト（同種フィールドが連続）:

struct Particles {
    x:        Vec<f32>,  // [x0, x1, x2, x3, x4, x5, x6, x7, x8, ...]
    y:        Vec<f32>,  // [y0, y1, y2, y3, y4, y5, y6, y7, y8, ...]
    vx:       Vec<f32>,  // ...
    vy:       Vec<f32>,
    lifetime: Vec<f32>,
    active:   Vec<bool>,
}

「x 座標だけ更新」する処理:
キャッシュライン 1 本（64 bytes）に x 値が 16 個連続 → 利用率 100%
    x0   x1   x2   x3   x4   x5   x6   x7   x8   x9   x10  x11  x12  x13  x14  x15
  [f32][f32][f32][f32][f32][f32][f32][f32][f32][f32][f32][f32][f32][f32][f32][f32]
  |<-------------- キャッシュライン 1 本 (64 bytes) ------------->|
```

---

## キャッシュ効率の違いをベンチマークで示す

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
```

```rust
// src/particle_aos.rs
#[derive(Clone)]
pub struct ParticleAoS {
    pub x: f32,
    pub y: f32,
    pub vx: f32,
    pub vy: f32,
    pub lifetime: f32,
    pub active: bool,
    _padding: [u8; 3], // アライメント調整
}

pub struct ParticlesAoS {
    pub data: Vec<ParticleAoS>,
}

impl ParticlesAoS {
    pub fn new(n: usize) -> Self {
        Self {
            data: (0..n).map(|i| ParticleAoS {
                x: i as f32 * 0.1,
                y: i as f32 * 0.2,
                vx: 1.0,
                vy: -0.5,
                lifetime: 10.0,
                active: true,
                _padding: [0; 3],
            }).collect(),
        }
    }

    /// 速度を位置に積分（x, y のみ更新）
    pub fn integrate(&mut self, dt: f32) {
        for p in &mut self.data {
            if p.active {
                p.x += p.vx * dt;
                p.y += p.vy * dt;
                p.lifetime -= dt;
                p.active = p.lifetime > 0.0;
            }
        }
    }
}
```

```rust
// src/particle_soa.rs
pub struct ParticlesSoA {
    pub x:        Vec<f32>,
    pub y:        Vec<f32>,
    pub vx:       Vec<f32>,
    pub vy:       Vec<f32>,
    pub lifetime: Vec<f32>,
    pub active:   Vec<bool>,
    pub len:      usize,
}

impl ParticlesSoA {
    pub fn new(n: usize) -> Self {
        Self {
            x:        (0..n).map(|i| i as f32 * 0.1).collect(),
            y:        (0..n).map(|i| i as f32 * 0.2).collect(),
            vx:       vec![1.0; n],
            vy:       vec![-0.5; n],
            lifetime: vec![10.0; n],
            active:   vec![true; n],
            len:      n,
        }
    }

    /// SIMD フレンドリーな積分（連続メモリアクセス）
    pub fn integrate(&mut self, dt: f32) {
        for i in 0..self.len {
            if self.active[i] {
                self.x[i] += self.vx[i] * dt;
                self.y[i] += self.vy[i] * dt;
                self.lifetime[i] -= dt;
                self.active[i] = self.lifetime[i] > 0.0;
            }
        }
    }

    /// active フラグのみ走査（他フィールドに触れない）
    pub fn count_active(&self) -> usize {
        self.active.iter().filter(|&&a| a).count()
    }
}
```

```rust
// benches/soa_vs_aos.rs
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_aos(c: &mut Criterion) {
    c.bench_function("AoS integrate 1M", |b| {
        let mut particles = ParticlesAoS::new(1_000_000);
        b.iter(|| particles.integrate(0.016));
    });
}

fn bench_soa(c: &mut Criterion) {
    c.bench_function("SoA integrate 1M", |b| {
        let mut particles = ParticlesSoA::new(1_000_000);
        b.iter(|| particles.integrate(0.016));
    });
}

criterion_group!(benches, bench_aos, bench_soa);
criterion_main!(benches);
```

```bash
cargo bench
```

```
典型的な結果（1,000,000 パーティクル, 参考値）:

AoS integrate 1M    time: [8.45 ms  8.52 ms  8.60 ms]
SoA integrate 1M    time: [2.31 ms  2.34 ms  2.38 ms]  ← 約 3.6x 高速

キャッシュミス率:
  AoS: L1 miss ~45%, L2 miss ~20%
  SoA: L1 miss ~5%,  L2 miss ~2%
```

---

## ECS（Entity Component System）パターンの解説

ECS は DoD の考え方をゲームエンジンのアーキテクチャに適用したものです。

```
従来の OOP アプローチ（AoS に対応）:

class GameObject {
    Position position;     ← 全コンポーネントが 1 オブジェクトに集まる
    Velocity velocity;
    Sprite sprite;
    Health health;
    Physics physics;
}
GameObject enemies[1000]; ← 各オブジェクトのデータが散在

ECS アプローチ（SoA に対応）:

Entity（= 単なる整数 ID）:
  entity_0, entity_1, entity_2, ...

Component（= 純粋なデータ）:
  positions:  [pos_0,  pos_1,  pos_2,  ...]  ← Position コンポーネント配列
  velocities: [vel_0,  vel_1,  vel_2,  ...]  ← Velocity コンポーネント配列
  healths:    [hp_0,   hp_1,   hp_2,   ...]  ← Health コンポーネント配列

System（= データを処理する関数）:
  MovementSystem: positions と velocities のみ走査（健康データに触れない）
  → 必要なデータだけキャッシュに載る
```

```rust
use std::collections::HashMap;

type EntityId = u32;

#[derive(Clone, Debug)]
struct Position { x: f32, y: f32 }

#[derive(Clone, Debug)]
struct Velocity { vx: f32, vy: f32 }

#[derive(Clone, Debug)]
struct Health { hp: f32 }

/// シンプルな ECS ワールド
struct World {
    next_id: EntityId,
    positions:  HashMap<EntityId, Position>,
    velocities: HashMap<EntityId, Velocity>,
    healths:    HashMap<EntityId, Health>,
}

impl World {
    fn new() -> Self {
        Self {
            next_id: 0,
            positions: HashMap::new(),
            velocities: HashMap::new(),
            healths: HashMap::new(),
        }
    }

    fn spawn(&mut self) -> EntityId {
        let id = self.next_id;
        self.next_id += 1;
        id
    }

    fn add_position(&mut self, id: EntityId, pos: Position) {
        self.positions.insert(id, pos);
    }

    fn add_velocity(&mut self, id: EntityId, vel: Velocity) {
        self.velocities.insert(id, vel);
    }

    fn add_health(&mut self, id: EntityId, hp: Health) {
        self.healths.insert(id, hp);
    }
}

/// MovementSystem: Position と Velocity を持つエンティティだけ処理
fn movement_system(world: &mut World, dt: f32) {
    // velocities のキーを先に収集してから更新
    let ids: Vec<EntityId> = world.velocities.keys().copied().collect();
    for id in ids {
        if let (Some(pos), Some(vel)) = (
            world.positions.get_mut(&id),
            world.velocities.get(&id),
        ) {
            pos.x += vel.vx * dt;
            pos.y += vel.vy * dt;
        }
    }
}

/// DamageSystem: Health のみ処理（位置は参照しない）
fn damage_system(world: &mut World, damage: f32) {
    for hp in world.healths.values_mut() {
        hp.hp -= damage;
    }
}

fn main() {
    let mut world = World::new();

    // 移動するエンティティ
    let e1 = world.spawn();
    world.add_position(e1, Position { x: 0.0, y: 0.0 });
    world.add_velocity(e1, Velocity { vx: 1.0, vy: 0.5 });
    world.add_health(e1, Health { hp: 100.0 });

    // 静的エンティティ（速度なし）
    let e2 = world.spawn();
    world.add_position(e2, Position { x: 10.0, y: 5.0 });
    world.add_health(e2, Health { hp: 50.0 });

    for _ in 0..60 {
        movement_system(&mut world, 1.0 / 60.0);
        damage_system(&mut world, 0.1);
    }

    println!("e1 位置: {:?}", world.positions[&e1]);
    println!("e1 HP: {:.1}", world.healths[&e1].hp);
    println!("e2 HP: {:.1}", world.healths[&e2].hp);
}
```

---

## `bevy` ゲームエンジンでの DoD の実例

```toml
[dependencies]
bevy = "0.14"
```

```rust
use bevy::prelude::*;

// コンポーネント = 純粋なデータ構造体
#[derive(Component)]
struct Position { x: f32, y: f32 }

#[derive(Component)]
struct Velocity { vx: f32, vy: f32 }

#[derive(Component)]
struct Health { hp: f32 }

#[derive(Component)]
struct Enemy; // タグコンポーネント（データなし）

// System = 特定のコンポーネントを持つエンティティに対して動作する関数
fn movement_system(
    time: Res<Time>,
    mut query: Query<(&mut Position, &Velocity)>, // Position と Velocity を持つもの全て
) {
    let dt = time.delta_secs();
    for (mut pos, vel) in &mut query {
        pos.x += vel.vx * dt;
        pos.y += vel.vy * dt;
    }
}

fn enemy_damage_system(
    mut query: Query<&mut Health, With<Enemy>>, // Enemy タグを持つもの全て
) {
    for mut health in &mut query {
        health.hp -= 1.0;
    }
}

fn spawn_entities(mut commands: Commands) {
    for i in 0..1000 {
        commands.spawn((
            Position { x: i as f32, y: 0.0 },
            Velocity { vx: 1.0, vy: 0.0 },
            Health { hp: 100.0 },
            Enemy,
        ));
    }
}

fn main() {
    App::new()
        .add_plugins(MinimalPlugins)
        .add_systems(Startup, spawn_entities)
        .add_systems(Update, (movement_system, enemy_damage_system))
        .run();
}
```

Bevy の内部では、同じコンポーネントの組み合わせを持つエンティティを「Archetype」として SoA 形式で格納します。
これにより `movement_system` は `Position` と `Velocity` の配列を連続的にアクセスでき、キャッシュ効率が最大化されます。

---

## SoA への変換ライブラリ（`soa-derive` クレート）

`soa-derive` は derive マクロで AoS 構造体から SoA 版を自動生成します。

```toml
[dependencies]
soa-derive = "0.13"
```

```rust
use soa_derive::StructOfArray;

// StructOfArray derive で SoA 版が自動生成される
#[derive(StructOfArray, Debug, Clone)]
pub struct Particle {
    pub x: f32,
    pub y: f32,
    pub vx: f32,
    pub vy: f32,
    pub lifetime: f32,
}

fn main() {
    // ParticleVec が自動生成される（SoA レイアウト）
    let mut particles = ParticleVec::new();

    for i in 0..100 {
        particles.push(Particle {
            x: i as f32,
            y: 0.0,
            vx: 1.0,
            vy: -0.5,
            lifetime: 10.0,
        });
    }

    // フィールドごとのスライスに直接アクセス可能
    let x_slice: &[f32] = &particles.x; // 連続メモリ
    let y_slice: &[f32] = &particles.y;

    println!("x[0] = {}", x_slice[0]);
    println!("y[0] = {}", y_slice[0]);

    // イテレーション（AoS と同様の書き方も可能）
    for p in particles.iter() {
        let _ = p.x + p.vx;
    }

    // インデックスアクセス
    let p0: ParticleRef = particles[0];
    println!("particle[0]: x={}, lifetime={}", p0.x, p0.lifetime);
}
```

---

## ホットフィールド / コールドフィールドの分離パターン

アクセス頻度の高いフィールド（ホット）と低いフィールド（コールド）を別の配列に分離することで、
ホットデータのキャッシュ効率を上げます。

```
ホット/コールド分離の例（ゲームのエンティティ）:

分離前（AoS）:
struct Entity {
    // ホットフィールド（毎フレーム更新）
    x: f32, y: f32, vx: f32, vy: f32,
    // コールドフィールド（たまに参照）
    name: String,         // 24 bytes
    texture_id: u32,      // 4 bytes
    spawn_time: u64,      // 8 bytes
    kill_count: u32,      // 4 bytes
    description: String,  // 24 bytes
}
// → 構造体が大きく、ホットデータのキャッシュ密度が下がる

分離後（SoA + ホット/コールド分離）:
struct EntityHot {        // 毎フレームアクセス
    x: f32, y: f32, vx: f32, vy: f32
}
struct EntityCold {       // たまにアクセス
    name: String,
    texture_id: u32,
    spawn_time: u64,
    kill_count: u32,
    description: String,
}
→ ホットデータが連続 → キャッシュラインを有効活用
```

```rust
pub struct EntityStorage {
    // ホット: 毎フレーム書き込み・読み込み
    x:  Vec<f32>,
    y:  Vec<f32>,
    vx: Vec<f32>,
    vy: Vec<f32>,

    // コールド: たまに読み込むだけ
    name:       Vec<String>,
    texture_id: Vec<u32>,
    spawn_time: Vec<u64>,
}

impl EntityStorage {
    pub fn new() -> Self {
        Self {
            x: Vec::new(), y: Vec::new(),
            vx: Vec::new(), vy: Vec::new(),
            name: Vec::new(), texture_id: Vec::new(), spawn_time: Vec::new(),
        }
    }

    pub fn spawn(&mut self, x: f32, y: f32, name: String, tex: u32, time: u64) -> usize {
        let id = self.x.len();
        self.x.push(x);    self.y.push(y);
        self.vx.push(0.0); self.vy.push(0.0);
        self.name.push(name);
        self.texture_id.push(tex);
        self.spawn_time.push(time);
        id
    }

    /// ホットパス: コールドデータに触れない
    pub fn integrate_all(&mut self, dt: f32) {
        let n = self.x.len();
        for i in 0..n {
            self.x[i] += self.vx[i] * dt;
            self.y[i] += self.vy[i] * dt;
        }
    }

    /// コールドパス: 必要な時だけ名前を取得
    pub fn get_name(&self, id: usize) -> &str {
        &self.name[id]
    }
}
```

---

## まとめ

| 設計手法 | メモリレイアウト | キャッシュ効率 | コード可読性 | SIMD 適性 |
|---|---|---|---|---|
| AoS（OOP） | 構造体が連続 | 低（フィールドが散在） | 高 | 低 |
| SoA（DoD） | 同種フィールドが連続 | 高 | 中 | 高 |
| ECS（Bevy） | Archetype 単位の SoA | 高 | 高（フレームワーク） | 高 |
| ホット/コールド分離 | 2 種の SoA | 最高（ホット部分） | 低 | 高 |

キャッシュ効率の最適化は、アルゴリズムの改善と同等かそれ以上の効果をもたらすことがあります。
10 倍以上の性能差は珍しくありません。
