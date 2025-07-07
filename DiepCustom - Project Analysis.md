# DiepCustom - Complete Project Analysis and Reimplementation Guide

## Project Overview
**DiepCustom** is an open-source TypeScript/Node.js server that implements a diep.io-compatible private server. It's a real-time multiplayer tank game with WebSocket communication, physics simulation, and multiple game modes.

### Core Technology Stack
- **Backend**: Node.js + TypeScript
- **WebSocket Library**: uWebSockets.js (high-performance WebSocket implementation)
- **Physics**: Custom collision detection system (QuadTree OR Spatial Hashing)
- **Client Communication**: Custom binary protocol with Reader/Writer classes
- **Game Loop**: 40ms tick interval (25 TPS)

### Project Structure Overview
```
/src/
├── index.ts              # Main server entry point
├── config.ts             # Server configuration
├── Game.ts               # Core game server class
├── Client.ts             # Client connection handler
├── util.ts               # Utility functions
├── Native/               # Core engine systems
├── Entity/               # All game entities
├── Physics/              # Physics and collision
├── Coder/                # Binary protocol encoding/decoding
├── Const/                # Constants and definitions
└── Gamemodes/            # Different arena implementations
```

## 1. Server Architecture

### Main Server Entry Point (`index.ts`)
- **WebSocket Server Setup**: Uses uWebSockets.js with compression and auto-pings
- **Connection Management**: Tracks client connections per IP, implements rate limiting
- **Game Server Instances**: Creates multiple game servers (FFA, Sandbox, etc.)
- **REST API**: Optional API endpoints for tank definitions, server info, commands
- **Static File Serving**: Serves client files (HTML, JS) when enabled

### Configuration System (`config.ts`)
```typescript
// Key configuration options:
- serverPort: 8080 (default)
- mspt: 40 (milliseconds per tick)
- tps: 25 (ticks per second)
- connectionsPerIp: Infinity
- maxPlayerLevel: 45
- buildHash: Game version identifier
- magicNum: XOR encryption for tank/stat data
- spatialHashingCellSize: Physics optimization
```

### Core Game Server Class (`Game.ts`)
```typescript
class GameServer {
    gamemode: DiepGamemodeID
    clients: Set<Client>
    entities: EntityManager
    arena: ArenaEntity
    tick: number
    
    // Main game loop
    private tickLoop() {
        // 1. Process client inputs first (lower latency)
        for (client of clients) client.tick(tick)
        
        // 2. Tick all entities
        entities.tick(tick)
    }
}
```

**Supported Game Modes**:
- `ffa`: Free-for-all
- `sandbox`: Creative mode with cheats
- `teams`: 2-team mode
- `4teams`: 4-team mode  
- `dom`: Domination (capture points)
- `mot`: Mothership protection
- `maze`: Maze navigation
- `tag`: Tag/infection mode

## 2. Entity System Architecture

### Core Entity Framework (`Native/Entity.ts`)
```typescript
class Entity {
    id: number              // Unique entity identifier
    hash: number            // Version hash (0 when deleted)
    entityState: number     // Update flags
    game: GameServer        // Reference to game instance
    
    // Field Groups (optional, added as needed):
    positionData: PositionGroup     // x, y, angle
    physicsData: PhysicsGroup       // size, speed, collision
    styleData: StyleGroup           // color, opacity, flags
    healthData: HealthGroup         // health, max health
    relationsData: RelationsGroup   // team, owner references
    // ... and more specialized groups
}
```

### Entity Manager (`Native/Manager.ts`)
- **Entity Storage**: Array-based storage with hash table for versions
- **Collision Management**: QuadTree OR Spatial Hashing for physics
- **Tick Processing**: Handles entity updates in specific order:
  1. Reset collision manager
  2. Insert physical entities into collision system
  3. Apply physics to all entities
  4. Tick all non-camera entities
  5. Tick AI entities
  6. Tick camera entities
  7. Clear update flags

