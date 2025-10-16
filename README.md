import SwiftUI

// MARK: - Game State Enum
// Added Equatable explicitly to help compiler resolve comparison errors
enum GameState: Equatable {
    case waitingToStart
    case active
    case checkmate(winnerIsWhite: Bool)
    case stalemate
    
    var statusText: String {
        switch self {
        case .waitingToStart: return "Press Start Game"
        case .active: return "In Play"
        case .checkmate(let isWhite): return isWhite ? "White Wins by Checkmate!" : "Black Wins by Checkmate!"
        case .stalemate: return "Game Over: Stalemate (Draw)"
        }
    }
}

// MARK: - Time Control Model
enum TimeControl: String, CaseIterable, Identifiable {
    case threeMin = "3 min"
    case fiveMin = "5 min"
    case tenMin = "10 min"
    case infinite = "∞ (Normal)"
    
    var id: String { self.rawValue }
    
    var duration: TimeInterval? {
        switch self {
        case .threeMin: return 3 * 60
        case .fiveMin: return 5 * 60
        case .tenMin: return 10 * 60
        case .infinite: return nil
        }
    }
}

// MARK: - Castling Helper Enum
enum CastleType {
    case kingSide
    case queenSide
}

// MARK: - Piece Models
struct ChessPiece: Identifiable, Equatable, Hashable {
    let id = UUID()
    let type: String
    let isWhite: Bool
    let symbol: String
    
    static func getSymbol(type: String, isWhite: Bool) -> String {
        switch type {
        case "king": return isWhite ? "♔" : "♚"
        case "queen": return isWhite ? "♕" : "♛"
        case "rook": return isWhite ? "♖" : "♜"
        case "bishop": return isWhite ? "♗" : "♝"
        case "knight": return isWhite ? "♘" : "♞"
        case "pawn": return isWhite ? "♙" : "♟︎"
        default: return ""
        }
    }
}

struct ChessSquare: Identifiable, Hashable {
    let id = UUID()
    let row: Int
    let col: Int
    var piece: ChessPiece?
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(row)
        hasher.combine(col)
        hasher.combine(piece) 
    }
    
    static func == (lhs: ChessSquare, rhs: ChessSquare) -> Bool {
        return lhs.row == rhs.row && lhs.col == rhs.col && lhs.piece == rhs.piece
    }
}

// MARK: - Chess Clock
class ChessClock: ObservableObject {
    @Published var player1Time: TimeInterval
    @Published var player2Time: TimeInterval
    @Published var isPlayer1Active = true
    @Published var selectedTimeControl: TimeControl = .fiveMin
    
    var timer: Timer? = nil
    
    init(initialTime: TimeInterval) {
        self.player1Time = initialTime
        self.player2Time = initialTime
    }
    
    func start(game: ChessGame) {
        // Explicitly reference game.gameState
        guard game.gameState == .waitingToStart else { return }
        
        game.gameState = .active
        
        // Only start timer if not in infinite mode
        guard selectedTimeControl != .infinite else { return }
        
        timer?.invalidate()
        timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
            guard let self = self else { return }
            
            // Stop if time runs out
            if self.player1Time <= 0 || self.player2Time <= 0 {
                self.stop()
                // The active player loses
                game.gameState = .checkmate(winnerIsWhite: !self.isPlayer1Active) 
                return
            }
            
            if self.player1Time > 0 && self.isPlayer1Active {
                self.player1Time -= 1
            } else if self.player2Time > 0 && !self.isPlayer1Active {
                self.player2Time -= 1
            }
        }
    }
    
    func switchPlayer() {
        isPlayer1Active.toggle()
    }
    
    func stop() {
        timer?.invalidate()
    }
    
    func reset() {
        if let duration = selectedTimeControl.duration {
            player1Time = duration
            player2Time = duration
        } else {
            // Set a large dummy time for infinite mode
            player1Time = 999999
            player2Time = 999999
        }
        isPlayer1Active = true
        stop()
    }
}

