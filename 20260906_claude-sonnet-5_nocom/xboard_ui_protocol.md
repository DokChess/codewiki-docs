# XBoard UI Protocol Module

## Introduction

The **xboard_ui_protocol** module implements the [XBoard/WinBoard chess engine communication protocol](https://www.gnu.org/software/xboard/engine-intf.html) for the DokChess engine. It acts as the outermost adapter layer that allows DokChess to be plugged into any standard chess GUI that speaks the xboard protocol (e.g. XBoard, WinBoard, or various chess frontends).

The module is centered around a single class, `XBoard`, which owns a text-based, blocking read/write loop over an arbitrary `Reader`/`Writer` pair (typically standard input/output). It translates simple text commands from the GUI into calls against the engine's [`Engine`](engine_core.md) interface and [`ChessRules`](chess_rules.md) contract, and translates engine results back into xboard-formatted text output.

This module is a **child** of the broader [xboard_ui](xboard_ui.md) module and works hand-in-hand with its sibling [xboard_ui_move_translation](xboard_ui_move_translation.md), which provides the `MoveParser` component responsible for converting between xboard's plain-text move notation (e.g. `e2e4`, `a7a8q`) and DokChess's internal `Move` domain object.

---

## Purpose and Core Functionality

`XBoard` is the **protocol driver**: it does not know how to play chess, evaluate positions, or generate legal moves itself. Instead, it:

1. Reads one line of text at a time from the input stream.
2. Recognizes a small set of xboard protocol commands (`xboard`, `protover 2`, `new`, `go`, `quit`) plus raw move strings.
3. Delegates actual chess logic to injected collaborators:
   - `Engine` — to set up pieces, request the engine's next move (asynchronously, via an RxJava `Observable<Move>`), apply moves to the engine's internal state, and shut the engine down.
   - `ChessRules` (optional) — to validate that an opponent's move is legal before forwarding it to the engine.
4. Maintains its own local `Position` mirror so that incoming moves can be parsed/interpreted correctly (e.g., detecting captures, promotions) and validated against the rules engine.
5. Subscribes to the `Engine`'s move `Observable` as an RxJava `Observer<Move>`, allowing it to react asynchronously when the engine reports candidate moves (`onNext`) and its final decision (`onCompleted`), or an error (`onError`).
6. Writes properly formatted xboard responses back out (e.g. `move e2e4`, `Illegal move: ...`, `tellusererror ...`).

### Supported Commands

| Input Line | Behavior |
|---|---|
| `xboard` | Acknowledges xboard mode; writes an empty line. |
| `protover 2` | Declares feature negotiation complete (`feature done=1`). |
| `new` | Resets local `Position` to the starting position and calls `engine.setupPieces(position)`. |
| `go` | Asks the engine to determine its own move via `engine.determineYourMove()`, subscribing to the result. |
| `<move string>` (e.g. `e2e4`) | Parsed via `MoveParser.fromXboard`. If legal (per optional `ChessRules` check), applied to both the engine and the local `Position`, then the engine is asked to respond with `determineYourMove()`. |
| `quit` or end-of-stream (`null`) | Terminates the read loop and calls `engine.close()`. |
| anything else | Writes `Error (unknown command): <line>`. |

---

## Architecture

### Component Diagram

```mermaid
graph TD
    GUI["External Chess GUI<br/>(XBoard / WinBoard)"]
    XBoard["XBoard<br/>(protocol driver, Observer&lt;Move&gt;)"]
    MoveParser["MoveParser<br/>(xboard_ui_move_translation)"]
    Engine["Engine interface<br/>(engine_core)"]
    ChessRules["ChessRules interface<br/>(chess_rules_engine)"]
    Position["Position / Move<br/>(domain)"]

    GUI -- "text lines (stdin)" --> XBoard
    XBoard -- "text lines (stdout)" --> GUI
    XBoard -- "fromXboard / toXboard" --> MoveParser
    XBoard -- "setupPieces / determineYourMove / performMove / close" --> Engine
    XBoard -- "getLegalMoves" --> ChessRules
    XBoard -- "reads/updates local mirror" --> Position
    MoveParser -- "constructs" --> Position
```

### Class Relationships

```mermaid
classDiagram
    class XBoard {
        -Reader input
        -BufferedReader bufferedReader
        -Writer output
        -ChessRules chessRules
        -Engine engine
        -Move bestMove
        -Position position
        -MoveParser moveParser
        +setInput(Reader)
        +setOutput(Writer)
        +setChessRules(ChessRules)
        +setEngine(Engine)
        +play()
        +onCompleted()
        +onError(Throwable)
        +onNext(Move)
    }
    class MoveParser {
        +fromXboard(String, Position) Move
        +toXboard(Move) String
    }
    class Engine {
        <<interface>>
        +setupPieces(Position)
        +determineYourMove() Observable~Move~
        +performMove(Move)
        +close()
    }
    class ChessRules {
        <<interface>>
        +getStartingPosition() Position
        +getLegalMoves(Position) Collection~Move~
        +isCheck(Position, Colour) boolean
        +isCheckmate(Position) boolean
        +isStalemate(Position) boolean
    }

    class RxObserverMove {
        <<interface>>
    }

    XBoard --> MoveParser : uses
    XBoard --> Engine : delegates to
    XBoard --> ChessRules : validates with
    XBoard ..|> RxObserverMove : implements
```

---

## Data Flow / Process Diagrams

### Main Read-Eval-Respond Loop

```mermaid
flowchart TD
    Start(["play invoked"]) --> ReadLine["readLine"]
    ReadLine --> IsNull{"line == null<br/>or 'quit'?"}
    IsNull -- yes --> StopLoop["running = false"]
    IsNull -- no --> IsXboard{"line == 'xboard'?"}
    IsXboard -- yes --> WriteEmpty["writeLine empty string"] --> ReadLine
    IsXboard -- no --> IsProtover{"line == 'protover 2'?"}
    IsProtover -- yes --> WriteFeature["writeLine 'feature done=1'"] --> ReadLine
    IsProtover -- no --> IsNew{"line == 'new'?"}
    IsNew -- yes --> ResetPos["reset position; engine.setupPieces"] --> ReadLine
    IsNew -- no --> IsGo{"line == 'go'?"}
    IsGo -- yes --> AskEngineMove["engine.determineYourMove; subscribe this"] --> ReadLine
    IsGo -- no --> ParseMove["moveParser.fromXboard line, position"]
    ParseMove --> MoveOk{"move != null?"}
    MoveOk -- no --> Unknown["writeLine 'Error unknown command'"] --> ReadLine
    MoveOk -- yes --> HasRules{"chessRules set?"}
    HasRules -- yes --> CheckLegal{"move in getLegalMoves position?"}
    CheckLegal -- no --> Illegal["writeLine 'Illegal move'"] --> ReadLine
    CheckLegal -- yes --> ApplyMove
    HasRules -- no --> ApplyMove["engine.performMove; update position"]
    ApplyMove --> AskEngineMove2["engine.determineYourMove; subscribe this"] --> ReadLine
    StopLoop --> CloseEngine["engine.close"]
    CloseEngine --> End(["Loop ends"])
```

### Engine Move Subscription Lifecycle (Observer callbacks)

```mermaid
sequenceDiagram
    participant GUI as GUI (stdin/stdout)
    participant XB as XBoard
    participant ENG as Engine
    participant MP as MoveParser

    GUI->>XB: "go" (or opponent move line)
    XB->>ENG: determineYourMove()
    ENG-->>XB: Observable<Move>
    XB->>ENG: subscribe(this)

    loop candidate moves emitted during search
        ENG-->>XB: onNext(move)
        XB->>GUI: "# better move found: <move>"
        Note over XB: bestMove = move
    end

    alt search completes successfully
        ENG-->>XB: onCompleted()
        XB->>MP: toXboard(bestMove)
        MP-->>XB: "move <from><to>[promo]"
        XB->>GUI: writeLine(formatted move)
        XB->>ENG: performMove(bestMove)
        Note over XB: position = position.performMove(bestMove)
    else search errors
        ENG-->>XB: onError(exception)
        XB->>GUI: "tellusererror <message>"
    end
```

### Opponent Move Handling and Legality Check

```mermaid
sequenceDiagram
    participant GUI as GUI
    participant XB as XBoard
    participant MP as MoveParser
    participant CR as ChessRules
    participant ENG as Engine

    GUI->>XB: line (e.g. "e2e4")
    XB->>MP: fromXboard(line, position)
    MP-->>XB: Move or null
    alt move == null
        XB->>GUI: "Error (unknown command): ..."
    else move parsed
        opt chessRules configured
            XB->>CR: getLegalMoves(position)
            CR-->>XB: Collection<Move>
            alt move not legal
                XB->>GUI: "Illegal move: ..."
            end
        end
        XB->>ENG: performMove(move)
        Note over XB: position = position.performMove(move)
        XB->>ENG: determineYourMove()
        ENG-->>XB: Observable<Move> (subscribed)
    end
```

---

## Key Design Points

- **Thin, stateless-ish protocol shell**: `XBoard` intentionally contains no chess logic (move generation, evaluation, search). All actual decision-making is delegated to the [`Engine`](engine_core.md) abstraction, keeping this module focused purely on protocol parsing/formatting and I/O.
- **Local `Position` mirror**: Although the `Engine` maintains its own authoritative internal state, `XBoard` keeps a parallel `Position` object. This is required because `MoveParser.fromXboard` needs a `Position` to resolve ambiguous xboard move strings into fully-qualified `Move` objects (identifying the piece being moved, detecting captures, and detecting pawn promotions).
- **Optional legality enforcement**: The `ChessRules` collaborator is optional (`setChessRules` may be left unset). When provided, incoming opponent moves are checked against `getLegalMoves(position)` before being forwarded to the engine, allowing the module to reject illegal input from the GUI/opponent with an `Illegal move:` message.
- **Asynchronous move determination via RxJava**: `Engine.determineYourMove()` returns an `Observable<Move>` rather than a single synchronous value. This allows the underlying search algorithm (see [engine_search](engine_search.md)) to report improving candidate moves as they are found (`onNext`), with `XBoard` only committing to and reporting the final choice once the `Observable` completes (`onCompleted`). This directly supports iterative/parallel search strategies such as those in [engine_search_parallel_search](engine_search_parallel_search.md), where intermediate best-move updates are streamed before a final result is settled.
- **Synchronized output**: `writeLine` synchronizes on the shared `output` writer, ensuring that engine-driven asynchronous notifications (from `onNext`/`onCompleted`, potentially triggered on a different thread by the underlying `Observable`) don't interleave corrupted text with the main loop's output.
- **Blocking read loop**: The main `play()` method runs on a single thread and blocks on `bufferedReader.readLine()`, which is standard for xboard's synchronous, line-oriented protocol.

---

## Dependencies on Other Modules

| Dependency | Role |
|---|---|
| [xboard_ui_move_translation](xboard_ui_move_translation.md) (`MoveParser`) | Converts between xboard plain-text move notation and the domain `Move` type. |
| [engine_core](engine_core.md) (`Engine`) | Provides the actual move-determination, move-application, and lifecycle (setup/close) behavior that `XBoard` orchestrates. |
| [chess_rules_engine](chess_rules.md) (`ChessRules`) | Optionally supplies legal-move validation for opponent input. |
| [domain](domain.md) (`Position`, `Move`) | Core value types representing board state and moves, used throughout the parsing/validation/application flow. |

`XBoard` itself is composed and wired together (its `Reader`/`Writer`/`Engine`/`ChessRules` dependencies injected) by the application's entry point — see [main_entry](main_entry.md) — and is exercised end-to-end by [integration_tests](integration_tests.md) (`XBoardIntegTest`), which drives the protocol loop with scripted input/output to verify correct behavior against a real engine and rules implementation.

---

## Usage Example (Wiring Sketch)

```mermaid
flowchart LR
    Main["Main - main_entry"] -->|"new XBoard"| XB["XBoard instance"]
    Main -->|"setInput System.in"| XB
    Main -->|"setOutput System.out"| XB
    Main -->|"setChessRules DefaultChessRules"| XB
    Main -->|"setEngine DefaultEngine"| XB
    Main -->|"xb.play"| XB
```

This mirrors how [main_entry](main_entry.md) bootstraps the application: constructing a `ChessRules` implementation (e.g. `DefaultChessRules` from [chess_rules_engine](chess_rules.md)) and an `Engine` implementation (e.g. `DefaultEngine` from [engine_core](engine_core.md), itself backed by [engine_search](engine_search.md), [engine_evaluation](engine_evaluation.md), and optionally an [opening_library](opening_library.md)/[opening_polyglot](opening_polyglot.md) book), then handing both to an `XBoard` instance bound to standard input/output before invoking `play()`.
