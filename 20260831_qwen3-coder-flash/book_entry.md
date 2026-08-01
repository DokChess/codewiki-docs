# Book Entry Module Documentation

## Overview

The `book_entry` module is responsible for handling individual entries within Polyglot opening books. This module provides the core functionality for parsing, representing, and manipulating book entries that contain chess moves and their associated weights for use in opening book systems.

## Purpose and Core Functionality

The `BookEntry` class represents a single entry in a Polyglot opening book format. Each entry contains:
- A key (8 bytes) representing the board position
- A move (2 bytes) indicating the chess move to play
- A weight (2 bytes) indicating the preference or frequency of this move

The module implements the `Comparable` interface to allow sorting entries by weight in descending order, which is essential for prioritizing moves in opening book usage.

## Module Architecture

### Core Components

The main component of this module is the `BookEntry` class, which handles:
1. **Data Parsing**: Extracting key, move, and weight from raw byte arrays
2. **Move Conversion**: Converting compact move representations into readable algebraic notation
3. **Weight Management**: Handling weight values as integers for comparison and processing
4. **String Representation**: Providing human-readable move notation

### Component Relationships

The `BookEntry` class interacts with the following components:

- **PolyglotTools**: Provides utility methods for byte-to-int conversions
- **FenTools**: Used for coordinate conversion to algebraic notation  
- **OpeningLibrary**: Manages collections of book entries
- **PolyglotOpeningBook**: Uses `BookEntry` objects to represent individual entries in opening books

### Data Flow

The data flows through the following sequence:
1. Raw byte array input is passed to the BookEntry constructor
2. Constructor parses the data into key (8 bytes), move (2 bytes), and weight (2 bytes)
3. Parsed components are stored internally
4. Move information is converted to readable algebraic notation
5. Final result is returned as a string representation of the move

## Detailed Component Analysis

### BookEntry Class

The `BookEntry` class is the primary component of this module. It implements `Comparable<BookEntry>` to enable sorting by weight (highest weight first).

#### Key Features:
- **Constructor**: Parses raw byte data into key, move, and weight components
- **Move Accessors**: Methods to extract from/to positions from compact move representation
- **Weight Handling**: Converts weight bytes to integer values for comparison
- **String Representation**: Returns move in algebraic notation via `toString()`

#### Data Structure:
Each `BookEntry` consists of three byte arrays:
1. `key` (8 bytes): Represents the board position using Zobrist hashing
2. `move` (2 bytes): Compact representation of the chess move
3. `weight` (2 bytes): Preference weight for this move

#### Move Decoding Logic:
The move decoding uses bit manipulation to extract file and rank information:
- From-file: Bits 6,7,8 of move[1] and bit 0 of move[0]
- From-row: Bits 3,4,5 of move[0]
- To-file: Bits 0,1,2 of move[1]
- To-row: Bits 3,4,5 of move[1]

## Integration Points

This module integrates with several other components in the system:

1. **[PolyglotOpeningBook](polyglot_opening_book.md)**: Uses `BookEntry` objects to represent individual entries in opening books
2. **[PolyglotTools](polyglot_tools.md)**: Provides utility methods for converting bytes to integers and coordinate representations
3. **[FenTools](fen_tools.md)**: Used for converting coordinates to algebraic notation
4. **[OpeningLibrary](opening.md)**: Manages collections of `BookEntry` objects

## Usage Patterns

### Typical Usage Flow:
1. Raw byte data is read from a Polyglot opening book file
2. `BookEntry` constructor parses the data into structured components
3. Move information is extracted and converted to readable notation
4. Entries are sorted by weight for optimal move selection
5. Selected entries are used to guide engine decisions during opening play

## Dependencies

This module depends on:
- `org.dokchess.opening.polyglot.PolyglotTools` for byte-to-int conversion
- `org.dokchess.opening.polyglot.FenTools` for coordinate conversion

## Related Modules

- [Polyglot Opening Book](polyglot_opening_book.md) - Manages collections of book entries
- [Polyglot Tools](polyglot_tools.md) - Utility functions for Polyglot format handling
- [FEN Tools](fen_tools.md) - Tools for FEN string manipulation
- [Opening Library](opening.md) - Higher-level opening book management

## Implementation Details

The implementation follows the Polyglot opening book specification where:
- Keys are 8-byte Zobrist hashes representing board positions
- Moves are encoded in a compact 2-byte format
- Weights are 2-byte unsigned integers indicating move preference
- Sorting is done by weight in descending order (higher weight = more preferred)

This design allows efficient storage and retrieval of opening knowledge while maintaining compatibility with standard Polyglot format specifications.