// MARK: - Game ViewModel and Logic
class ChessGame: ObservableObject {
    @Published var board: [[ChessSquare]] = []
    @Published var isWhiteTurn: Bool = true
    @Published var lastMoveFrom: (Int, Int)? = nil
    @Published var lastMoveTo: (Int, Int)? = nil
    @Published var gameState: GameState = .waitingToStart // Game state control
    
    // MARK: - Castling Rights
    @Published var hasWhiteKingMoved: Bool = false
    @Published var hasWhiteKingSideRookMoved: Bool = false
    @Published var hasWhiteQueenSideRookMoved: Bool = false
    @Published var hasBlackKingMoved: Bool = false
    @Published var hasBlackKingSideRookMoved: Bool = false
    @Published var hasBlackQueenSideRookMoved: Bool = false
    
    // MARK: - Promotion State Variables
    @Published var isPromoting: Bool = false
    @Published var promotionLocation: (Int, Int)? = nil
    
    init() {
        setupBoard()
    }
    
    // MARK: Board Setup
    func createPiece(type: String, isWhite: Bool) -> ChessPiece {
        return ChessPiece(type: type, isWhite: isWhite, symbol: ChessPiece.getSymbol(type: type, isWhite: isWhite))
    }
    
    func setupBoard() {
        board = []
        for row in 0..<8 {
            var rowArray: [ChessSquare] = []
            for col in 0..<8 {
                rowArray.append(ChessSquare(row: row, col: col, piece: nil))
            }
            board.append(rowArray)
        }
        
        let pieceTypes: [String] = ["rook", "knight", "bishop", "queen", "king", "bishop", "knight", "rook"]
        
        for col in 0..<8 {
            // White pieces (Row 0)
            board[0][col].piece = createPiece(type: pieceTypes[col], isWhite: true)
            // White Pawns (Row 1)
            board[1][col].piece = createPiece(type: "pawn", isWhite: true)
            
            // Black Pawns (Row 6)
            board[6][col].piece = createPiece(type: "pawn", isWhite: false)
            // Black pieces (Row 7)
            board[7][col].piece = createPiece(type: pieceTypes[col], isWhite: false)
        }
        
        // Reset castling rights
        hasWhiteKingMoved = false
        hasWhiteKingSideRookMoved = false
        hasWhiteQueenSideRookMoved = false
        hasBlackKingMoved = false
        hasBlackKingSideRookMoved = false
        hasBlackQueenSideRookMoved = false
        
        // Reset last move state
        lastMoveFrom = nil
        lastMoveTo = nil
        
        gameState = .waitingToStart // Ensure game is paused on reset
    }
    
    // MARK: Check Logic
    
    // Finds the current position of the King of the specified color
    func findKing(for color: Bool) -> (Int, Int)? {
        for r in 0..<8 {
            for c in 0..<8 {
                if board[r][c].piece?.type == "king" && board[r][c].piece?.isWhite == color {
                    return (r, c)
                }
            }
        }
        return nil
    }
    
    // Checks if a King of a specific color is currently attacked
    func isInCheck(for kingColor: Bool) -> Bool {
        guard let kingPos = self.findKing(for: kingColor) else { return false }
        
        // This leverages the `isSquareAttacked` helper to check the King's position.
        return self.isSquareAttacked(at: kingPos, by: kingColor)
    }
    
    // Checks if a specific square is under attack by the opponent of `color`
    private func isSquareAttacked(at pos: (Int, Int), by color: Bool) -> Bool {
        let opponentColor = !color
        
        for r in 0..<8 {
            for c in 0..<8 {
                if let piece = board[r][c].piece, piece.isWhite == opponentColor {
                    // Use pseudo-legal check to see if opponent can attack the square
                    if self.isMovePseudoLegal(from: (r, c), to: pos) {
                        return true
                    }
                }
            }
        }
        return false
    }
    
    // MARK: Core Legality Checks
    
