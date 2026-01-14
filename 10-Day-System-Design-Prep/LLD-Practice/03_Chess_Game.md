# LLD Problem 3: Chess Game Design

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **Inheritance & Polymorphism**: Different piece types with unique movements
- **Object Modeling**: Complex entity relationships
- **Command Pattern**: Undo/Redo functionality
- **Game State Management**: Turn-based logic

---

# R — Requirements

## Functional Requirements

```markdown
1. Game Setup
   - Standard 8×8 chess board
   - Proper piece placement
   - Two players (White/Black)

2. Piece Movement
   - Each piece follows chess rules
   - Validate legal moves
   - Capture mechanics
   - Special moves (castling, en passant, pawn promotion)

3. Game Flow
   - Turn-based alternation
   - Check detection
   - Checkmate/Stalemate detection
   - Move history

4. Undo/Redo
   - Undo last move
   - Redo undone move
   - Full move history
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Move validation | < 50ms | Instant feedback |
| Animation | 60 FPS | Smooth piece movement |
| Memory | < 50MB | Efficient for old devices |
| Offline | Full | Single device play |
| Testability | High | Complex game logic |

## iOS-Specific Requirements

```markdown
- Smooth drag-and-drop animations
- Haptic feedback on piece capture
- Highlight valid move squares
- Portrait/Landscape support
- Game save/restore
```

## Clarifying Questions

1. **Multiplayer**: Local only or online? → Local for now
2. **Timer**: Chess clocks needed? → Out of scope
3. **History**: Move notation display? → Yes, algebraic notation
4. **Variants**: Standard chess only? → Yes

---

# E — Entities

## Core Classes

```swift
// MARK: - Game Core

class ChessGame {
    let id: UUID
    private(set) var board: Board
    private(set) var players: (white: Player, black: Player)
    private(set) var currentTurn: PieceColor
    private(set) var gameState: GameState
    private(set) var moveHistory: [Move]
    
    private let commandManager: CommandManager
    
    init(whitePlayer: Player, blackPlayer: Player) {
        self.id = UUID()
        self.board = Board()
        self.players = (whitePlayer, blackPlayer)
        self.currentTurn = .white
        self.gameState = .active
        self.moveHistory = []
        self.commandManager = CommandManager()
        
        setupPieces()
    }
    
    func makeMove(from: Position, to: Position) throws {
        guard gameState == .active else {
            throw ChessError.gameNotActive
        }
        
        guard let piece = board.piece(at: from) else {
            throw ChessError.noPieceAtPosition
        }
        
        guard piece.color == currentTurn else {
            throw ChessError.notYourTurn
        }
        
        let move = try piece.createMove(from: from, to: to, board: board)
        
        // Execute via command pattern
        let command = MoveCommand(move: move, board: board)
        try commandManager.execute(command)
        
        // Update state
        moveHistory.append(move)
        switchTurn()
        checkGameState()
    }
    
    func undo() throws {
        try commandManager.undo()
        moveHistory.removeLast()
        switchTurn()
    }
    
    func redo() throws {
        guard let command = commandManager.redo() else {
            throw ChessError.noMoveToRedo
        }
        moveHistory.append(command.move)
        switchTurn()
    }
}

// MARK: - Board

class Board {
    private var squares: [[Square]]
    
    init() {
        squares = (0..<8).map { row in
            (0..<8).map { col in
                Square(position: Position(row: row, col: col))
            }
        }
    }
    
    func piece(at position: Position) -> Piece? {
        guard position.isValid else { return nil }
        return squares[position.row][position.col].piece
    }
    
    func setPiece(_ piece: Piece?, at position: Position) {
        guard position.isValid else { return }
        squares[position.row][position.col].piece = piece
    }
    
    func movePiece(from: Position, to: Position) -> Piece? {
        let piece = squares[from.row][from.col].piece
        let capturedPiece = squares[to.row][to.col].piece
        
        squares[to.row][to.col].piece = piece
        squares[from.row][from.col].piece = nil
        piece?.hasMoved = true
        
        return capturedPiece
    }
    
    func isPathClear(from: Position, to: Position) -> Bool {
        let direction = from.direction(to: to)
        var current = from.moved(by: direction)
        
        while current != to {
            if piece(at: current) != nil {
                return false
            }
            current = current.moved(by: direction)
        }
        
        return true
    }
}

// MARK: - Position

struct Position: Equatable, Hashable {
    let row: Int  // 0-7
    let col: Int  // 0-7 (a-h)
    