### Entity Hierarchy
```
Entity (base)
├── ObjectEntity                    # Physical entities
│   ├── LivingEntity               # Entities with health/damage
│   │   ├── TankBody               # Player tanks
│   │   ├── AbstractShape          # Polygons (Square, Triangle, etc.)
│   │   └── AbstractBoss           # Boss entities
│   ├── Projectiles                # All fired projectiles
│   │   ├── Bullet                 # Basic bullets
│   │   ├── Drone                  # Controllable drones
│   │   ├── Trap                   # Stationary traps
│   │   ├── Minion                 # AI-controlled minions
│   │   └── Skimmer/Rocket/etc.    # Special projectiles
│   ├── Barrel                     # Tank weapon barrels
│   └── Misc Entities              # Walls, bases, dominators
└── CameraEntity                   # Client viewport/stats
```

## 3. Physics System

### Collision Detection Options
**Two collision detection systems available:**

#### QuadTree (`Physics/QuadTree.ts`)
- Hierarchical spatial partitioning
- Good for sparse entity distributions  
- Recursively subdivides space into quadrants
- Each node stores up to 5 entities before splitting

#### Spatial Hashing (`Physics/SpatialHashing.ts`)  
- Grid-based collision detection
- Better for dense entity distributions
- Configurable cell size (default: 7x7)
- Uses bit manipulation for efficient hash keys

### Physics Implementation
```typescript
// Vector system with common operations
class Vector {
    x: number, y: number
    get magnitude(): number
    get angle(): number
    add(vector), distanceToSQ(vector), etc.
}

// Velocity tracking with position history
class Velocity extends Vector {
    position: Vector
    previousPosition: Vector
    updateVelocity()  // calculates velocity from position delta
}
```

### Collision Types & Flags
- **noOwnTeamCollision**: Projectiles don't hit teammates
- **canEscapeArena**: Bullets can leave arena bounds
- **onlySameOwnerCollision**: Necromancer squares
- **isBase**: Immovable objects (bases, walls)
- **showsOnMap**: Visible on minimap


## 4. Tank & Weapon System

### Tank Definition Structure (`Const/TankDefinitions.ts`)
```typescript
interface TankDefinition {
    id: Tank | DevTank                    // Unique identifier
    name: string                          // Display name
    levelRequirement: number              // Required level to upgrade
    upgrades: (Tank | DevTank)[]         // Available upgrade paths
    
    // Visual properties
    sides: number                         // Body shape (1=circle, 2=rectangle, etc.)
    borderWidth: number                   // Border thickness
    widthRatio?: number                   // For rectangular tanks
    
    // Gameplay properties  
    speed: number                         // Movement speed
    maxHealth: number                     // Base health
    absorbtionFactor: number             // Knockback resistance
    fieldFactor: number                  // Camera zoom/FOV
    
    // Abilities
    flags: {
        invisibility: boolean             // Can go invisible
        zoomAbility: boolean             // Can zoom like Predator
        canClaimSquares: boolean         // Necromancer ability
    }
    
    // Weapons & addons
    barrels: BarrelDefinition[]          // Tank's weapons
    preAddon: addonId | null            // Before barrels (dom base)
    postAddon: addonId | null           // After barrels (spikes, auto-turrets)
    
    // Stats system
    stats: StatDefinition[]              // Available upgrades
}
```

### Barrel & Projectile System
```typescript
interface BarrelDefinition {
    // Positioning
    angle: number                        // Barrel angle relative to tank
    offset: number                       // Side offset (Twin barrels)
    distance?: number                    // Forward/back distance
    size: number                         // Barrel length
    width: number                        // Barrel width (determines bullet size)
    
    // Firing behavior
    delay: number                        // Firing delay in cycle
    reload: number                       // Reload speed multiplier
    recoil: number                       // Knockback when firing
    
    // Visual  
    isTrapezoid: boolean                // Machine Gun barrel shape
    trapezoidDirection: number          // Trapezoid orientation
    color?: Color                       // Custom barrel color
    addon: barrelAddonId | null        // Barrel addons (trap launcher)
    
    // Projectile definition
    bullet: BulletDefinition
}

interface BulletDefinition {
    type: "bullet" | "drone" | "trap" | "necrodrone" | "minion" | "skimmer" | "rocket" | "swarm" | "flame"
    sizeRatio: number                   // Size relative to barrel
    health: number                      // Bullet health
    damage: number                      // Damage multiplier
    speed: number                       // Speed multiplier
    scatterRate: number                 // Spread/accuracy
    lifeLength: number                  // Lifetime in ticks
    absorbtionFactor: number           // Knockback factor
    sides?: number                      // Override bullet shape
    color?: Color                       // Custom bullet color
}
```