    // Checks if the path between two squares is clear (for Rook, Bishop, Queen)
    private func isPathClear(from: (Int, Int), to: (Int, Int)) -> Bool {
        let (r1, c1) = from
        let (r2, c2) = to
        
        let dRow = r2 - r1
        let dCol = c2 - c1
        
        if dRow == 0 {
            let step = dCol > 0 ? 1 : -1
            for col in stride(from: c1 + step, to: c2, by: step) {
                if board[r1][col].piece != nil { return false }
            }
        } else if dCol == 0 {
            let step = dRow > 0 ? 1 : -1
            for row in stride(from: r1 + step, to: r2, by: step) {
                if board[row][c1].piece != nil { return false }
            }
        } else if abs(dRow) == abs(dCol) {
            let rStep = dRow > 0 ? 1 : -1
            let cStep = dCol > 0 ? 1 : -1
            
            var r = r1 + rStep
            var c = c1 + cStep
            
            while r != r2 {
                if board[r][c].piece != nil { return false }
                r += rStep
                c += cStep
            }
        }
        return true
    }
    
    // Checks if a move is physically legal according to the piece's rules (Pseudo-Legal).
    func isMovePseudoLegal(from: (Int, Int), to: (Int, Int)) -> Bool {
        let (r1, c1) = from
        let (r2, c2) = to
        
        // Bounds check and self-move check
        guard r1 >= 0, r1 < 8, c1 >= 0, c1 < 8,
              r2 >= 0, r2 < 8, c2 >= 0, c2 < 8,
              r1 != r2 || c1 != c2 else { return false }
        
        guard let piece = board[r1][c1].piece else { return false }
        let targetPiece = board[r2][c2].piece
        
        // Cannot capture your own piece
        if targetPiece?.isWhite == piece.isWhite { return false }
        
        let dRow = r2 - r1
        let dCol = c2 - c1
        
        switch piece.type {
        case "pawn":
            let direction: Int = piece.isWhite ? 1 : -1
            let startRow: Int = piece.isWhite ? 1 : 6
            
            // 1. Forward move (no capture)
            if dCol == 0 {
                // One step forward
                if dRow == direction { return targetPiece == nil }
                // Two steps forward (only from start row)
                if dRow == 2 * direction && r1 == startRow {
                    return targetPiece == nil && self.isPathClear(from: from, to: to)
                }
            }
            // 2. Diagonal capture/attack
            if abs(dCol) == 1 && dRow == direction { 
                // A pawn can only move diagonally if there is an opponent piece to capture.
                // This ensures the diagonal move is only pseudo-legal if there is a piece there.
                if targetPiece != nil {
                    return true // Diagonal capture is legal
                }
                
                // If there's no piece at the target, the move is invalid (ignoring en passant for now)
                return false
            } 
            return false
            
        case "rook":
            guard (dRow == 0 && dCol != 0) || (dRow != 0 && dCol == 0) else { return false }
            return self.isPathClear(from: from, to: to)
            
        case "bishop":
            guard abs(dRow) == abs(dCol) else { return false }
            return self.isPathClear(from: from, to: to)
            
        case "queen":
            guard (dRow == 0 && dCol != 0) || (dRow != 0 && dCol == 0) || (abs(dRow) == abs(dCol)) else { return false }
            return self.isPathClear(from: from, to: to)
            
        case "king":
            // Standard King move
            if abs(dRow) <= 1 && abs(dCol) <= 1 { return true }
            
            // Castling attempt
            let rank = piece.isWhite ? 0 : 7
            let kingStartCol = 4
            
            // Castling is handled by `isMoveLegal` with a separate `canCastle` check, 
            // but we must allow the 2-step move here for pseudo-legality.
            if r1 == rank && c1 == kingStartCol && r2 == rank && abs(dCol) == 2 {
                return true
            }
            return false 
            
        case "knight":
            let dx = abs(dRow)
            let dy = abs(dCol)
            return (dx == 1 && dy == 2) || (dx == 2 && dy == 1)
            
        default:
            return false
        }
    }
    