    var isValid: Bool {
        row >= 0 && row < 8 && col >= 0 && col < 8
    }
    
    var algebraic: String {
        let file = Character(UnicodeScalar(97 + col)!)  // a-h
        let rank = row + 1  // 1-8
        return "\(file)\(rank)"
    }
    
    func direction(to other: Position) -> (dRow: Int, dCol: Int) {
        let dRow = (other.row - row).signum()
        let dCol = (other.col - col).signum()
        return (dRow, dCol)
    }
    
    func moved(by direction: (dRow: Int, dCol: Int)) -> Position {
        Position(row: row + direction.dRow, col: col + direction.dCol)
    }
}

// MARK: - Square

class Square {
    let position: Position
    var piece: Piece?
    
    init(position: Position) {
        self.position = position
    }
    
    var color: SquareColor {
        (position.row + position.col) % 2 == 0 ? .dark : .light
    }
}

// MARK: - Player

struct Player {
    let id: String
    let name: String
    let color: PieceColor
}
```

## Piece Hierarchy (Polymorphism)

```swift
// MARK: - Abstract Piece

protocol Piece: AnyObject {
    var color: PieceColor { get }
    var type: PieceType { get }
    var position: Position { get set }
    var hasMoved: Bool { get set }
    var movementStrategy: MovementStrategy { get }
    
    func validMoves(on board: Board) -> [Position]
    func canMove(to position: Position, on board: Board) -> Bool
    func createMove(from: Position, to: Position, board: Board) throws -> Move
}

// MARK: - Concrete Pieces

class King: Piece {
    let color: PieceColor
    let type: PieceType = .king
    var position: Position
    var hasMoved: Bool = false
    lazy var movementStrategy: MovementStrategy = KingMovementStrategy()
    
    init(color: PieceColor, position: Position) {
        self.color = color
        self.position = position
    }
    
    func validMoves(on board: Board) -> [Position] {
        movementStrategy.validMoves(for: self, on: board)
    }
    
    func canMove(to position: Position, on board: Board) -> Bool {
        validMoves(on: board).contains(position)
    }
    
    func createMove(from: Position, to: Position, board: Board) throws -> Move {
        guard canMove(to: to, on: board) else {
            throw ChessError.invalidMove
        }
        
        // Check for castling
        if abs(to.col - from.col) == 2 {
            return CastlingMove(king: self, from: from, to: to, board: board)
        }
        
        let capturedPiece = board.piece(at: to)
        return StandardMove(piece: self, from: from, to: to, capturedPiece: capturedPiece)
    }
}

class Queen: Piece {
    let color: PieceColor
    let type: PieceType = .queen
    var position: Position
    var hasMoved: Bool = false
    lazy var movementStrategy: MovementStrategy = QueenMovementStrategy()
    
    // ... similar implementation
}

class Rook: Piece {
    let color: PieceColor
    let type: PieceType = .rook
    var position: Position
    var hasMoved: Bool = false
    lazy var movementStrategy: MovementStrategy = RookMovementStrategy()
    
    // ... similar implementation
}

class Bishop: Piece {
    let color: PieceColor
    let type: PieceType = .bishop
    var position: Position
    var hasMoved: Bool = false
    lazy var movementStrategy: MovementStrategy = BishopMovementStrategy()
    
    // ... similar implementation
}

class Knight: Piece {
    let color: PieceColor
    let type: PieceType = .knight
    var position: Position
    var hasMoved: Bool = false
    lazy var movementStrategy: MovementStrategy = KnightMovementStrategy()
    
    // ... similar implementation
}

class Pawn: Piece {
    let color: PieceColor
    let type: PieceType = .pawn
    var position: Position
    var hasMoved: Bool = false
    lazy var movementStrategy: MovementStrategy = PawnMovementStrategy()
    var enPassantVulnerable: Bool = false
    
    func createMove(from: Position, to: Position, board: Board) throws -> Move {
        guard canMove(to: to, on: board) else {
            throw ChessError.invalidMove
        }
        
        // Check for en passant
        if isEnPassantMove(from: from, to: to, board: board) {
            return EnPassantMove(pawn: self, from: from, to: to, board: board)
        }
        
        // Check for promotion
        if isPromotionMove(to: to) {
            return PromotionMove(pawn: self, from: from, to: to, board: board)
        }
        
        let capturedPiece = board.piece(at: to)
        return StandardMove(piece: self, from: from, to: to, capturedPiece: capturedPiece)
    }
}
```

## Enums

```swift
enum PieceColor: String {
    case white
    case black
    