## 5. Client Communication System

### Binary Protocol (`Coder/`)
**Custom binary encoding for efficient networking:**

#### Writer Class (`Coder/Writer.ts`)
```typescript
class Writer {
    // Basic data types
    u8(val: number)           // Unsigned 8-bit integer
    u16(val: number)          // Unsigned 16-bit integer  
    u32(val: number)          // Unsigned 32-bit integer
    float(val: number)        // 32-bit float
    
    // Variable-length encoding (space efficient)
    vu(val: number)           // Variable-length unsigned int
    vi(val: number)           // Variable-length signed int
    vf(val: number)           // Variable-length float
    
    // String & binary data
    stringNT(str: string)     // Null-terminated string
    bytes(buffer: Uint8Array) // Raw bytes
    
    // Game-specific
    entid(entity)             // Entity ID + hash
    degrees(degrees: number)  // Angle encoding
}
```

#### Reader Class (`Coder/Reader.ts`)
- **Matching decode methods** for all Writer operations
- **Efficient parsing** of binary WebSocket messages
- **Position tracking** with internal cursor

### Client Management (`Client.ts`)
```typescript
class Client {
    accessLevel: AccessLevel           // Permission level
    camera: ClientCamera              // Viewport and stats
    inputs: ClientInputs              // Cached user inputs
    game: GameServer                  // Current game instance
    
    // Input handling
    onMessage(buffer: ArrayBuffer)    // Parse incoming packets
    tick(tick: number)               // Process inputs each tick
    
    // Communication
    write(): WSWriterStream          // Get writer for this client
    send(data: Uint8Array)          // Send raw data
    notify(text, color, time)       // Send notification
}
```

### Message Types (ClientBound/ServerBound)
- **ServerBound**: Input, StatUpgrade, TankUpgrade, Spawn, Command
- **ClientBound**: Update, PlayerCount, Accept, ServerInfo, Notification, etc.

## 6. Game Mode System (`Gamemodes/`)

### Arena Architecture
```typescript
class ArenaEntity extends Entity {
    // Core arena properties
    width: number, height: number        // Arena boundaries
    shapes: ShapeManager                 // Manages polygon spawns
    allowBoss: boolean                   // Enable boss spawns
    
    // Arena lifecycle
    spawnPlayer(tank: TankBody, client: Client)  // Handle player spawning
    updateBounds(width: number, height: number)  // Resize arena
    tick(tick: number)                          // Arena updates
    close()                                     // End game instance
}
```

### Game Mode Implementations

#### **FFA Arena** (`FFA.ts`)
- **Simplest mode**: Extends ArenaEntity with no modifications
- **Players spawn randomly** throughout the arena
- **No teams or special mechanics**

#### **Sandbox Arena** (`Sandbox.ts`) 
- **Creative mode** with cheats enabled
- **Dynamic arena size** based on player count: `25 * sqrt(playerCount) * 100`
- **Reduced shape spawns**: ~12.5 * player count

#### **Team-Based Modes**
**Teams2Arena** (`Team2.ts`):
- **Two team bases** on opposite sides of arena
- **Team colors**: Blue vs Red
- **Base guards** with defensive drones
- **Players spawn in their team's base area**

**Teams4Arena** (`Team4.ts`):
- **Four corner bases**: Blue, Red, Green, Purple
- **Square arena layout** with bases in corners
- **Similar base guard system**

#### **Advanced Modes**

**Domination** (`Domination.ts`):
- **Capture point system** with 4 dominators
- **Team bases** + neutral dominator bases
- **Score tracking** based on controlled dominators

**Mothership** (`Mothership.ts`):
- **Protect the mothership** gameplay
- **Large mothership entities** for each team
- **Win condition**: Destroy enemy motherships

**Tag Arena** (`Tag.ts`):
- **Infection/zombie mode** mechanics
- **Players change teams** when killed
- **Shrinking arena** over time (100 units per 15 seconds)
- **Win condition**: All players on one team

