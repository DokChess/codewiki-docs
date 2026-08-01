# Engine Module Architecture

## Complete System Architecture Diagram

```mermaid
graph LR
    subgraph "Engine Module"
        A[Engine Interface] --> B(DefaultEngine)
        B --> C[DetermineMove Chain]
        C --> D{Opening Library?}
        D -->|Yes| E[FromLibrary]
        D -->|No| F[FromSearch]
        E --> G[Search Strategy]
        F --> G
        G --> H[Minimax Algorithm]
        H --> I[Evaluation Function]
    end
    
    subgraph "External Dependencies"
        J[ChessRules] --> H
        K[OpeningLibrary] --> E
        L[Position] --> A
        M[Move] --> A
    end
    
    subgraph "Data Flow"
        L --> B
        B --> M
        M --> B
        H --> I
        I --> H
    end
```

## Component Interactions

### Chain of Responsibility Pattern
The engine uses a chain-of-responsibility pattern implemented through the `DetermineMove` abstract class:

```mermaid
sequenceDiagram
    participant Engine
    participant FromLibrary
    participant FromSearch
    participant Search
    participant Evaluation
    
    Engine->>FromLibrary: determineMove()
    FromLibrary->>OpeningLibrary: lookUpMove()
    FromLibrary-->>Engine: Found move or delegate
    FromSearch->>Search: searchMove()
    Search->>MinimaxAlgorithm: evaluatePosition()
    MinimaxAlgorithm->>Evaluation: evaluatePosition()
    Evaluation-->>MinimaxAlgorithm: score
    MinimaxAlgorithm-->>Search: best move
    Search-->>Engine: best move
```

## Data Flow Through Engine

1. **Setup Phase**: 
   - `setupPieces()` sets the current game position
   - Cancels any ongoing searches

2. **Move Determination Phase**:
   - `determineYourMove()` initiates asynchronous move search
   - Chain evaluates opening library first (if available)
   - Falls back to search algorithm if no opening move found
   - Search algorithm evaluates positions using minimax
   - Evaluation function scores positions
   - Best move is reported through RxJava Observable

3. **Move Execution Phase**:
   - `performMove()` applies move to internal position
   - Cancels ongoing searches

4. **Cleanup Phase**:
   - `close()` method shuts down resources
   - Cancels any pending searches