    var opposite: PieceColor {
        self == .white ? .black : .white
    }
}

enum PieceType: String {
    case king, queen, rook, bishop, knight, pawn
    
    var symbol: String {
        switch self {
        case .king: return "♔"
        case .queen: return "♕"
        case .rook: return "♖"
        case .bishop: return "♗"
        case .knight: return "♘"
        case .pawn: return "♙"
        }
    }
}

enum GameState {
    case active
    case check(PieceColor)
    case checkmate(winner: PieceColor)
    case stalemate
    case draw
    case resigned(PieceColor)
}

enum SquareColor {
    case light
    case dark
}

enum ChessError: Error {
    case gameNotActive
    case noPieceAtPosition
    case notYourTurn
    case invalidMove
    case stillInCheck
    case noMoveToUndo
    case noMoveToRedo
}
```

## Entity Relationships

```
┌─────────────┐       ┌─────────────┐
│  ChessGame  │──────▶│    Board    │
└──────┬──────┘       └──────┬──────┘
       │                     │ 64
       │ 2                   ▼
       ▼              ┌─────────────┐       ┌─────────────┐
┌─────────────┐       │   Square    │──────▶│    Piece    │
│   Player    │       └─────────────┘  0..1 └──────┬──────┘
└─────────────┘                                    │
                                                   │ uses
       ┌─────────────┐                             ▼
       │  MoveHistory│◀────────────────────┌─────────────┐
       │   [Move]    │                     │  Movement   │
       └─────────────┘                     │  Strategy   │
                                           └─────────────┘
```

---

# S — States

## Game State Machine

```swift
enum GameState {
    case setup
    case active
    case check(PieceColor)  // King is in check
    case checkmate(winner: PieceColor)
    case stalemate
    case draw(DrawReason)
    case resigned(loser: PieceColor)
    
    func canTransition(to newState: GameState) -> Bool {
        switch (self, newState) {
        case (.setup, .active),
             (.active, .check),
             (.active, .checkmate),
             (.active, .stalemate),
             (.active, .draw),
             (.active, .resigned),
             (.check, .active),
             (.check, .checkmate),
             (.check, .stalemate):
            return true
        default:
            return false
        }
    }
}

enum DrawReason {
    case agreement
    case fiftyMoveRule
    case threefoldRepetition
    case insufficientMaterial
}
```

## State Diagram: Game

```
                        ┌─────────────┐
                        │    Setup    │
                        └──────┬──────┘
                               │ start game
                               ▼
    ┌─────────────────────────────────────────────────┐
    │                                                 │
    │  ┌──────────┐  move  ┌──────────┐              │
    │  │  Active  │◀──────▶│  Check   │              │
    │  └────┬─────┘        └────┬─────┘              │
    │       │                   │                    │
    │       │ no legal moves    │ no legal moves     │
    │       ▼                   ▼                    │
    │  ┌──────────┐        ┌──────────┐              │
    │  │Stalemate │        │Checkmate │              │
    │  └──────────┘        └──────────┘              │
    │                                                 │
    │       ┌──────────┐   ┌──────────┐              │
    │       │   Draw   │   │ Resigned │              │
    │       └──────────┘   └──────────┘              │
    └─────────────────────────────────────────────────┘
```

## Turn State

```swift
struct TurnState {
    var currentPlayer: PieceColor
    var moveNumber: Int
    var isCheck: Bool
    
    mutating func switchTurn() {
        currentPlayer = currentPlayer.opposite
        if currentPlayer == .white {
            moveNumber += 1
        }
    }
}
```

---

# H — Handling Concurrency

## Why Minimal Concurrency Needed

Chess is turn-based with atomic moves. However, we need concurrency for:

| Concern | Scenario | Solution |
|---------|----------|----------|
| AI moves | Computer thinking | Background task |
| Save game | Persist to disk | Async write |
| Animations | Move animation | Main thread |
| Move validation | Deep analysis | Can be async |

## AI Move Computation

```swift
actor ChessAI {
    private let difficulty: AIDifficulty
    private var isThinking = false
    
    enum AIDifficulty {
        case easy   // Random valid move
        case medium // 2-ply minimax
        case hard   // 4-ply with alpha-beta
    }
    
    func computeBestMove(for board: Board, color: PieceColor) async -> Move? {
        isThinking = true
        defer { isThinking = false }
        
        return await Task.detached(priority: .userInitiated) {
            switch self.difficulty {
            case .easy:
                return self.randomMove(board: board, color: color)
            case .medium:
                return self.minimax(board: board, color: color, depth: 2)
            case .hard:
                return self.alphaBeta(board: board, color: color, depth: 4)
            }
        }.value
    }
    
    func cancelThinking() {
        // Cancel current computation
    }
}
```

## Game State Protected by MainActor

```swift
@MainActor
class ChessGameViewModel: ObservableObject {
    @Published private(set) var board: BoardViewModel
    @Published private(set) var currentTurn: PieceColor
    @Published private(set) var gameState: GameState
    @Published private(set) var moveHistory: [MoveViewModel]
    @Published private(set) var highlightedSquares: Set<Position>
    