**Maze Arena** (`Maze.ts`):
- **Procedurally generated maze** walls
- **Custom maze generation algorithm**
- **Wall-based navigation challenges**

## 7. AI & Drone System

### AI Framework (`Entity/AI.ts`)
```typescript
class AI {
    owner: ObjectEntity               // Entity this AI controls
    state: AIState                   // Current behavior state
    target: ObjectEntity | null      // Current target
    inputs: Inputs                   // Control inputs (movement, shooting)
    
    // Behavior properties
    viewRange: number                // Detection radius
    aimSpeed: number                // Turning speed
    movementSpeed: number           // Movement speed
    
    // State machine
    tick(tick: number) {
        // 1. Find targets within view range
        // 2. Update state based on targets
        // 3. Calculate movement and aiming
        // 4. Update inputs for owner entity
    }
}

enum AIState {
    idle = 0,
    lookingForTarget = 1,
    hasTarget = 2,
    waitingAfterKill = 3,
    smoothTurning = 4
}
```

### Drone System
**Controllable projectiles with AI**:
- **Drone**: Basic controllable projectile
- **Minion**: Drone with its own weapons
- **NecromancerSquare**: Converted from killed shapes
- **Swarm**: Short-lived seeking projectiles

### Shape Management (`Entity/Shape/Manager.ts`)
```typescript
class ShapeManager {
    protected wantedShapes: number    // Desired shape count
    protected shapes: AbstractShape[] // Current shapes
    
    spawnShape(): AbstractShape      // Create new shape
    tick()                          // Maintain shape count
}
```

**Shape Types**: Square, Triangle, Pentagon, Crasher (moves and attacks)

## 8. Implementation Roadmap

### Phase 1: Core Foundation
1. **Basic Node.js + TypeScript setup**
   - Package.json with dependencies (uWebSockets.js, typescript, @types/node)
   - TypeScript configuration
   - Basic server with WebSocket handling

2. **Entity System**
   - Base Entity class with field groups
   - EntityManager for entity lifecycle
   - Basic ObjectEntity for physical objects

3. **Binary Protocol**
   - Writer/Reader classes for efficient networking
   - Message type enums (ClientBound/ServerBound)

### Phase 2: Game Mechanics
4. **Physics System**
   - Vector and Velocity classes
   - Choose collision detection (QuadTree OR Spatial Hashing)
   - Basic collision resolution

5. **Tank System**
   - TankBody class with movement
   - Barrel system for weapons
   - Basic projectile system (bullets)

6. **Client Management**
   - Client class with input handling
   - Camera system for viewport
   - Basic FFA arena

### Phase 3: Advanced Features  
7. **Tank Definitions**
   - JSON-based tank configuration
   - Upgrade system with level requirements
   - Multiple projectile types (drones, traps, etc.)

8. **Game Modes**
   - Team-based arenas
   - Advanced modes (Domination, Mothership, etc.)
   - Win conditions and scoring

9. **AI & Shapes**
   - AI system for drones and bosses
   - Shape spawning and management
   - Boss entities

### Phase 4: Polish & Features
10. **Advanced Systems**
    - Commands system for admin control
    - Statistics and leaderboards  
    - Anti-cheat measures
    - Performance optimizations

## Key Implementation Notes

### Critical Dependencies
```json
{
  "dependencies": {
    "uWebSockets.js": "github:uNetworking/uWebSockets.js#v20.52.0",
    "tweetnacl": "^1.0.3"  // For cryptographic functions
  }
}
```

### Performance Considerations
- **25 TPS game loop** (40ms intervals)
- **Binary protocol** essential for network efficiency
- **Collision detection choice** affects CPU usage significantly
- **Entity pooling** for frequently created/destroyed objects
- **Input prediction** for responsive controls

### Architecture Patterns
- **Entity-Component-System (ECS)** with field groups
- **Observer pattern** for entity updates
- **State machine** for AI behaviors
- **Factory pattern** for entity creation
- **Command pattern** for user actions

### Testing Strategy
- Start with **basic FFA mode**
- Test with **multiple clients** early
- Profile **physics performance** with many entities  
- Validate **network protocol** compatibility
- Stress test **collision detection** systems

