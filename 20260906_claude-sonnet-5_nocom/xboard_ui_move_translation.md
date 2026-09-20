# XBoard UI – Move Translation

## Introduction

The **xboard_ui_move_translation** module is a small but critical translation
layer inside the [xboard_ui](xboard_ui_protocol.md) subsystem. It converts
chess moves between two representations:

* the plain-text move notation used by the **XBoard/WinBoard chess engine
  communication protocol** (e.g. `e2e4`, `e7e8q`), and
* the strongly-typed [`Move`](domain.md) object used internally throughout
  the DokChess engine.

It is the single component responsible for parsing incoming protocol
commands into domain moves, and for serializing domain moves back into
protocol-compliant strings when the engine reports its choice. This makes it
the boundary between "text protocol" and "internal chess domain" — every
other component in the engine works exclusively with `Move`, `Position`,
`Square`, and `Piece` objects from the [`domain`](domain.md) module.

The module consists of a single class:

| Component | Responsibility |
|---|---|
| `MoveParser` | Bidirectional translation between XBoard move strings and `Move` objects |

---

## 1. Purpose and Core Functionality

`MoveParser` provides exactly two operations:

1. **`fromXboard(String input, Position position)`** — Parses a raw line of
   text received from an XBoard-compatible GUI/controller and, if it matches
   the coordinate-move pattern, converts it into a fully-populated `Move`
   instance (including piece identity, capture flag, and promotion piece
   type). If the input does not look like a move, `null` is returned so that
   the caller ([`XBoard`](xboard_ui_protocol.md)) can try to interpret it as
   a different protocol command.

2. **`toXboard(Move move)`** — Serializes an internal `Move` object into the
   XBoard `move <from><to>[promotion]` wire format that the protocol expects
   the engine to emit after it decides on a move.

Because parsing requires knowledge of *what piece is on the board*, whether a
capture is occurring, and whether a pawn is promoting, `MoveParser` needs
read-only access to the current [`Position`](domain.md) — it does not itself
mutate any game state. All state mutation (applying the move to the board)
happens later, driven by [`Position.performMove`](domain.md) and the
[`Engine`](engine_core.md).

### Move String Format

XBoard's coordinate notation is a fixed-width string:

```
<fromFile><fromRank><toFile><toRank>[<promotionLetter>]
```

Examples:
* `e2e4` — pawn from e2 to e4
* `e7e8q` — pawn promotes to queen on e8
* `g1f3` — knight move

The class validates input against the regular expression:

```
[a-h][1-8][a-h][1-8][qrnb]?
```

Any string not matching this pattern yields `null` from `fromXboard`,
signaling "this is not a move".

---

## 2. Architecture

### 2.1 Component Position in the System

`MoveParser` sits directly beneath [`XBoard`](xboard_ui_protocol.md), which
owns the read/process/respond loop of the XBoard protocol. `XBoard` delegates
*all* move-string handling to `MoveParser`, keeping protocol control flow
separate from notation parsing/formatting concerns.

```mermaid
graph TD
    subgraph xboard_ui["xboard_ui module"]
        XB["XBoard<br/>(xboard_ui_protocol)"]
        MP["MoveParser<br/>(xboard_ui_move_translation)"]
    end

    subgraph domain["domain module"]
        Move["Move"]
        Position["Position"]
        Square["Square"]
        Piece["Piece"]
        PieceType["PieceType"]
    end

    subgraph engine["engine_core module"]
        Engine["Engine"]
    end

    subgraph rules["chess_rules module"]
        ChessRules["ChessRules"]
    end

    XB -->|"delegates parsing/formatting"| MP
    MP -->|constructs| Move
    MP -->|reads pieces from| Position
    MP -->|constructs| Square
    MP -->|reads type from| Piece
    MP -->|resolves letter to| PieceType

    XB --> Engine
    XB --> ChessRules
    XB --> Position

    style MP fill:#f9d77e,stroke:#333,stroke-width:2px
```

### 2.2 Class Diagram

```mermaid
classDiagram
    class MoveParser {
        +fromXboard(String input, Position position) Move
        +toXboard(Move move) String
    }

    class Move {
        +getFrom() Square
        +getTo() Square
        +getPiece() Piece
        +isCapture() boolean
        +isPromotion() boolean
        +getPromotion() PieceType
    }

    class Position {
        +getPiece(Square) Piece
    }

    class Square {
        +Square(String name)
        +getRank() int
        +getFile() int
    }

    class Piece {
        +getType() PieceType
    }

    class PieceType {
        <<enum>>
        +fromLetter(char) PieceType
        +getLetter() char
    }

    MoveParser --> Move : creates
    MoveParser --> Position : queries
    MoveParser --> Square : creates
    MoveParser --> Piece : reads
    MoveParser --> PieceType : uses
    Move --> Square
    Move --> Piece
    Move --> PieceType
```

---

## 3. Data Flow

### 3.1 Parsing an Incoming Move (`fromXboard`)

