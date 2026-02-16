# RFC-001: ChessPath — Architecture Design for a Modular Chess Learning Platform

**Status:** Draft  
**Authors:** Staff Architecture Team  
**Date:** 2026-02-16  
**Audience:** Engineering, Product, Content  

---

## 0. Executive Summary

This RFC proposes the technical architecture for **ChessPath**, a Duolingo-style chess learning application. The system is designed as a **domain-agnostic learning engine** with chess as a pluggable content layer. Every architectural decision prioritizes extensibility: content creators ship lessons as data, not code. The board engine, evaluation pipeline, and progression system are all module boundaries with published interfaces and no cross-contamination.

The core insight driving this design: **a chess lesson is not fundamentally different from a language lesson**. Both present a stimulus, expect a decision, evaluate correctness, and update a learner model. We build the engine once and let the domain (chess) live entirely in content and evaluation plugins.

---

## 1. High-Level Architecture

### 1.1 System Boundary Diagram

```
┌──────────────────────────────────────────────────────────┐
│                      CLIENT SHELL                        │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌───────────┐  │
│  │ Lesson  │  │  Board   │  │ Puzzle  │  │ Progress  │  │
│  │ Player  │  │ Explorer │  │ Solver  │  │ Dashboard │  │
│  └────┬────┘  └────┬─────┘  └────┬────┘  └─────┬─────┘  │
│       │            │             │              │         │
│  ┌────┴────────────┴─────────────┴──────────────┴─────┐  │
│  │              BOARD ENGINE (client-side)             │  │
│  │   state · validation · annotations · branching     │  │
│  └────────────────────────┬───────────────────────────┘  │
└───────────────────────────┼──────────────────────────────┘
                            │ WebSocket / REST
┌───────────────────────────┼──────────────────────────────┐
│                      API GATEWAY                         │
│  auth · rate limiting · routing · session management     │
└──┬──────────┬──────────┬──────────┬──────────┬───────────┘
   │          │          │          │          │
┌──┴───┐ ┌───┴────┐ ┌───┴────┐ ┌───┴────┐ ┌───┴───────┐
│Content│ │ Eval   │ │Progress│ │Analytics│ │ Real-time │
│Engine │ │ Engine │ │ Engine │ │ Engine  │ │   Sync    │
└──┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬───────┘
   │          │          │          │          │
┌──┴──────────┴──────────┴──────────┴──────────┴───────────┐
│                    DATA LAYER                             │
│  PostgreSQL · Redis · S3 (content) · ClickHouse (events) │
└──────────────────────────────────────────────────────────┘
```

### 1.2 Module Responsibilities

**Client Shell (Frontend)**  
Single-page application hosting four primary views. The shell owns layout, routing, and theming. Each view is a feature module that communicates exclusively through the Board Engine and backend APIs. No view imports from another view — they share state only through the board engine's public interface and a global session store.

**Board Engine (Client-Side)**  
The critical shared module. Maintains board state, validates moves, manages annotation layers, and supports branching move trees. This is a pure logic library with no UI — rendering is the shell's job. It runs entirely client-side for zero-latency interaction. The board engine is the single source of truth for position state across all four modes.

**Content Engine (Backend)**  
Serves lesson definitions, exercise sets, and position data. Content is stored as structured JSON/YAML and versioned. The content engine never evaluates correctness — it only describes what to present and what to expect. It supports content tagging, search, prerequisite graphs, and A/B test variants.

**Evaluation Engine (Backend)**  
Determines whether a user action is correct, partially correct, or incorrect. It is **pluggable**: different evaluators (rules-based, engine-based, pedagogy-based) register themselves and content references them by evaluator ID. The evaluation engine also returns structured feedback payloads, not just pass/fail.