    // Checks if all castling conditions are met
    func canCastle(color: Bool, type: CastleType) -> Bool {
        let rank = color ? 0 : 7
        let kingStartCol = 4
        
        // 1. Ensure King is at the start position and is the correct piece
        guard let kingPiece = board[rank][kingStartCol].piece, kingPiece.type == "king", kingPiece.isWhite == color else {
            return false
        }
        
        // 2. Check for King and Rook moves
        let hasKingMoved = color ? self.hasWhiteKingMoved : self.hasBlackKingMoved
        let hasRookMoved: Bool
        let rookCol: Int
        let squaresKingPasses: [(Int, Int)] // Squares King moves through/to
        let pathCheckSquares: [Int] // Columns between King and Rook
        
        switch type {
        case .kingSide:
            hasRookMoved = color ? self.hasWhiteKingSideRookMoved : self.hasBlackKingSideRookMoved
            rookCol = 7
            squaresKingPasses = [(rank, 5), (rank, 6)]
            pathCheckSquares = [5, 6]
        case .queenSide:
            hasRookMoved = color ? self.hasWhiteQueenSideRookMoved : self.hasBlackQueenSideRookMoved
            rookCol = 0
            squaresKingPasses = [(rank, 3), (rank, 2)]
            pathCheckSquares = [1, 2, 3]
        }
        
        if hasKingMoved || hasRookMoved { return false }
        
        // 3. Ensure the Rook is actually there 
        guard let rookPiece = board[rank][rookCol].piece, rookPiece.type == "rook" else { return false }
        
        // 4. Check for pieces in the path (pathCheckSquares covers all squares between King and Rook)
        for col in pathCheckSquares {
            if board[rank][col].piece != nil { return false }
        }
        
        // 5. King must not be in check and must not pass through or land on an attacked square
        
        // Must not be in check *now*
        if self.isInCheck(for: color) { return false }
        
        // Must not pass through or land on attacked squares
        for squarePos in squaresKingPasses {
            if self.isSquareAttacked(at: squarePos, by: color) {
                return false
            }
        }
        
        return true
    }
    
    // Checks if a move is legal (pseudo-legal AND does not leave King in check).
    func isMoveLegal(from: (Int, Int), to: (Int, Int)) -> Bool {
        // 1. Check pseudo-legality first
        guard self.isMovePseudoLegal(from: from, to: to) else { return false }
        
        guard let originalPiece = board[from.0][from.1].piece else { return false }
        let playerColor = originalPiece.isWhite
        
        // --- Special check for Castling ---
        if originalPiece.type == "king" && abs(to.1 - from.1) == 2 {
            let castleType: CastleType = to.1 > from.1 ? .kingSide : .queenSide
            return self.canCastle(color: playerColor, type: castleType)
        }
        
        // --- Simulation check for all other moves ---
        
        // Create a value type copy of the board array for simulation
        var simulationBoard = board 
        
        // 1. Simulate the move on the copied board 
        
        // Remove piece from source square
        simulationBoard[from.0][from.1].piece = nil
        
        // Place piece at destination square
        simulationBoard[to.0][to.1].piece = originalPiece
        
        // 2. Check if the player's King is in check *after* the simulated move
        // Temporarily store the original board and replace it with the simulation
        let originalBoard = board
        board = simulationBoard // Temporarily switch to the simulated board
        
        let leavesKingSafe = !self.isInCheck(for: playerColor)
        
        // --- Revert move ---
        board = originalBoard // Revert to the original board state
        
        return leavesKingSafe
    }
    
    // Checks if the specified color has any move that keeps their King safe
    func hasLegalMoves(for color: Bool) -> Bool {
        for r1 in 0..<8 {
            for c1 in 0..<8 {
                if let piece = board[r1][c1].piece, piece.isWhite == color {
                    for r2 in 0..<8 {
                        for c2 in 0..<8 {
                            if self.isMoveLegal(from: (r1, c1), to: (r2, c2)) {
                                return true // Found at least one legal move
                            }
                        }
                    }
                }
            }
        }
        return false // No legal moves found
    }
    
    // MARK: - Pawn Promotion Logic
    func promotePawn(to type: String, clock: ChessClock) {
        guard isPromoting, let loc = promotionLocation else { return }
        
        let isWhite = board[loc.0][loc.1].piece?.isWhite ?? true
        
        // 1. Replace the piece with the promotion choice
        var promotionRow = board[loc.0]
        promotionRow[loc.1].piece = createPiece(type: type, isWhite: isWhite)
        board[loc.0] = promotionRow
        
        // 2. Reset promotion state
        isPromoting = false
        promotionLocation = nil
        
        // 3. Complete Turn
        isWhiteTurn.toggle()
        clock.switchPlayer()
        
        // 4. Check for Checkmate/Stalemate
        let nextPlayerColor = isWhiteTurn
        if !self.hasLegalMoves(for: nextPlayerColor) {
            if self.isInCheck(for: nextPlayerColor) {
                // Checkmate!
                gameState = .checkmate(winnerIsWhite: !nextPlayerColor)
                clock.stop()
            } else {
                // Stalemate!
                gameState = .stalemate
                clock.stop()
            }
        }
    }
    
