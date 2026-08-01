# Piece Moves Module Documentation

## Overview

The `piece_moves` module is a core component of the DokChess chess engine that defines the movement rules for individual chess pieces. This module contains specialized classes for each chess piece type (knight, rook, bishop, queen, pawn, and king) that implement how these pieces can legally move on the chess board according to standard chess rules.

## Module Structure

The module consists of six main classes, each implementing movement rules for a specific chess piece:

- **KnightMoves**: Implements L-shaped knight movements
- **RookMoves**: Implements orthogonal rook movements
- **BishopMoves**: Implements diagonal bishop movements
- **QueenMoves**: Implements combined rook and bishop movements
- **PawnMoves**: Implements pawn-specific movements including advances, captures, and promotions
- **KingMoves**: Implements one-square king movements

## Architecture and Relationships

### Core Dependencies

The piece_moves module depends on several other modules in the DokChess system:

- [`domain`](domain.md): Provides fundamental chess domain objects like `Position`, `Square`, and `Piece`
- [`movement_base`](movement_base.md): Contains base classes for movement implementations
- [`special_moves`](special_moves.md): Handles special moves like castling (though king moves don't directly handle castling)

### Component Interactions

The piece moves classes inherit from either `Movement` or `ComplexMovement` base classes and work together with the core chess rules implementation to determine valid moves.

## Detailed Component Documentation

### KnightMoves

The `KnightMoves` class implements the unique L-shaped movement pattern of the knight. It can move two squares in one direction and then one square perpendicular to that direction, making it the only piece that can jump over other pieces.

### RookMoves

The `RookMoves` class implements orthogonal movements for the rook, allowing it to move any number of squares horizontally or vertically until blocked by another piece or the edge of the board.

### BishopMoves

The `BishopMoves` class implements diagonal movements for the bishop, allowing it to move any number of squares diagonally until blocked by another piece or the edge of the board.

### QueenMoves

The `QueenMoves` class combines both rook and bishop movement patterns, allowing the queen to move any number of squares in any direction (horizontal, vertical, or diagonal).

### PawnMoves

The `PawnMoves` class implements the complex pawn movement rules including:
- Single and double square advancement
- Diagonal captures
- En passant captures
- Promotion mechanics

### KingMoves

The `KingMoves` class implements standard king movement rules, allowing one square in any direction. Note that castling is handled separately in the special moves module.

## Integration with System Components

The piece_moves module integrates with the broader system through:

1. **ChessRules Interface**: The movement rules are used by the core chess rules implementation to validate moves
2. **Position Management**: All movement calculations depend on current board position state
3. **Move Generation**: These classes are responsible for generating valid move candidates for each piece type

## Usage Patterns

The piece_moves classes are typically used during:
- Move generation for legal moves
- Move validation against current board state
- Engine decision-making processes

Each class follows a consistent pattern of determining reachable squares based on the current position and source square, then returning appropriate move candidates.

## Related Modules

- [domain](domain.md): Provides the foundational chess domain objects
- [movement_base](movement_base.md): Contains base classes for movement implementations
- [core_rules](core_rules.md): Coordinates all movement rules within the chess game logic
- [special_moves](special_moves.md): Handles special moves like castling that aren't covered by regular piece movement rules

## Implementation Details

All piece moves classes extend either `Movement` or `ComplexMovement` base classes, which provide utility methods for checking square reachability and handling common movement patterns. The classes are designed to be lightweight and efficient, focusing solely on movement calculation without side effects.

The module uses the coordinate system where:
- Files (columns) are numbered 0-7 (a-h)
- Ranks (rows) are numbered 0-7 (1-8)
- White pieces start at rank 6-7, black pieces at rank 0-1