    private let game: ChessGame
    
    func selectPiece(at position: Position) {
        guard let piece = game.board.piece(at: position),
              piece.color == currentTurn else {
            return
        }
        
        highlightedSquares = Set(piece.validMoves(on: game.board))
    }
    
    func makeMove(to position: Position) async {
        guard let selectedPiece = selectedPiece else { return }
        
        do {
            try game.makeMove(from: selectedPiece.position, to: position)
            updateUI()
        } catch {
            showError(error)
        }
    }
}
```

---

# A — Architecture & Patterns

## Pattern 1: Factory (Piece Creation)

### What it does:
Creates different piece types at game initialization.

### Implementation:
```swift
protocol PieceFactory {
    func createPiece(type: PieceType, color: PieceColor, position: Position) -> Piece
}

class StandardPieceFactory: PieceFactory {
    func createPiece(type: PieceType, color: PieceColor, position: Position) -> Piece {
        switch type {
        case .king:
            return King(color: color, position: position)
        case .queen:
            return Queen(color: color, position: position)
        case .rook:
            return Rook(color: color, position: position)
        case .bishop:
            return Bishop(color: color, position: position)
        case .knight:
            return Knight(color: color, position: position)
        case .pawn:
            return Pawn(color: color, position: position)
        }
    }
}

// Usage in Board setup
extension Board {
    func setupStandardGame(factory: PieceFactory = StandardPieceFactory()) {
        // Back row
        let backRowTypes: [PieceType] = [.rook, .knight, .bishop, .queen, .king, .bishop, .knight, .rook]
        
        for (col, type) in backRowTypes.enumerated() {
            let whitePiece = factory.createPiece(type: type, color: .white, position: Position(row: 0, col: col))
            let blackPiece = factory.createPiece(type: type, color: .black, position: Position(row: 7, col: col))
            
            setPiece(whitePiece, at: whitePiece.position)
            setPiece(blackPiece, at: blackPiece.position)
        }
        
        // Pawns
        for col in 0..<8 {
            let whitePawn = factory.createPiece(type: .pawn, color: .white, position: Position(row: 1, col: col))
            let blackPawn = factory.createPiece(type: .pawn, color: .black, position: Position(row: 6, col: col))
            
            setPiece(whitePawn, at: whitePawn.position)
            setPiece(blackPawn, at: blackPawn.position)
        }
    }
}
```

### Why chosen:
1. **Centralizes** piece creation logic
2. **Enables** custom piece sets (themed chess)
3. **Testable** - can inject mock factory

---

## Pattern 2: Strategy (Movement Rules)

### What it does:
Encapsulates movement logic for each piece type, making it swappable.

### Implementation:
```swift
protocol MovementStrategy {
    func validMoves(for piece: Piece, on board: Board) -> [Position]
}

// King moves one square in any direction
class KingMovementStrategy: MovementStrategy {
    func validMoves(for piece: Piece, on board: Board) -> [Position] {
        let directions = [
            (-1, -1), (-1, 0), (-1, 1),
            (0, -1),           (0, 1),
            (1, -1),  (1, 0),  (1, 1)
        ]
        
        var validPositions: [Position] = []
        
        for (dRow, dCol) in directions {
            let newPos = Position(row: piece.position.row + dRow, col: piece.position.col + dCol)
            
            if newPos.isValid && canOccupy(newPos, piece: piece, board: board) {
                validPositions.append(newPos)
            }
        }
        
        // Add castling moves
        validPositions.append(contentsOf: castlingMoves(for: piece, on: board))
        
        return validPositions
    }
}

// Knight moves in L-shape
class KnightMovementStrategy: MovementStrategy {
    func validMoves(for piece: Piece, on board: Board) -> [Position] {
        let moves = [
            (-2, -1), (-2, 1), (-1, -2), (-1, 2),
            (1, -2), (1, 2), (2, -1), (2, 1)
        ]
        
        return moves.compactMap { (dRow, dCol) in
            let newPos = Position(row: piece.position.row + dRow, col: piece.position.col + dCol)
            return newPos.isValid && canOccupy(newPos, piece: piece, board: board) ? newPos : nil
        }
    }
}