    // Helper function to execute the Rook move for castling
    private func executeRookMove(forCastling moveType: CastleType, color: Bool) {
        let rank = color ? 0 : 7 // White (0) or Black (7)
        let rookFromCol: Int
        let rookToCol: Int
        
        switch moveType {
        case .kingSide:
            rookFromCol = 7
            rookToCol = 5
        case .queenSide:
            rookFromCol = 0
            rookToCol = 3
        }
        
        // 1. Get the Rook
        guard let rookPiece = board[rank][rookFromCol].piece else { return }
        
        // 2. Modify the board row
        var rankRow = board[rank]
        rankRow[rookFromCol].piece = nil // Remove from source
        rankRow[rookToCol].piece = rookPiece // Place at destination
        
        // 3. Re-assign the modified row struct
        board[rank] = rankRow
    }
    
    // MARK: Move Execution
    func movePiece(from: (Int, Int), to: (Int, Int), clock: ChessClock) {
        // 1. Game State Guard
        guard self.gameState == .active else { return }
        
        let fromPos = from
        let toPos = to
        
        let piece = board[fromPos.0][fromPos.1].piece
        guard let movingPiece = piece else { return }
        
        // 2. Check Turn
        guard movingPiece.isWhite == isWhiteTurn else { return }
        
        // 3. Check FULL Legality
        guard self.isMoveLegal(from: fromPos, to: toPos) else { return }
        
        // 4. Determine if it is a castling move BEFORE piece is moved
        let isCastlingMove = movingPiece.type == "king" && abs(toPos.1 - fromPos.1) == 2
        let isWhite = movingPiece.isWhite
        
        // 5. Update Castling Rights
        let startRank = isWhite ? 0 : 7
        
        if movingPiece.type == "king" && fromPos.0 == startRank && fromPos.1 == 4 {
            if isWhite { self.hasWhiteKingMoved = true } else { self.hasBlackKingMoved = true }
        }
        
        if movingPiece.type == "rook" && fromPos.0 == startRank {
            if fromPos.1 == 0 { // Queenside Rook (Rook on A-file)
                if isWhite { self.hasWhiteQueenSideRookMoved = true } else { self.hasBlackQueenSideRookMoved = true }
            } else if fromPos.1 == 7 { // Kingside Rook (Rook on H-file)
                if isWhite { self.hasWhiteKingSideRookMoved = true } else { self.hasBlackKingSideRookMoved = true }
            }
        }
        
        // 6. Execute King/General Piece Move 
        
        // Modify the board directly using value semantics.
        
        // A. Remove piece from source
        board[fromPos.0][fromPos.1].piece = nil
        
        // B. Place piece at destination (capturing if necessary)
        board[toPos.0][toPos.1].piece = movingPiece
        
        // 7. Execute Rook move if Castling
        if isCastlingMove {
            let castleType: CastleType = toPos.1 == 6 ? .kingSide : .queenSide
            executeRookMove(forCastling: castleType, color: isWhite)
        }
        
        // Set last move coordinates for visual feedback
        lastMoveFrom = fromPos
        lastMoveTo = toPos
        
        // 8. Check for Pawn Promotion (Must happen BEFORE turn switch)
        if movingPiece.type == "pawn" {
            let promotionRank = movingPiece.isWhite ? 7 : 0
            if toPos.0 == promotionRank {
                self.isPromoting = true
                self.promotionLocation = toPos
                // Game is now paused waiting for promotion choice
                return
            }
        }
        
        // 9. Complete Turn (Only if no promotion is needed)
        
        // Switch turn and clock
        isWhiteTurn.toggle()
        clock.switchPlayer()
        
        // --- Post-move Checkmate/Stalemate Check ---
        let nextPlayerColor = isWhiteTurn
        if !self.hasLegalMoves(for: nextPlayerColor) {
            if self.isInCheck(for: nextPlayerColor) {
                // Checkmate! The current player (who just moved) wins
                gameState = .checkmate(winnerIsWhite: !nextPlayerColor)
                clock.stop()
            } else {
                // Stalemate! Draw
                gameState = .stalemate
                clock.stop()
            }
        }
    }
}