```mermaid
flowchart TD
    A["Raw protocol line, e.g. 'e7e8q'"] --> B{"Matches regex<br/>[a-h][1-8][a-h][1-8][qrnb]?"}
    B -- no --> Z["return null<br/>(not a move — caller tries other commands)"]
    B -- yes --> C["Parse 'from' Square<br/>from chars 0-1"]
    C --> D["Parse 'to' Square<br/>from chars 2-3"]
    D --> E["Lookup piece at 'from'<br/>via Position.getPiece(from)"]
    E --> F{"Is 'to' occupied?"}
    F -- yes --> G["capture = true"]
    F -- no --> H["capture = false"]
    G --> I
    H --> I{"Is piece a PAWN and<br/>'to' rank is 0 or 7?"}
    I -- yes --> J["Read 5th char as promotion letter<br/>PieceType.fromLetter(c)"]
    I -- no --> K["promotionPieceType = null"]
    J --> L["new Move(piece, from, to, capture, promotionPieceType)"]
    K --> L
    L --> M["return Move"]
```

### 3.2 Serializing an Outgoing Move (`toXboard`)

```mermaid
flowchart LR
    A["Move object<br/>(from, to, promotion)"] --> B["Append 'move '"]
    B --> C["Append move.getFrom()<br/>(Square.toString)"]
    C --> D["Append move.getTo()<br/>(Square.toString)"]
    D --> E{"move.isPromotion()?"}
    E -- yes --> F["Append lowercase promotion letter"]
    E -- no --> G["skip"]
    F --> H["return final string, e.g. 'move e7e8q'"]
    G --> H
```

---

## 4. Sequence: End-to-End Move Exchange in XBoard

The diagram below shows how `MoveParser` is invoked within the broader
protocol loop implemented by [`XBoard`](xboard_ui_protocol.md), illustrating
both directions of translation in a single game turn.

```mermaid
sequenceDiagram
    participant GUI as XBoard/WinBoard GUI
    participant XB as XBoard
    participant MP as MoveParser
    participant POS as Position
    participant RULES as ChessRules
    participant ENG as Engine

    GUI->>XB: "e2e4" (text line)
    XB->>MP: fromXboard("e2e4", position)
    MP->>POS: getPiece(from)
    POS-->>MP: Piece(PAWN, WHITE)
    MP->>POS: getPiece(to)
    POS-->>MP: null (no capture)
    MP-->>XB: Move(piece, e2, e4, capture=false, promotion=null)

    XB->>RULES: getLegalMoves(position)
    RULES-->>XB: Collection<Move>
    XB->>XB: verify move is legal

    XB->>ENG: performMove(move)
    XB->>POS: performMove(move)
    POS-->>XB: new Position

    XB->>ENG: determineYourMove()
    ENG-->>XB: Observable<Move> (async)
    ENG-->>XB: onNext(bestMove)
    XB->>MP: toXboard(bestMove)
    MP-->>XB: "move e7e5"
    XB->>GUI: "move e7e5" (text line)
```

---

## 5. Design Notes and Rationale

* **Stateless translator.** `MoveParser` holds no internal state between
  calls; each `fromXboard`/`toXboard` invocation is self-contained. This
  keeps it trivially reusable and testable — a single instance is created
  once by `XBoard` and reused for the entire game session.

* **Position is required only for parsing.** Determining whether a move is
  a capture, and whether a pawn move is a promotion, requires knowing the
  board state — this is why `fromXboard` takes a `Position` parameter while
  `toXboard` does not (the `Move` object already encodes capture/promotion
  flags once constructed).

* **Graceful non-move handling.** Returning `null` for non-matching input
  (rather than throwing an exception) allows `XBoard` to treat unrecognized
  lines as candidate protocol commands (`quit`, `new`, `go`, etc.) rather
  than immediately failing, keeping the protocol loop resilient to the wide
  variety of commands XBoard/WinBoard may send.

* **Deferred rule validation.** `MoveParser` performs only *syntactic*
  parsing. It does not check chess legality — that responsibility belongs to
  [`ChessRules`](chess_rules_engine.md), which `XBoard` consults separately
  after parsing succeeds. This separation keeps `MoveParser` simple and
  free of rules-engine dependencies.

* **Symmetry with `Move.toString()`.** Note that `Move` already provides a
  human-readable `toString()` (e.g. `"P e2-e4"`) used for logging/debugging.
  `MoveParser.toXboard` is a distinct, protocol-specific serialization
  (`"move e2e4"`) and must not be confused with `Move.toString()`.

---

## 6. Related Modules

| Module | Relationship |
|---|---|
| [domain.md](domain.md) | Supplies `Move`, `Piece`, `PieceType`, `Square`, and `Position` — the core types `MoveParser` consumes and produces. |
| [xboard_ui_protocol.md](xboard_ui_protocol.md) | `XBoard` is the sole consumer of `MoveParser`, driving the read-eval-respond loop of the XBoard protocol. |
| [engine_core.md](engine_core.md) | Consumes translated `Move` objects via `Engine.performMove` and produces `Move` results via `Engine.determineYourMove`, which `XBoard` then passes to `MoveParser.toXboard`. |
| [chess_rules_engine.md](chess_rules_engine.md) | Provides legality checks (`ChessRules.getLegalMoves`) applied to moves *after* they have been parsed by `MoveParser`. |