// Rook moves horizontally/vertically
class RookMovementStrategy: MovementStrategy {
    func validMoves(for piece: Piece, on board: Board) -> [Position] {
        let directions = [(0, 1), (0, -1), (1, 0), (-1, 0)]
        return slidingMoves(from: piece.position, directions: directions, piece: piece, board: board)
    }
}

// Bishop moves diagonally
class BishopMovementStrategy: MovementStrategy {
    func validMoves(for piece: Piece, on board: Board) -> [Position] {
        let directions = [(1, 1), (1, -1), (-1, 1), (-1, -1)]
        return slidingMoves(from: piece.position, directions: directions, piece: piece, board: board)
    }
}

// Queen combines Rook + Bishop
class QueenMovementStrategy: MovementStrategy {
    private let rookStrategy = RookMovementStrategy()
    private let bishopStrategy = BishopMovementStrategy()
    
    func validMoves(for piece: Piece, on board: Board) -> [Position] {
        rookStrategy.validMoves(for: piece, on: board) +
        bishopStrategy.validMoves(for: piece, on: board)
    }
}

// Pawn has complex rules
class PawnMovementStrategy: MovementStrategy {
    func validMoves(for piece: Piece, on board: Board) -> [Position] {
        guard let pawn = piece as? Pawn else { return [] }
        
        let direction = pawn.color == .white ? 1 : -1
        let startRow = pawn.color == .white ? 1 : 6
        var validPositions: [Position] = []
        
        // Forward move
        let oneForward = Position(row: pawn.position.row + direction, col: pawn.position.col)
        if oneForward.isValid && board.piece(at: oneForward) == nil {
            validPositions.append(oneForward)
            
            // Two-square first move
            if pawn.position.row == startRow {
                let twoForward = Position(row: pawn.position.row + 2 * direction, col: pawn.position.col)
                if board.piece(at: twoForward) == nil {
                    validPositions.append(twoForward)
                }
            }
        }
        
        // Diagonal captures
        for dCol in [-1, 1] {
            let capturePos = Position(row: pawn.position.row + direction, col: pawn.position.col + dCol)
            if let targetPiece = board.piece(at: capturePos), targetPiece.color != pawn.color {
                validPositions.append(capturePos)
            }
        }
        
        // En passant
        validPositions.append(contentsOf: enPassantMoves(for: pawn, on: board))
        
        return validPositions
    }
}
```

### Why chosen:
1. **Separates** movement logic from piece class
2. **Easy to modify** rules per piece
3. **Testable** in isolation

---

## Pattern 3: Command (Undo/Redo)

### What it does:
Encapsulates moves as objects for undo/redo.

### Implementation:
```swift
protocol Command {
    func execute() throws
    func undo()
}

// Standard move command
class MoveCommand: Command {
    let move: Move
    private let board: Board
    private var capturedPiece: Piece?
    private var previousHasMoved: Bool = false
    
    init(move: Move, board: Board) {
        self.move = move
        self.board = board
    }
    
    func execute() throws {
        capturedPiece = board.piece(at: move.to)
        previousHasMoved = move.piece.hasMoved
        
        board.movePiece(from: move.from, to: move.to)
        move.piece.hasMoved = true
        
        // Handle special moves
        move.executeSpecialLogic(on: board)
    }
    
    func undo() {
        board.movePiece(from: move.to, to: move.from)
        move.piece.hasMoved = previousHasMoved
        
        if let captured = capturedPiece {
            board.setPiece(captured, at: move.to)
        }
        
        move.undoSpecialLogic(on: board)
    }
}

// Command Manager
class CommandManager {
    private var executedCommands: [Command] = []
    private var undoneCommands: [Command] = []
    
    func execute(_ command: Command) throws {
        try command.execute()
        executedCommands.append(command)
        undoneCommands.removeAll()  // Clear redo stack
    }
    
    func undo() throws {
        guard let command = executedCommands.popLast() else {
            throw ChessError.noMoveToUndo
        }
        command.undo()
        undoneCommands.append(command)
    }
    
    func redo() -> Command? {
        guard let command = undoneCommands.popLast() else {
            return nil
        }
        try? command.execute()
        executedCommands.append(command)
        return command
    }
    