// MARK: - Chess Clock View
struct ChessClockView: View {
    @ObservedObject var clock: ChessClock
    @ObservedObject var game: ChessGame
    
    private func formatTime(time: TimeInterval) -> String {
        guard clock.selectedTimeControl != .infinite else { return "∞" }
        let minutes = Int(time) / 60
        let seconds = Int(time) % 60
        return String(format: "%02d:%02d", minutes, seconds)
    }
    
    // Computed property for status text color
    private var statusTextColor: Color {
        switch game.gameState {
        case .checkmate(let isWhite): return isWhite ? .white : .cyan 
        case .active, .waitingToStart, .stalemate: return .white
        }
    }
    
    // Computed property for status background color
    private var statusBackgroundColor: Color {
        switch game.gameState {
        case .checkmate, .stalemate: return Color.green.opacity(0.8)
        case .active: return game.isPromoting ? Color.orange.opacity(0.8) : (game.isWhiteTurn ? Color.blue.opacity(0.8) : Color.red.opacity(0.8))
        case .waitingToStart: return Color.gray.opacity(0.8)
        }
    }
    
    // Computed property for the Start/Unpause button text
    private var startButtonText: String {
        if game.gameState == .active {
            return "In Play"
        } else if game.gameState == .waitingToStart && game.lastMoveFrom != nil {
            // Game is paused after a move was made
            return "Unpause Game"
        } else {
            // Game is waiting to start from the beginning
            return "Start Game"
        }
    }
    
    func performFullReset() {
        // This ensures the board, castling rights, and game state are all reset
        game.setupBoard() 
        game.isWhiteTurn = true
        // clock.reset() is called below by the onChange handler
    }
    
    var body: some View {
        VStack(spacing: 15) {
            
            // Time Control Picker
            Picker("Time Control", selection: $clock.selectedTimeControl) {
                ForEach(TimeControl.allCases) { control in
                    Text(control.rawValue).tag(control)
                }
            }
            .pickerStyle(.segmented)
            .padding(.horizontal)
            // Picker is only disabled when game is active or promoting
            .disabled(game.gameState == .active || game.isPromoting) 
            
            Text(game.gameState == .active ? 
                 (game.isPromoting ? "PROMOTION REQUIRED" : (game.isWhiteTurn ? "White's Turn" : "Black's Turn")) : 
                    game.gameState.statusText)
            .font(.title2)
            .bold()
            .foregroundColor(statusTextColor)
            .padding(.horizontal)
            .padding(.vertical, 5)
            .background(statusBackgroundColor)
            .cornerRadius(10)
            
            HStack(spacing: 20) {
                // Player 1 / White Clock
                Text("White: \(formatTime(time: clock.player1Time))")
                    .font(.title3)
                    .bold()
                    .frame(width: 120)
                    .foregroundColor(clock.isPlayer1Active && game.gameState == .active && !game.isPromoting ? .green : .white)
                    .padding()
                    .background(.ultraThinMaterial)
                    .cornerRadius(10)
                
                // Player 2 / Black Clock
                Text("Black: \(formatTime(time: clock.player2Time))")
                    .font(.title3)
                    .bold()
                    .frame(width: 120)
                    .foregroundColor(!clock.isPlayer1Active && game.gameState == .active && !game.isPromoting ? .green : .white)
                    .padding()
                    .background(.ultraThinMaterial)
                    .cornerRadius(10)
            }
            
            HStack(spacing: 15) {
                // Start / Unpause Button
                Button(startButtonText) { 
                    clock.start(game: game) 
                }
                .buttonStyle(.borderedProminent)
                .disabled(game.gameState != .waitingToStart) 
                
                // Stop/Pause Button
                Button("Stop/Pause") { 
                    clock.stop()
                    if game.gameState == .active {
                        game.gameState = .waitingToStart 
                    }
                }
                .buttonStyle(.borderedProminent)
                .disabled(game.gameState != .active) 
                
                // Reset Button
                Button("Reset Board & Clock") { 
                    performFullReset() // Reset game state
                    clock.reset() // Reset clock
                }
                .buttonStyle(.borderedProminent)
            }
        }
        .onAppear {
            clock.reset() 
        }
        // CRITICAL UPDATE: When time control changes, reset the ENTIRE game state.
        .onChange(of: clock.selectedTimeControl) { _ in 
            performFullReset() // Reset game state (board, moves, etc.)
            clock.reset() // Reset clock times and timer
        }
    }
}