**Progression Engine (Backend)**  
Tracks mastery, schedules review, categorizes mistakes, and determines what content to surface next. It operates on abstract "skill atoms" — it has no concept of what a chess opening is, only that a user demonstrated proficiency (or didn't) on skill `X` at time `T`.

**Analytics Engine (Backend)**  
Ingests events from all modules into a columnar store. Provides aggregations for the progress dashboard, content effectiveness reports, and system health. It is append-only and never blocks user-facing flows.

**Real-time Sync (Backend)**  
WebSocket layer for live board state synchronization. MVP scope is single-user (syncing analysis across tabs/devices). Future scope includes multiplayer, spectating, and collaborative analysis.

### 1.3 Communication Rules

All inter-service communication is via well-defined API contracts (OpenAPI for REST, AsyncAPI for events). No service reads another service's database. The progression engine subscribes to evaluation events — it never calls the evaluation engine directly. Content is fetched, not pushed. The board engine is never called by the backend; it is a client-only library.

---

## 2. Monorepo / Project Structure

```
chessspath/
├── apps/
│   ├── web/                        # SPA shell (Next.js or Vite + React)
│   │   ├── src/
│   │   │   ├── features/
│   │   │   │   ├── lesson-player/  # Guided lesson view
│   │   │   │   ├── board-explorer/ # Free analysis / sandbox view
│   │   │   │   ├── puzzle-solver/  # Exercise solving view
│   │   │   │   └── progress/       # Dashboard view
│   │   │   ├── shell/              # Layout, routing, global providers
│   │   │   └── shared/             # Design system tokens, common hooks
│   │   └── public/
│   │
│   └── api/                        # Backend API gateway (Node/Fastify or Go)
│       ├── src/
│       │   ├── routes/             # HTTP + WS route handlers
│       │   ├── middleware/         # Auth, rate-limit, validation
│       │   └── workers/           # Background job processors
│       └── Dockerfile
│
├── packages/
│   ├── board-engine/               # Pure TS board logic library
│   │   ├── src/
│   │   │   ├── state/             # Position, FEN, PGN
│   │   │   ├── rules/             # Move generation & validation
│   │   │   ├── tree/              # Move tree, branching, undo/redo
│   │   │   ├── annotations/       # Arrows, highlights, text notes
│   │   │   └── modes/             # Mode-specific constraints
│   │   ├── __tests__/
│   │   └── package.json
│   │
│   ├── board-ui/                   # React board rendering component
│   │   ├── src/
│   │   │   ├── Board.tsx          # SVG/Canvas board renderer
│   │   │   ├── Pieces.tsx         # Piece sprite rendering
│   │   │   ├── Overlays.tsx       # Arrows, highlights, drag ghosts
│   │   │   └── themes/            # Board + piece style themes
│   │   └── package.json
│   │
│   ├── content-sdk/                # Content definition types + validators
│   │   ├── src/
│   │   │   ├── schema/            # JSON Schema + TS types for content
│   │   │   ├── validators/        # Runtime content validation
│   │   │   └── loaders/           # Content fetching + caching
│   │   └── package.json
│   │
│   ├── eval-sdk/                   # Evaluation interface + built-in evaluators
│   │   ├── src/
│   │   │   ├── interface.ts       # IEvaluator contract
│   │   │   ├── evaluators/        # Plugin directory
│   │   │   │   ├── exact-move.ts
│   │   │   │   ├── engine-range.ts
│   │   │   │   └── rule-based.ts
│   │   │   └── registry.ts        # Evaluator plugin registry
│   │   └── package.json
│   │
│   ├── progression-sdk/            # Skill tracking types + algorithms
│   │   ├── src/
│   │   │   ├── models/            # SkillAtom, MasteryRecord, etc.
│   │   │   ├── schedulers/        # Spaced repetition, adaptive
│   │   │   └── categorizers/      # Mistake taxonomy framework
│   │   └── package.json
│   │
│   └── shared-types/               # Cross-package type definitions
│       ├── src/
│       │   ├── content.ts         # Lesson, Exercise, Position types
│       │   ├── evaluation.ts      # Result, Feedback types
│       │   ├── progression.ts     # Mastery, Schedule types
│       │   └── events.ts          # Domain event schemas
│       └── package.json
│
├── services/
│   ├── content-engine/             # Content serving microservice
│   │   ├── src/
│   │   │   ├── storage/           # S3 / filesystem content store
│   │   │   ├── indexer/           # Content search & prerequisite graph
│   │   │   └── versioning/        # Content version management
│   │   └── Dockerfile
│   │
│   ├── eval-engine/                # Evaluation microservice
│   │   ├── src/
│   │   │   ├── dispatcher.ts      # Routes to correct evaluator
│   │   │   ├── engine-pool/       # Stockfish process pool manager
│   │   │   └── cache/             # Evaluation result cache
│   │   └── Dockerfile
│   │
│   ├── progression-engine/         # Progression microservice
│   │   ├── src/
│   │   │   ├── tracker.ts         # Mastery state machine
│   │   │   ├── scheduler.ts       # Next-content selection
│   │   │   └── analyzer.ts        # Mistake pattern detection
│   │   └── Dockerfile
│   │
│   └── analytics-engine/           # Event ingestion + aggregation
│       ├── src/
│       └── Dockerfile
│
├── content/                        # PURE DATA — no code here
│   ├── courses/
│   │   ├── opening/
│   │   │   ├── _course.yaml       # Course metadata + skill tree
│   │   │   ├── unit-01/
│   │   │   │   ├── _unit.yaml     # Unit metadata
│   │   │   │   ├── lesson-01.yaml
│   │   │   │   └── lesson-02.yaml
│   │   │   └── unit-02/
│   │   ├── middlegame/
│   │   └── endgame/
│   ├── puzzles/
│   │   └── collections/
│   └── schemas/                    # JSON Schema definitions for content
│       ├── lesson.schema.json
│       ├── exercise.schema.json
│       └── course.schema.json
│
├── tools/
│   ├── content-cli/                # CLI for content authors
│   │   ├── validate.ts            # Lint + validate content files
│   │   ├── preview.ts             # Local content preview server
│   │   └── publish.ts             # Push content to content engine
│   └── seed/                       # Dev seed data generators
│
├── infra/
│   ├── docker-compose.yml
│   ├── k8s/
│   └── terraform/
│
├── turbo.json                      # Turborepo pipeline config
├── package.json
└── tsconfig.base.json
```

### 2.1 Boundary Rules

**Rule 1: `content/` is code-free.**  
Content authors never write TypeScript. They write YAML/JSON validated against schemas in `content/schemas/`. A CI pipeline runs `tools/content-cli/validate.ts` on every content PR. If validation fails, the PR is blocked.

**Rule 2: Feature modules never import each other.**  
`lesson-player/` cannot import from `puzzle-solver/`. If they need shared behavior, it goes into `packages/board-engine/` or `packages/board-ui/`. The dependency graph is strictly: `apps → packages`, `services → packages`, never `packages → apps` or `apps/featureA → apps/featureB`.

**Rule 3: `packages/` are publishable, `apps/` and `services/` are deployable.**  
Every package must be independently testable and versioned. Packages depend only on other packages and `shared-types`, never on apps or services.

**Rule 4: The board engine has zero dependencies on React, the DOM, or any UI framework.**  
`board-engine` is a pure TypeScript library. `board-ui` wraps it for React. This separation allows future native mobile ports (React Native, Flutter) to reuse the engine.

**Rule 5: Services communicate only via API contracts and events.**  
No service imports another service's code. Shared logic goes into an SDK package (`eval-sdk`, `progression-sdk`). Services may share `shared-types` for type definitions.

---

## 3. Board & Move System Design

The board engine is the most complex client-side module. It must support four operating modes through a single state model.

### 3.1 State Representation

```typescript
// packages/board-engine/src/state/types.ts

/** 8x8 mailbox representation. Index 0 = a8, index 63 = h1. */
type Square = number; // 0–63

type PieceType = 'king' | 'queen' | 'rook' | 'bishop' | 'knight' | 'pawn';
type Color = 'white' | 'black';

interface Piece {
  type: PieceType;
  color: Color;
}

interface CastlingRights {
  whiteKingside: boolean;
  whiteQueenside: boolean;
  blackKingside: boolean;
  blackQueenside: boolean;
}

/** Immutable snapshot of a board position. */
interface Position {
  readonly board: ReadonlyArray<Piece | null>;   // length 64
  readonly activeColor: Color;
  readonly castling: Readonly<CastlingRights>;
  readonly enPassantSquare: Square | null;
  readonly halfmoveClock: number;
  readonly fullmoveNumber: number;
}

/** A validated, legal move. */
interface Move {
  readonly from: Square;
  readonly to: Square;
  readonly promotion?: PieceType;
  readonly capturedPiece?: Piece;
  readonly isEnPassant: boolean;
  readonly isCastling: boolean;
  readonly san: string;                          // "Nf3", "O-O", etc.
  readonly resultingPosition: Position;
}
```

### 3.2 Move Validation Interface

```typescript
// packages/board-engine/src/rules/interface.ts

interface IMoveValidator {
  /** Returns all legal moves for the given position. */
  generateLegalMoves(position: Position): Move[];

  /** Validates a specific move attempt. Returns the Move if legal, null otherwise. */
  validateMove(position: Position, from: Square, to: Square, promotion?: PieceType): Move | null;

  /** Checks game-ending conditions. */
  getGameStatus(position: Position): GameStatus;
}

type GameStatus =
  | { state: 'active' }
  | { state: 'checkmate'; winner: Color }
  | { state: 'stalemate' }
  | { state: 'draw'; reason: 'fifty-move' | 'threefold' | 'insufficient-material' };
```

### 3.3 Move Tree & Branching Timelines

This is the core data structure that enables analysis mode, lesson variations, and undo/redo.

```typescript
// packages/board-engine/src/tree/types.ts

/** A node in the move tree. The root node has no move (it's the starting position). */
interface MoveNode {
  readonly id: string;                          // UUID
  readonly move: Move | null;                   // null for root
  readonly position: Position;
  readonly parent: MoveNode | null;
  readonly children: ReadonlyArray<MoveNode>;   // first child = main line
  readonly annotations: ReadonlyArray<Annotation>;
  readonly metadata: NodeMetadata;
}

interface NodeMetadata {
  readonly comment?: string;
  readonly nag?: number[];                      // Numeric Annotation Glyphs (!, ?, !!, etc.)
  readonly evaluation?: EngineEvaluation;
  readonly timestamp?: number;
  readonly authorId?: string;
}

/** The full tree with a cursor pointing to the current position. */
interface MoveTree {
  readonly root: MoveNode;
  readonly cursor: MoveNode;                    // current position in the tree
  readonly mainLine: ReadonlyArray<MoveNode>;   // flattened main variation
}
```

### 3.4 Annotation Layer

```typescript
// packages/board-engine/src/annotations/types.ts

type Annotation =
  | ArrowAnnotation
  | HighlightAnnotation
  | TextAnnotation;

interface ArrowAnnotation {
  type: 'arrow';
  from: Square;
  to: Square;
  color: string;
  opacity?: number;
}

interface HighlightAnnotation {
  type: 'highlight';
  square: Square;
  color: string;
  opacity?: number;
}

interface TextAnnotation {
  type: 'text';
  square: Square;
  text: string;
  style?: 'label' | 'bubble';
}
```

### 3.5 Board Controller Interface

The unified controller that all four views use:

```typescript
// packages/board-engine/src/controller.ts

interface IBoardController {
  // --- State ---
  getPosition(): Position;
  getMoveTree(): MoveTree;
  getFEN(): string;
  getPGN(): string;

  // --- Move Execution ---
  makeMove(from: Square, to: Square, promotion?: PieceType): MoveResult;
  makeMoveFromSAN(san: string): MoveResult;

  // --- Navigation ---
  goToNode(nodeId: string): void;
  goForward(): boolean;                        // follows main line
  goBack(): boolean;
  goToStart(): void;
  goToEnd(): void;

  // --- Branching ---
  addVariation(move: Move): MoveNode;          // branch from current position
  promoteVariation(nodeId: string): void;       // make a variation the main line
  deleteVariation(nodeId: string): void;

  // --- Annotations ---
  addAnnotation(nodeId: string, annotation: Annotation): void;
  removeAnnotation(nodeId: string, annotationId: string): void;
  setComment(nodeId: string, comment: string): void;

  // --- Serialization ---
  loadFromFEN(fen: string): void;
  loadFromPGN(pgn: string): void;
  exportPGN(): string;
  serialize(): SerializedTree;                  // for persistence
  deserialize(data: SerializedTree): void;

  // --- Events ---
  on(event: BoardEvent, handler: BoardEventHandler): Unsubscribe;
}

type MoveResult =
  | { ok: true; node: MoveNode }
  | { ok: false; reason: 'illegal' | 'mode-restricted' | 'game-over' };

type BoardEvent =
  | 'move'
  | 'navigate'
  | 'branch'
  | 'annotate'
  | 'position-change'
  | 'game-status-change';
```

### 3.6 Mode System

Modes constrain what the board controller allows. They don't change the interface — they change what `makeMove` permits.

```typescript
// packages/board-engine/src/modes/types.ts

interface IBoardMode {
  readonly name: string;

  /** Can the user make this move in the current mode state? */
  canMakeMove(move: Move, context: ModeContext): boolean;

  /** Can the user navigate freely? */
  canNavigate(): boolean;

  /** Can the user create branches? */
  canBranch(): boolean;

  /** Called after a move is made. Returns mode-specific side effects. */
  onMoveMade(move: Move, context: ModeContext): ModeEffect[];
}

interface ModeContext {
  tree: MoveTree;
  expectedMoves?: string[];                    // for lesson/puzzle modes
  moveHistory: Move[];
}

type ModeEffect =
  | { type: 'show-feedback'; feedback: Feedback }
  | { type: 'lock-board' }
  | { type: 'unlock-board' }
  | { type: 'auto-respond'; move: string }     // play opponent's move
  | { type: 'emit-event'; event: DomainEvent };
```

**Built-in modes:**

| Mode | Navigation | Branching | Move Restriction |
|---|---|---|---|
| `FreeExploreMode` | Unrestricted | Unrestricted | Legal moves only |
| `LessonMode` | Restricted to lesson line | No | Only expected moves |
| `PuzzleMode` | None | No | Only correct moves |
| `ReplayMode` | Unrestricted | No | None (read-only) |

---

## 4. Content Abstraction Layer

### 4.1 Design Principles

Content is pure data. It describes **what** to present and **what** to evaluate, but never **how** to evaluate it. The `evaluator` field references a registered evaluation plugin by ID. Content authors can add new lessons by dropping YAML files into `content/` — zero code changes required.

### 4.2 Content Hierarchy

```
Course (e.g., "Opening Principles")
  └── Unit (e.g., "Center Control")
       └── Lesson (a sequence of steps)
            └── Step (a single interaction point)
```

### 4.3 Schema: Course Definition

```yaml
# content/courses/opening/_course.yaml
id: "opening"
version: "1.0.0"
title: "Opening Principles"
description: "..."
phase: "opening"                        # top-level bucket for UI grouping
icon: "sword"
color: "#7B61FF"

skillTree:
  - unitId: "center-control"
    prerequisites: []
  - unitId: "piece-development"
    prerequisites: ["center-control"]
  - unitId: "king-safety"
    prerequisites: ["piece-development"]

metadata:
  estimatedHours: 8
  difficulty: "beginner"
  tags: ["fundamentals"]
```

### 4.4 Schema: Lesson Definition

```yaml
# content/courses/opening/unit-01/lesson-01.yaml
id: "center-control-intro"
version: "1.2.0"
unitId: "center-control"
title: "Why the Center Matters"
description: "..."
estimatedMinutes: 5

# Skills this lesson teaches/reinforces (abstract IDs, not chess terms)
skillAtoms:
  - "opening.center.pawn-placement"
  - "opening.center.piece-activity"

prerequisites:
  lessons: []
  skills: []

steps:
  - id: "step-1"
    type: "instruction"
    content:
      text: "The four central squares are the most important on the board."
      position: "rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1"
      annotations:
        - type: "highlight"
          squares: ["d4", "d5", "e4", "e5"]
          color: "#FFD700"
      audio: null                               # future: narration asset ID

  - id: "step-2"
    type: "interactive-move"
    content:
      prompt: "Place a pawn in the center."
      position: "rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1"
      boardMode: "lesson"
    evaluation:
      evaluatorId: "multi-correct-move"         # references a registered evaluator
      params:
        acceptedMoves: ["e2e4", "d2d4"]
        idealMove: "e2e4"
        feedback:
          onCorrect: "Great! You've placed a pawn in the center."
          onAcceptable: "d4 also controls the center — good instinct."
          onIncorrect: "That move doesn't fight for the center. Try again."
    maxAttempts: 3
    hints:
      - "Look at the highlighted squares."
      - "Which pawn can reach the center in one move?"

  - id: "step-3"
    type: "interactive-move"
    content:
      prompt: "Your opponent responds. What's the best continuation?"
      position: "rnbqkbnr/pppp1ppp/8/4p3/4P3/8/PPPP1PPP/RNBQKBNR w KQkq e6 0 2"
      autoRespond:                              # system makes Black's move
        move: "e7e5"
        delay: 800                              # milliseconds
    evaluation:
      evaluatorId: "engine-range"
      params:
        engineDepth: 15
        acceptableRange: 0.5                    # within 0.5 of best eval
        feedback:
          onCorrect: "Excellent development!"
          onIncorrect: "There's a more active square for this piece."
    maxAttempts: 2

  - id: "step-4"
    type: "branch-decision"
    content:
      prompt: "Which approach appeals to you?"
      position: "rnbqkbnr/pppp1ppp/8/4p3/4P3/5N2/PPPP1PPP/RNBQKB1R b KQkq - 1 2"
      options:
        - label: "Aggressive center"
          nextStep: "step-5a"
          skillAtom: "opening.center.pawn-push"
        - label: "Solid development"
          nextStep: "step-5b"
          skillAtom: "opening.center.piece-development"
    evaluation: null                            # no right answer — user choice

  - id: "step-5a"
    type: "interactive-move"
    # ... aggressive branch continues
    nextStep: "step-6"

  - id: "step-5b"
    type: "interactive-move"
    # ... solid branch continues
    nextStep: "step-6"

  - id: "step-6"
    type: "summary"
    content:
      text: "You've learned the basics of center control."
      keyTakeaways:
        - "Central pawns control more squares."
        - "Pieces are more active from central positions."
    evaluation: null

completionCriteria:
  requiredSteps: ["step-1", "step-2", "step-3", "step-4", "step-6"]
  passingScore: 0.6                             # 60% of evaluated steps correct
```

### 4.5 Schema: Exercise / Puzzle Definition

```yaml
# content/puzzles/collections/tactics-01/puzzle-001.yaml
id: "puzzle-001"
version: "1.0.0"
position: "r1bqkb1r/pppp1ppp/2n2n2/4p2Q/2B1P3/8/PPPP1PPP/RNB1K1NR w KQkq - 4 4"
sideToMove: "white"

skillAtoms:
  - "tactics.checkmate.scholars-mate"

evaluation:
  evaluatorId: "exact-sequence"
  params:
    solution:
      - move: "h5f7"
        annotation: "Checkmate!"
    allowTranspositions: false

difficulty:
  rating: 600
  tags: ["one-move", "checkmate"]

metadata:
  source: "classical-collection"
  theme: ["checkmate-pattern"]
```

### 4.6 Content Validation Pipeline

```
content author writes YAML
        │
        ▼
┌─────────────────┐     ┌───────────────────────┐
│  JSON Schema     │────▶│  Structural valid?     │── no ──▶ CI FAIL
│  Validation      │     │  (types, required      │
└─────────────────┘     │  fields, enums)        │
                        └──────────┬────────────┘
                                   │ yes
                        ┌──────────▼────────────┐
                        │  Semantic validation   │── no ──▶ CI FAIL
                        │  (FEN valid? Evaluator │
                        │   ID exists? Moves     │
                        │   legal? Skill IDs in  │
                        │   registry?)           │
                        └──────────┬────────────┘
                                   │ yes
                        ┌──────────▼────────────┐
                        │  Preview server        │
                        │  (local playthrough)   │
                        └──────────┬────────────┘
                                   │ author approves
                        ┌──────────▼────────────┐
                        │  Publish to content    │
                        │  engine (versioned)    │
                        └───────────────────────┘
```

---

## 5. Evaluation Engine

### 5.1 Architecture

The evaluation engine is a service that receives an evaluation request and delegates to the appropriate evaluator plugin. The plugin is determined by the `evaluatorId` field in the content definition.

```
┌──────────────────────────────────────────────────┐
│                EVALUATION SERVICE                 │
│                                                  │
│  ┌────────────┐    ┌──────────────────────────┐  │
│  │  Dispatch   │───▶│  Evaluator Registry       │  │
│  │  Controller │    │                          │  │
│  └────────────┘    │  ┌────────────────────┐  │  │
│                    │  │ exact-move          │  │  │
│                    │  │ multi-correct-move  │  │  │
│                    │  │ exact-sequence      │  │  │
│                    │  │ engine-range        │  │  │
│                    │  │ engine-best         │  │  │
│                    │  │ rule-based          │  │  │
│                    │  │ pedagogy-composite  │  │  │
│                    │  └────────────────────┘  │  │
│                    └──────────────────────────┘  │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │  Stockfish Process Pool                   │    │
│  │  (shared resource, accessed by engine-*   │    │
│  │   evaluators only)                        │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │  Evaluation Cache (Redis)                 │    │
│  │  key: hash(position + evaluatorId + params)│    │
│  └──────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘
```

### 5.2 Evaluator Plugin Interface

```typescript
// packages/eval-sdk/src/interface.ts

interface IEvaluator {
  readonly id: string;
  readonly version: string;

  /**
   * Evaluate a user's move in context.
   * The evaluator receives the position, the move, and author-defined params.
   * It returns a structured result — never a raw boolean.
   */
  evaluate(request: EvaluationRequest): Promise<EvaluationResult>;

  /** Validate that the params in a content definition are well-formed. */
  validateParams(params: unknown): ValidationResult;
}

interface EvaluationRequest {
  position: string;                             // FEN
  userMove: string;                             // UCI format (e.g., "e2e4")
  params: Record<string, unknown>;              // from content definition
  context: EvaluationContext;
}

interface EvaluationContext {
  userId: string;
  attemptNumber: number;
  moveHistory: string[];                        // moves made so far in this exercise
  timeSpentMs: number;
}

interface EvaluationResult {
  verdict: 'correct' | 'acceptable' | 'incorrect';
  score: number;                                // 0.0 – 1.0
  feedback: Feedback;
  metadata: Record<string, unknown>;            // evaluator-specific data
}

interface Feedback {
  message: string;
  annotations?: Annotation[];                   // arrows/highlights to show
  suggestedMove?: string;                       // hint for next attempt
  explanation?: string;                         // detailed explanation
}
```

### 5.3 Built-in Evaluator Types

**`exact-move`** — Checks if the user's move matches a single expected move. Simplest evaluator. Used for scripted lessons where there is exactly one correct answer.

**`multi-correct-move`** — Checks if the user's move is in a set of accepted moves, with one marked as "ideal." Returns `correct` for ideal, `acceptable` for others.

**`exact-sequence`** — For multi-move puzzles. Tracks a sequence of expected moves, auto-responds with opponent moves, and validates each user move in order. Supports optional transposition tolerance.

**`engine-range`** — Runs Stockfish evaluation on both the user's move and the best move. Returns `correct` if the user's move is within `acceptableRange` centipawns of the best. This is the key evaluator for avoiding hardcoded chess logic in content.

**`engine-best`** — Stricter: only accepts the engine's top-N moves at a given depth.

**`rule-based`** — Evaluates against declarative rules rather than specific moves. Example params:
```yaml
params:
  rules:
    - type: "piece-develops"              # a piece moves from back rank
    - type: "controls-square"
      square: "d4"
    - type: "does-not-weaken"
      evaluation: "king-safety"
  allRequired: false                       # any rule match = acceptable
```
Rule evaluators are registered functions that operate on position diff (before/after move).

**`pedagogy-composite`** — Chains multiple evaluators with weighted scoring. Content authors can combine engine evaluation with rule-based evaluation:
```yaml
evaluatorId: "pedagogy-composite"
params:
  evaluators:
    - id: "engine-range"
      weight: 0.6
      params: { engineDepth: 12, acceptableRange: 0.3 }
    - id: "rule-based"
      weight: 0.4
      params:
        rules:
          - type: "piece-develops"
```

### 5.4 Evaluator Registration

```typescript
// packages/eval-sdk/src/registry.ts

interface IEvaluatorRegistry {
  register(evaluator: IEvaluator): void;
  get(evaluatorId: string): IEvaluator | null;
  list(): EvaluatorManifest[];
  validateContentReference(evaluatorId: string, params: unknown): ValidationResult;
}
```

New evaluators are added by implementing `IEvaluator` and registering them at service startup. Content references evaluators by `id` — the registry resolves them at runtime. This is the extension point: a team can ship a new evaluator without touching any content or any other service.

---

## 6. Progression Engine

### 6.1 Core Abstractions

The progression engine operates on **skill atoms** — opaque string identifiers that represent the smallest unit of learnable knowledge. The progression engine has no idea what `"opening.center.pawn-placement"` means in chess terms. It only knows that a user attempted this skill at a given time and achieved a given score.

```typescript
// packages/progression-sdk/src/models/types.ts

/** The smallest trackable unit of knowledge. */
interface SkillAtom {
  readonly id: string;                          // e.g., "opening.center.pawn-placement"
  readonly domain: string;                      // e.g., "opening" — for UI grouping only
  readonly metadata: Record<string, unknown>;   // content-defined, engine-opaque
}

/** A single data point: user attempted a skill and scored X. */
interface AttemptRecord {
  readonly userId: string;
  readonly skillAtomId: string;
  readonly timestamp: number;
  readonly score: number;                       // 0.0 – 1.0
  readonly attemptNumber: number;
  readonly timeSpentMs: number;
  readonly sourceType: 'lesson' | 'puzzle' | 'review';
  readonly sourceId: string;                    // lesson or puzzle ID
  readonly mistakeCategory?: string;            // see 6.3
}

/** Computed mastery state for one skill atom. */
interface MasteryRecord {
  readonly userId: string;
  readonly skillAtomId: string;
  readonly level: MasteryLevel;
  readonly confidence: number;                  // 0.0 – 1.0 (Bayesian certainty)
  readonly lastAttempt: number;                 // timestamp
  readonly nextReviewAt: number;                // scheduled review timestamp
  readonly attemptCount: number;
  readonly streakCount: number;                 // consecutive correct
  readonly decayFactor: number;                 // forgetting curve parameter
}

type MasteryLevel = 'unstarted' | 'learning' | 'practiced' | 'mastered' | 'decayed';
```

### 6.2 Mastery State Machine

```
                    first attempt
  ┌──────────┐  ─────────────────▶  ┌──────────┐
  │ unstarted│                      │ learning │
  └──────────┘                      └────┬─────┘
                                         │
                              score > threshold N times
                                         │
                                    ┌────▼─────┐
                                    │ practiced│
                                    └────┬─────┘
                                         │
                              sustained accuracy + retention
                                         │
                                    ┌────▼─────┐
                          ┌─────────│ mastered │
                          │         └──────────┘
                   time decay                    
                   (no review)                   
                          │                      
                     ┌────▼─────┐                
                     │ decayed  │                
                     └──────────┘                
                     (re-enters learning on next attempt)
```

Transition thresholds are configurable per-domain, not hardcoded. The content layer defines what thresholds are appropriate via course metadata:

```yaml
# _course.yaml
progressionConfig:
  learningThreshold: 0.6           # score needed to move to 'practiced'
  practicedReps: 3                 # correct reps to move to 'practiced'
  masteryReps: 5                   # sustained reps for 'mastered'
  masteryMinConfidence: 0.85
  decayHalfLifeHours: 168          # 1 week
```

### 6.3 Mistake Categorization Framework

The progression engine maintains a **taxonomy of mistake types** defined as a registry, not hardcoded categories.

```typescript
// packages/progression-sdk/src/categorizers/types.ts

interface IMistakeCategorizer {
  readonly id: string;

  /**
   * Given an evaluation result, classify the mistake.
   * Returns null if the attempt was correct (no mistake).
   */
  categorize(
    attempt: AttemptRecord,
    evaluationResult: EvaluationResult,
    position: string
  ): MistakeCategory | null;
}

interface MistakeCategory {
  readonly id: string;                          // e.g., "blunder.hanging-piece"
  readonly severity: 'minor' | 'moderate' | 'critical';
  readonly parentId?: string;                   // hierarchical categories
  readonly metadata: Record<string, unknown>;
}
```

Mistake categories are **registered by the content domain**, not the engine. The engine stores them and uses them for scheduling (e.g., "this user keeps making `tactical.missed-fork` mistakes, so surface more fork exercises"), but it doesn't interpret what `tactical.missed-fork` means.

### 6.4 Adaptive Scheduler Interface

```typescript
// packages/progression-sdk/src/schedulers/interface.ts

interface IScheduler {
  /**
   * Given the user's mastery state, determine what they should do next.
   * Returns a ranked list of content recommendations.
   */
  getNextContent(
    userId: string,
    masteryRecords: MasteryRecord[],
    availableContent: ContentManifest[],
    constraints: SchedulingConstraints
  ): ContentRecommendation[];
}

interface SchedulingConstraints {
  maxSessionMinutes?: number;
  preferredDifficulty?: 'easy' | 'medium' | 'hard';
  focusDomain?: string;                        // e.g., "endgame"
  includeReview: boolean;
  reviewRatio: number;                          // 0.0 – 1.0, portion of session for review
}

interface ContentRecommendation {
  contentId: string;
  contentType: 'lesson' | 'puzzle' | 'review';
  reason: SchedulingReason;
  priority: number;
  estimatedMinutes: number;
}

type SchedulingReason =
  | { type: 'new-content'; prerequisitesMet: true }
  | { type: 'review-due'; lastSeen: number; decayLevel: number }
  | { type: 'weakness-remediation'; mistakeCategory: string; frequency: number }
  | { type: 'reinforcement'; skillAtomId: string; currentLevel: MasteryLevel };
```

### 6.5 Progression API

```typescript
// services/progression-engine/src/api.ts

interface IProgressionAPI {
  /** Record a completed attempt. Triggers mastery recalculation. */
  recordAttempt(attempt: AttemptRecord): Promise<MasteryRecord>;

  /** Get current mastery state for a user. */
  getMastery(userId: string, domain?: string): Promise<MasteryRecord[]>;

  /** Get next recommended content. */
  getNextRecommendation(userId: string, constraints: SchedulingConstraints): Promise<ContentRecommendation[]>;

  /** Get mistake analysis for a user. */
  getMistakeAnalysis(userId: string, timeRange?: TimeRange): Promise<MistakeAnalysis>;

  /** Get aggregate progress metrics for a domain. */
  getDomainProgress(userId: string, domain: string): Promise<DomainProgress>;
}

interface DomainProgress {
  domain: string;
  totalSkills: number;
  masteredCount: number;
  practicedCount: number;
  learningCount: number;
  unstartedCount: number;
  overallScore: number;                         // weighted aggregate
  currentStreak: number;                        // days
  estimatedCompletion: number | null;           // hours remaining
}
```

---

## 7. Explore / Sandbox Mode

### 7.1 Architecture

The sandbox is the simplest mode architecturally — it's the board engine in `FreeExploreMode` with persistence.

```
┌───────────────────────────────────────────────┐
│              BOARD EXPLORER VIEW               │
│                                               │
│  ┌─────────────┐  ┌──────────────────────┐   │
│  │  Board UI   │  │  Analysis Panel      │   │
│  │  (board-ui) │  │  ┌────────────────┐  │   │
│  │             │  │  │ Move Tree View │  │   │
│  │  Drag/drop  │  │  │ (collapsible)  │  │   │
│  │  Right-click│  │  └────────────────┘  │   │
│  │  annotations│  │  ┌────────────────┐  │   │
│  │             │  │  │ Engine Eval    │  │   │
│  │             │  │  │ (future)       │  │   │
│  │             │  │  └────────────────┘  │   │
│  │             │  │  ┌────────────────┐  │   │
│  │             │  │  │ Notes Editor  │  │   │
│  └─────────────┘  │  └────────────────┘  │   │
│                    └──────────────────────┘   │
└───────────────────────────────────────────────┘
```

### 7.2 Branching Analysis Tree

In explore mode, every move the user makes is a node in the move tree. When the user goes back and plays a different move, a **branch** is created. The tree is unbounded in depth and branching factor.

```
                    root (starting position)
                     │
                     ├── e2e4
                     │    ├── e7e5  (main line)
                     │    │    ├── g1f3
                     │    │    │    ├── b8c6
                     │    │    │    └── d7d6  (branch)
                     │    │    └── f2f4  (branch: King's Gambit)
                     │    └── c7c5  (branch: Sicilian)
                     └── d2d4  (branch)
                          └── d7d5
```

The cursor can be at any node. The user can navigate, branch, annotate any node, and the tree persists across sessions.

### 7.3 State Persistence

```typescript
// Serialized format stored in DB or localStorage

interface SavedAnalysis {
  id: string;
  userId: string;
  title: string;
  createdAt: number;
  updatedAt: number;
  startingPosition: string;                     // FEN
  tree: SerializedMoveTree;                     // JSON-serialized tree
  cursorNodeId: string;
  tags: string[];
  isPublic: boolean;
}

interface SerializedMoveTree {
  nodes: SerializedNode[];                      // flat array, references by ID
  rootId: string;
}

interface SerializedNode {
  id: string;
  parentId: string | null;
  childIds: string[];                           // ordered; first = main line
  move: string | null;                          // UCI
  san: string | null;
  fen: string;
  comment?: string;
  nag?: number[];
  annotations?: Annotation[];
  evaluation?: { depth: number; score: number; pv: string[] };
}
```

### 7.4 Persistence Strategy

**MVP:** Client-side auto-save to backend via REST. Debounced — save after 2 seconds of inactivity, or on every 5th move. The serialized tree is compact (a 100-move game with 10 branches is ~15KB JSON).

**Future:** Real-time sync via WebSocket for multi-device and collaborative analysis. Operational transforms or CRDTs on the tree structure for conflict resolution.

### 7.5 Engine Integration (Future)

```typescript
// Future interface — not MVP

interface IEngineAnalysis {
  /** Start continuous analysis at the current position. */
  startAnalysis(position: string, options: AnalysisOptions): AnalysisStream;

  /** Stop analysis. */
  stopAnalysis(): void;

  /** Get cached evaluation for a position. */
  getCachedEval(position: string): EngineEvaluation | null;
}

interface AnalysisStream {
  on(event: 'depth', handler: (eval: EngineEvaluation) => void): void;
  on(event: 'bestmove', handler: (move: string) => void): void;
  on(event: 'error', handler: (error: Error) => void): void;
}

interface AnalysisOptions {
  maxDepth: number;
  multiPV: number;                              // number of lines to analyze
  timeoutMs: number;
}
```

Engine analysis runs server-side via the eval-engine's Stockfish pool. Results stream to the client via WebSocket and are cached per-position in Redis.

---

## 8. MVP vs Future Proofing

### 8.1 MVP Scope (v1.0)

| Component | MVP Scope |
|---|---|
| **Board Engine** | Full implementation: state, validation, tree, annotations, all 4 modes |
| **Board UI** | SVG renderer, drag-drop, click-to-move, right-click annotations, responsive |
| **Content Engine** | YAML loader, JSON Schema validation, REST API for content serving |
| **Evaluation Engine** | `exact-move`, `multi-correct-move`, `exact-sequence` evaluators only. No Stockfish. |
| **Progression Engine** | Mastery tracking, basic spaced repetition. No adaptive scheduling. |
| **Lesson Player** | Step-through lessons, interactive moves, hints, feedback display |
| **Puzzle Solver** | Single-position puzzles, move validation, streak tracking |
| **Explore Mode** | Free play, branching, annotations, save/load. No engine analysis. |
| **Progress Dashboard** | Per-domain progress bars, streak counter, XP/level display |
| **Auth** | Email/password, JWT sessions |
| **Infrastructure** | Single PostgreSQL instance, Redis cache, S3 for content, monolith API |

### 8.2 Extension Points Reserved

These are deliberate architectural seams where future features plug in without refactoring:

| Extension Point | Seam | Future Feature |
|---|---|---|
| `IEvaluator` plugin registry | `eval-sdk/registry.ts` | Stockfish evaluator, rule-based evaluator, ML-based evaluator |
| `IBoardMode` interface | `board-engine/modes/` | Multiplayer mode, tournament mode, coaching mode |
| `IScheduler` interface | `progression-sdk/schedulers/` | ML-adaptive scheduling, weakness-targeted practice |
| `IMistakeCategorizer` registry | `progression-sdk/categorizers/` | Deep mistake analysis, pattern detection |
| `IEngineAnalysis` interface | `eval-sdk/` | Live Stockfish analysis in explore mode |
| Content `type` field in steps | `content/schemas/` | New step types: video, interactive diagram, timed challenge |
| `ModeEffect` union type | `board-engine/modes/types.ts` | New side effects: sound, haptic, animation triggers |
| Real-time sync layer | `services/` | Multi-device sync, collaborative analysis, live spectating |
| Event bus | `shared-types/events.ts` | Analytics pipeline, achievement system, social feed |
| Board UI theme system | `board-ui/themes/` | Custom board/piece sets, accessibility themes |

### 8.3 What We Deliberately Do NOT Build in MVP

**No Stockfish integration.** The `engine-range` and `engine-best` evaluators are defined but not implemented. All MVP content uses `exact-move`, `multi-correct-move`, or `exact-sequence`. This avoids the operational complexity of managing Stockfish processes, but the evaluator interface is ready for it.

**No multiplayer.** The board engine supports it architecturally (modes are pluggable, state is serializable), but we don't build WebSocket game rooms or matchmaking.

**No ML-based adaptive scheduling.** The scheduler interface exists, but MVP uses a simple spaced-repetition algorithm (SM-2 variant). The `IScheduler` interface allows a drop-in replacement.

**No mobile native apps.** The board engine is framework-agnostic by design. When we build React Native or Flutter clients, they import `board-engine` directly and only replace `board-ui`.

**No content marketplace or user-generated content.** The content pipeline is designed for internal authors. But the YAML-based content format and validation pipeline could be exposed to external authors in the future.

---

## Appendix A: Technology Recommendations

| Layer | Recommendation | Rationale |
|---|---|---|
| Monorepo tool | Turborepo | Fast, JS-native, good caching |
| Frontend | React + Vite | Fast dev server, broad ecosystem |
| Board rendering | SVG (MVP), Canvas (future) | SVG is simpler for annotations and accessibility |
| Backend | Node.js + Fastify | Shares types with frontend, fast HTTP |
| Database | PostgreSQL 16 | JSONB for flexible schemas, proven reliability |
| Cache | Redis | Evaluation cache, session store, rate limiting |
| Content storage | S3 + CloudFront | Versioned content, global CDN |
| Event streaming | pg_notify (MVP), Kafka (future) | Start simple, migrate when scale demands |
| CI/CD | GitHub Actions | Monorepo-native with Turborepo integration |
| Analytics store | ClickHouse (future) | Columnar, fast aggregation. MVP uses PostgreSQL. |

## Appendix B: Key Architectural Decisions Record

**ADR-001: Board engine is client-only.**  
We rejected server-authoritative board state for MVP. Latency on every move is unacceptable for a learning tool. Server validation happens only at evaluation boundaries (when a user submits a puzzle answer). Multiplayer mode will require server authority — the mode system supports this without changing the engine.

**ADR-002: Content is YAML, not a CMS.**  
We chose flat-file YAML over a database-backed CMS because: (a) content authors can use Git workflow with PR review, (b) content is versioned automatically, (c) no CMS admin UI to build for MVP, (d) migration to a CMS later only requires changing the content-engine's storage backend.

**ADR-003: Evaluation is server-side, not client-side.**  
Even for simple evaluators, we run evaluation on the server to prevent cheating (inspecting correct answers in client-side JavaScript). The evaluation request/response cycle adds ~50ms latency, which is acceptable.

**ADR-004: Progression engine is domain-agnostic.**  
The progression engine never interprets skill atom IDs. This was a deliberate constraint to allow the same engine to power future non-chess learning products (e.g., poker, Go) without modification.

---

*End of RFC. Review requested from: Frontend Lead, Backend Lead, Content Lead, Product.*