    var canUndo: Bool { !executedCommands.isEmpty }
    var canRedo: Bool { !undoneCommands.isEmpty }
}
```

### Why chosen:
1. **Encapsulates** move as object
2. **Full undo/redo** support
3. **Move history** for replay

---

## ❌ Why Observer is NOT Ideal for Moves

### The Problem:

```swift
// Observer pattern for moves
protocol MoveObserver {
    func didMakeMove(_ move: Move)
}

class ChessGame {
    var observers: [MoveObserver] = []
    
    func makeMove(_ move: Move) {
        // Execute move
        observers.forEach { $0.didMakeMove(move) }
    }
}
```

### Why It Doesn't Fit:

| Issue | Explanation |
|-------|-------------|
| **Overkill** | Only one UI needs updates |
| **Ordering problems** | Observers don't know order |
| **State sync** | Hard to keep observers in sync |
| **Memory leaks** | Observer management overhead |

### Better Alternative: Direct Binding

```swift
@MainActor
class ChessGameViewModel: ObservableObject {
    @Published private(set) var boardState: BoardState
    
    func makeMove(_ move: Move) {
        // Execute move
        boardState = game.currentBoardState()  // Direct update
    }
}

// SwiftUI automatically updates
struct BoardView: View {
    @ObservedObject var viewModel: ChessGameViewModel
    
    var body: some View {
        // Redraws when boardState changes
    }
}
```

### When Observer WOULD Be Used:
- Multiple independent UIs (main board + mini preview)
- Network multiplayer with multiple clients
- Analytics/logging systems

---

# D — Data Flow

## Sequence Diagram: Move → Validation → State Update

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│  User  │ │  ViewModel │ │ ChessGame  │ │   Piece    │ │   Board    │
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │              │
    │ tap square │              │              │              │
    │───────────▶│              │              │              │
    │            │              │              │              │
    │            │ piece at?    │              │              │
    │            │──────────────────────────────────────────▶│
    │            │              │              │              │
    │            │◀──────────────────────────────────────────│ Piece
    │            │              │              │              │
    │            │ validMoves() │              │              │
    │            │─────────────────────────────▶              │
    │            │              │              │              │
    │            │              │              │ canOccupy?   │
    │            │              │              │─────────────▶│
    │            │              │              │              │
    │            │◀─────────────────────────────[Position]    │
    │            │              │              │              │
    │ highlight  │              │              │              │
    │◀───────────│              │              │              │
    │            │              │              │              │
    │ tap dest   │              │              │              │
    │───────────▶│              │              │              │
    │            │              │              │              │
    │            │ makeMove(from, to)          │              │
    │            │─────────────▶│              │              │
    │            │              │              │              │
    │            │              │ createMove() │              │
    │            │              │─────────────▶│              │
    │            │              │              │              │
    │            │              │◀─────────────│ Move         │
    │            │              │              │              │
    │            │              │ execute(command)            │
    │            │              │──────────────────────────────▶
    │            │              │              │              │
    │            │              │ checkGameState()            │
    │            │              │──┐           │              │
    │            │              │◀─┘           │              │
    │            │              │              │              │
    │            │◀─────────────│ BoardState   │              │
    │            │              │              │              │
    │ animate    │              │              │              │
    │◀───────────│              │              │              │
    │            │              │              │              │
```

## Step-by-Step Flow

### Flow: Standard Move

**Trigger:** User taps destination square

**Steps:**
1. ViewModel receives tap at destination position
2. Get piece at selected position from board
3. Validate piece belongs to current player
4. Create Move object with from/to positions
5. Wrap in MoveCommand for undo support
6. Execute command (updates board state)
7. Check for check/checkmate/stalemate
8. Switch turn
9. Publish new board state to UI
10. Animate piece movement

### Flow: Undo Move

**Trigger:** User taps undo button

**Steps:**
1. ViewModel calls game.undo()
2. CommandManager pops last command
3. Command.undo() reverses board changes
4. Restore captured piece if any
5. Switch turn back
6. Update UI with previous state

---

# E — Edge Cases

## Edge Case 1: Move Puts Own King in Check

**Scenario:** User moves piece that exposes their king

**Handling:**
```swift
func createMove(from: Position, to: Position, board: Board) throws -> Move {
    // Create tentative move
    let move = StandardMove(piece: self, from: from, to: to)
    
    // Simulate the move
    let simulatedBoard = board.copy()
    simulatedBoard.applyMove(move)
    
    // Check if own king is now in check
    if simulatedBoard.isKingInCheck(color: self.color) {
        throw ChessError.stillInCheck
    }
    
    return move
}
```