// MARK: - Chess Square View
struct ChessSquareView: View {
    var square: ChessSquare
    @ObservedObject var game: ChessGame
    @ObservedObject var clock: ChessClock
    var squareSize: CGFloat 
    
    @State private var dragOffset: CGSize = .zero
    @State private var isDragging: Bool = false
    
    var isWhiteSquare: Bool { (square.row + square.col) % 2 == 0 }
    
    var isLastMoveSquare: Bool {
        guard let from = game.lastMoveFrom, let to = game.lastMoveTo else { return false }
        return (square.row == from.0 && square.col == from.1) || (square.row == to.0 && square.col == to.1)
    }
    
    var isKingInCheck: Bool {
        guard let piece = square.piece, piece.type == "king" else { return false }
        return game.gameState == .active && game.isInCheck(for: piece.isWhite)
    }
    
    // Drag Gesture Definition
    var dragPiece: some Gesture {
        DragGesture(coordinateSpace: .global) 
            .onChanged { value in
                guard game.gameState == .active, 
                        !game.isPromoting,
                      let piece = square.piece, 
                        piece.isWhite == game.isWhiteTurn 
                else { return }
                
                isDragging = true
                dragOffset = value.translation
            }
            .onEnded { value in
                guard isDragging else { return }
                
                let startPos = (row: square.row, col: square.col)
                let startCenter = value.startLocation
                let dropLocation = value.location
                
                let dX = dropLocation.x - startCenter.x
                // Row is reversed due to SwiftUI coordinate system and the board rendering (row 7 at top)
                let dY = dropLocation.y - startCenter.y
                
                let colDisplacement = Int(round(dX / squareSize))
                let rowDisplacement = Int(round(-dY / squareSize)) 
                
                let targetRow = startPos.row + rowDisplacement
                let targetCol = startPos.col + colDisplacement
                
                let targetPos = (targetRow, targetCol) 
                
                // 4. Attempt the move if target is valid
                if targetRow >= 0 && targetRow < 8 && targetCol >= 0 && targetCol < 8 {
                    game.movePiece(from: (startPos.row, startPos.col), to: targetPos, clock: clock)
                }
                
                // Reset state
                isDragging = false
                dragOffset = .zero
            }
    }
    
    var body: some View {
        ZStack {
            // Square Background.
            Rectangle()
                .fill(isKingInCheck ? Color.red.opacity(0.8) : 
                        (isLastMoveSquare ? Color.yellow.opacity(0.5) : 
                            (isWhiteSquare ? Color(red: 0.9, green: 0.9, blue: 0.8) : Color(red: 0.2, green: 0.3, blue: 0.5))))
                .border(Color.cyan.opacity(0.5), width: 1)
            
            // Piece Display and Drag implementation
            if let piece = square.piece {
                Text(piece.symbol)
                    .font(.system(size: 50))
                    .foregroundColor(piece.isWhite ? .white : .cyan) 
                    .shadow(color: .black, radius: 1.5)
                    .scaleEffect(isDragging ? 1.2 : 1.0) 
                    .opacity(isDragging ? 0.8 : 1.0) 
                    .offset(dragOffset) 
                    .zIndex(isDragging ? 10 : 0) 
                    .gesture(dragPiece) 
            }
        }
        .frame(width: squareSize, height: squareSize) 
        .aspectRatio(1, contentMode: .fit)
    }
}