## Edge Case 2: Pawn Promotion

**Scenario:** Pawn reaches opposite end of board

**Handling:**
```swift
class PromotionMove: Move {
    let pawn: Pawn
    var promoteTo: PieceType = .queen  // Default
    
    func executeSpecialLogic(on board: Board) {
        // Remove pawn
        board.setPiece(nil, at: to)
        
        // Create promoted piece
        let promotedPiece = PieceFactory.create(
            type: promoteTo,
            color: pawn.color,
            position: to
        )
        board.setPiece(promotedPiece, at: to)
    }
}

// ViewModel asks user for choice
func handlePromotion(move: PromotionMove) async {
    let choice = await showPromotionPicker()  // Queen, Rook, Bishop, Knight
    move.promoteTo = choice
    try await executeMove(move)
}
```

## Edge Case 3: Castling Blocked

**Scenario:** King or rook has moved, or squares attacked

**Handling:**
```swift
func castlingMoves(for king: King, on board: Board) -> [Position] {
    guard !king.hasMoved else { return [] }
    
    var moves: [Position] = []
    
    // Kingside castling
    if let rook = board.piece(at: Position(row: king.position.row, col: 7)) as? Rook,
       !rook.hasMoved,
       board.isPathClear(from: king.position, to: Position(row: king.position.row, col: 7)),
       !isAnySquareAttacked(from: king.position, to: Position(row: king.position.row, col: 6), board: board) {
        moves.append(Position(row: king.position.row, col: 6))
    }
    
    // Queenside castling (similar)
    
    return moves
}
```

## Edge Case 4: Stalemate Detection

**Scenario:** Player has no legal moves but not in check

**Handling:**
```swift
func checkGameState() {
    let currentPlayerPieces = board.pieces(of: currentTurn)
    
    // Check if any piece has legal moves
    let hasLegalMove = currentPlayerPieces.contains { piece in
        !piece.validMoves(on: board).isEmpty
    }
    
    if !hasLegalMove {
        if board.isKingInCheck(color: currentTurn) {
            gameState = .checkmate(winner: currentTurn.opposite)
        } else {
            gameState = .stalemate
        }
    }
}
```

## Edge Case Checklist

```markdown
□ Move exposes own king to check
□ Castling through attacked squares
□ En passant timing (only after pawn double move)
□ Pawn promotion choice
□ Stalemate vs checkmate distinction
□ Three-fold repetition detection
□ Fifty-move rule
□ Insufficient material draw
□ App kill during game (save state)
□ Undo after checkmate
```

---

# D — Design Trade-offs

## Trade-off 1: Piece Hierarchy vs Composition

| Inheritance | Composition |
|-------------|-------------|
| Natural modeling | More flexible |
| Easy polymorphism | Harder to understand |
| Type-safe | Runtime configuration |

**Decision:** Inheritance with Strategy composition

**Rationale:** Pieces have IS-A relationship (King IS-A Piece), but movement rules are composed via Strategy pattern for flexibility.

## Trade-off 2: Board Copy for Validation

| Copy Board | In-Place + Rollback |
|------------|---------------------|
| Simple logic | Complex state management |
| Memory overhead | Memory efficient |
| Thread-safe | Needs careful handling |

**Decision:** Copy board for move validation

**Rationale:** Simplicity outweighs ~50KB memory overhead. Makes code much easier to reason about.

## Trade-off 3: UI Updates

| Full Board Redraw | Incremental Updates |
|-------------------|---------------------|
| Simple | Complex change tracking |
| Slower on old devices | More efficient |
| Always correct | Potential sync issues |

**Decision:** Full board redraw with SwiftUI

**Rationale:** SwiftUI efficiently diffs views. 64-square board is trivial to render.

---

# iOS Implementation

## Complete ViewModel