// MARK: - Pawn Promotion View
struct PromotionView: View {
    @ObservedObject var game: ChessGame
    @ObservedObject var clock: ChessClock
    
    // Array of piece types a pawn can promote to
    let promotionPieces = ["queen", "rook", "bishop", "knight"]
    
    var body: some View {
        VStack(spacing: 20) {
            Text("Pawn Promotion")
                .font(.largeTitle)
                .bold()
                .foregroundColor(.white)
            
            Text("Select a piece to replace your pawn:")
                .font(.headline)
                .foregroundColor(.white.opacity(0.8))
            
            HStack(spacing: 15) {
                ForEach(promotionPieces, id: \.self) { type in
                    Button {
                        // Call the game function to finalize promotion and resume turn
                        game.promotePawn(to: type, clock: clock)
                    } label: {
                        Text(ChessPiece.getSymbol(type: type, isWhite: game.isWhiteTurn))
                            .font(.system(size: 60))
                            .frame(width: 60, height: 60)
                            .background(Color.blue.opacity(0.7))
                            .cornerRadius(10)
                            .shadow(radius: 5)
                            .overlay(
                                RoundedRectangle(cornerRadius: 10)
                                    .stroke(Color.cyan, lineWidth: 2)
                            )
                    }
                }
            }
        }
        .padding(30)
        .background(
            RoundedRectangle(cornerRadius: 20)
                .fill(Color(red: 0.1, green: 0.1, blue: 0.3).opacity(0.95))
        )
        .overlay(
            RoundedRectangle(cornerRadius: 20)
                .stroke(Color.cyan, lineWidth: 3)
        )
    }
}


// MARK: - Main Chess View
struct ChessGameView: View {
    // Initialize clock with the default duration based on the TimeControl enum
    @StateObject var game = ChessGame()
    @StateObject var clock = ChessClock(initialTime: TimeControl.fiveMin.duration ?? 300) 
    
    var body: some View {
        ZStack { 
            VStack(spacing: 20) {
                Spacer()
                
                // Clock and Status
                ChessClockView(clock: clock, game: game)
                    .frame(maxWidth: 500) 
                
                GeometryReader { boardGeometry in
                    // Calculate the square size based on the available space (minimum of width/height)
                    let padding: CGFloat = 16 
                    let boardWidth = min(boardGeometry.size.width, boardGeometry.size.height)
                    let calculatedSquareSize = (boardWidth - padding) / 8 
                    
                    VStack(spacing: 0) {
                        // Board is rendered from Black's side (row 7) to White's side (row 0)
                        ForEach((0..<game.board.count).reversed(), id: \.self) { rowIndex in 
                            let row = game.board[rowIndex]
                            HStack(spacing: 0) {
                                ForEach(row) { square in
                                    // Pass the calculated square size to each square view
                                    ChessSquareView(square: square, game: game, clock: clock, squareSize: calculatedSquareSize)
                                }
                            }
                        }
                    }
                    .padding(8)
                    .background(Color.gray.opacity(0.7))
                    .cornerRadius(15)
                    .frame(width: boardWidth, height: boardWidth) 
                    // Center the board within the GeometryReader
                    .position(x: boardGeometry.size.width / 2, y: boardGeometry.size.height / 2) 
                    .shadow(color: .black.opacity(0.5), radius: 10, x: 5, y: 5)
                }
                .aspectRatio(1, contentMode: .fit)
                
                Spacer()
            }
            .padding()
            // Full screen effect
            .background(LinearGradient(colors: [.black, Color(red: 0.1, green: 0, blue: 0.2), .black], startPoint: .top, endPoint: .bottom))
            .edgesIgnoringSafeArea(.all)
            .blur(radius: game.isPromoting ? 5 : 0) // Blur the board when promoting 
            
            // Promotion View Overlay
            if game.isPromoting {
                PromotionView(game: game, clock: clock)
            }
        }
    }
}

// MARK: - SwiftUI App Entry (The ONLY structure with @main)
@main
struct SciFiChessApp: App {
    var body: some Scene {
        WindowGroup {
            ChessGameView()
        }
    }
}