```swift
@MainActor
class ChessGameViewModel: ObservableObject {
    // State
    @Published private(set) var squares: [[SquareViewModel]] = []
    @Published private(set) var currentTurn: PieceColor = .white
    @Published private(set) var gameState: GameState = .active
    @Published private(set) var moveHistory: [String] = []
    @Published private(set) var highlightedMoves: Set<Position> = []
    @Published private(set) var selectedPosition: Position?
    
    @Published var canUndo: Bool = false
    @Published var canRedo: Bool = false
    
    private let game: ChessGame
    
    init(game: ChessGame = ChessGame.standardGame()) {
        self.game = game
        updateBoardState()
    }
    
    // MARK: - User Actions
    
    func tapSquare(at position: Position) {
        if let selected = selectedPosition {
            // Try to move
            if highlightedMoves.contains(position) {
                makeMove(from: selected, to: position)
            }
            deselectPiece()
        } else {
            // Try to select
            selectPiece(at: position)
        }
    }
    
    func selectPiece(at position: Position) {
        guard let piece = game.board.piece(at: position),
              piece.color == currentTurn else {
            return
        }
        
        selectedPosition = position
        highlightedMoves = Set(piece.validMoves(on: game.board))
    }
    
    func deselectPiece() {
        selectedPosition = nil
        highlightedMoves = []
    }
    
    func makeMove(from: Position, to: Position) {
        do {
            try game.makeMove(from: from, to: to)
            updateBoardState()
            
            // Haptic for capture
            if game.moveHistory.last?.capturedPiece != nil {
                UIImpactFeedbackGenerator(style: .medium).impactOccurred()
            }
        } catch {
            UINotificationFeedbackGenerator().notificationOccurred(.error)
        }
    }
    
    func undo() {
        try? game.undo()
        updateBoardState()
    }
    
    func redo() {
        try? game.redo()
        updateBoardState()
    }
    
    // MARK: - State Updates
    
    private func updateBoardState() {
        // Map board to view models
        squares = (0..<8).map { row in
            (0..<8).map { col in
                let pos = Position(row: row, col: col)
                return SquareViewModel(
                    position: pos,
                    piece: game.board.piece(at: pos).map { PieceViewModel(from: $0) }
                )
            }
        }
        
        currentTurn = game.currentTurn
        gameState = game.gameState
        canUndo = game.commandManager.canUndo
        canRedo = game.commandManager.canRedo
        
        moveHistory = game.moveHistory.map { $0.algebraicNotation }
    }
}
```

## SwiftUI Board View

```swift
struct ChessBoardView: View {
    @ObservedObject var viewModel: ChessGameViewModel
    
    var body: some View {
        VStack(spacing: 0) {
            ForEach((0..<8).reversed(), id: \.self) { row in
                HStack(spacing: 0) {
                    ForEach(0..<8, id: \.self) { col in
                        let position = Position(row: row, col: col)
                        SquareView(
                            square: viewModel.squares[row][col],
                            isSelected: viewModel.selectedPosition == position,
                            isHighlighted: viewModel.highlightedMoves.contains(position)
                        )
                        .onTapGesture {
                            viewModel.tapSquare(at: position)
                        }
                    }
                }
            }
        }
        .aspectRatio(1, contentMode: .fit)
    }
}

struct SquareView: View {
    let square: SquareViewModel
    let isSelected: Bool
    let isHighlighted: Bool
    
    var body: some View {
        ZStack {
            // Square background
            Rectangle()
                .fill(squareColor)
            
            // Highlight indicator
            if isHighlighted {
                Circle()
                    .fill(Color.green.opacity(0.5))
                    .padding(12)
            }
            
            // Selection indicator
            if isSelected {
                Rectangle()
                    .stroke(Color.yellow, lineWidth: 3)
            }
            
            // Piece
            if let piece = square.piece {
                Text(piece.symbol)
                    .font(.system(size: 40))
                    .foregroundColor(piece.color == .white ? .white : .black)
            }
        }
    }
    
    var squareColor: Color {
        square.isLight ? Color(.systemBrown).opacity(0.3) : Color(.systemBrown)
    }
}
```

---

# Interview Tips for This Problem

## What to Say

```markdown
1. "Chess naturally maps to inheritance - each piece IS-A Piece..."
   - Show understanding of OOP principles

2. "I'll separate movement rules using Strategy pattern..."
   - Each piece delegates movement logic

3. "For undo/redo, Command pattern is perfect because..."
   - Encapsulates moves as reversible objects

4. "Observer isn't needed here because..."
   - Proactively explain what you're NOT using

5. "The tricky edge cases are castling conditions and en passant..."
   - Show deep knowledge of chess rules
```

## Red Flags to Avoid

```markdown
❌ "I'll use a giant switch statement for all piece moves"
   → Misses Strategy pattern opportunity

❌ "Observer pattern to notify UI of moves"
   → Overcomplicating for single-UI scenario

❌ "I'll store moves as just from/to positions"
   → Misses Command pattern, can't undo properly

❌ Forgetting special moves (castling, en passant, promotion)
   → Incomplete domain understanding
```

---

*This is how an Amazon iOS LLD interview expects you to approach the Chess Game problem!*
