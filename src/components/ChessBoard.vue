<script setup>
import { ref, onMounted, computed, watch } from 'vue'
import OpenAI from 'openai'

const board = ref(Array(8).fill().map(() => Array(8).fill(null)))
const isWhiteTurn = ref(true)
const selectedPiece = ref(null)
const selectedPosition = ref(null)
const validMoves = ref([])
const currentFen = ref('')
const isGameOver = ref(false)
const gameStatus = ref('')
const moveHistory = ref([])
const currentMoveIndex = ref(-1)
const openaiEndpoint = ref('https://api.openai.com/v1/chat/completions')
const openaiModel = ref('gpt-4')
const openaiKey = ref('')
const isWaitingForAI = ref(false)
const aiError = ref('')
const isSettingsValid = ref(false)
const isTestingSettings = ref(false)
const settingsError = ref('')
const availableModels = ref([])
const isLoadingModels = ref(false)
const modelError = ref('')
const autoPlayBlack = ref(true)

const canMakeAIMove = computed(() => {
  return openaiKey.value && 
         !isWaitingForAI.value && 
         !isGameOver.value && 
         currentMoveIndex.value === moveHistory.value.length - 1
})

const canMakeMove = computed(() => {
  return isSettingsValid.value && currentMoveIndex.value === moveHistory.value.length - 1 && !isGameOver.value
})

// Initialize the board with pieces
const initializeBoard = () => {
  console.log('Initializing board...')
  // Clear the board first
  board.value = Array(8).fill().map(() => Array(8).fill(null))
  
  // Set up pawns
  for (let i = 0; i < 8; i++) {
    board.value[6][i] = { type: 'pawn', color: 'white' }
    board.value[1][i] = { type: 'pawn', color: 'black' }
  }

  // Set up other pieces
  const pieces = ['rook', 'knight', 'bishop', 'queen', 'king', 'bishop', 'knight', 'rook']
  for (let i = 0; i < 8; i++) {
    board.value[7][i] = { type: pieces[i], color: 'white' }
    board.value[0][i] = { type: pieces[i], color: 'black' }
  }
  console.log('Board initialized:', board.value)
}

initializeBoard()

const isValidPosition = (row, col) => {
  return row >= 0 && row < 8 && col >= 0 && col < 8
}

const isPathClear = (fromRow, fromCol, toRow, toCol) => {
  const rowStep = fromRow === toRow ? 0 : (toRow - fromRow) / Math.abs(toRow - fromRow)
  const colStep = fromCol === toCol ? 0 : (toCol - fromCol) / Math.abs(toCol - fromCol)
  let currentRow = fromRow + rowStep
  let currentCol = fromCol + colStep

  while (currentRow !== toRow || currentCol !== toCol) {
    if (board.value[currentRow][currentCol] !== null) {
      return false
    }
    currentRow += rowStep
    currentCol += colStep
  }
  return true
}

const isValidMove = (fromRow, fromCol, toRow, toCol) => {
  const piece = board.value[fromRow][fromCol]
  const targetSquare = board.value[toRow][toCol]

  // Can't capture your own piece
  if (targetSquare && targetSquare.color === piece.color) {
    return false
  }

  const rowDiff = Math.abs(toRow - fromRow)
  const colDiff = Math.abs(toCol - fromCol)

  let isValidBasicMove = false;

  switch (piece.type) {
    case 'pawn':
      // Direction of movement depends on color
      const direction = piece.color === 'white' ? -1 : 1
      const startRow = piece.color === 'white' ? 6 : 1

      // Regular move forward
      if (colDiff === 0 && toRow === fromRow + direction && !targetSquare) {
        isValidBasicMove = true
      }
      // Initial two-square move
      else if (colDiff === 0 && fromRow === startRow && toRow === fromRow + 2 * direction && !targetSquare && !board.value[fromRow + direction][fromCol]) {
        isValidBasicMove = true
      }
      // Capture diagonally
      else if (colDiff === 1 && toRow === fromRow + direction && targetSquare && targetSquare.color !== piece.color) {
        isValidBasicMove = true
      }
      break

    case 'knight':
      isValidBasicMove = (rowDiff === 2 && colDiff === 1) || (rowDiff === 1 && colDiff === 2)
      break

    case 'bishop':
      isValidBasicMove = rowDiff === colDiff && isPathClear(fromRow, fromCol, toRow, toCol)
      break

    case 'rook':
      isValidBasicMove = (rowDiff === 0 || colDiff === 0) && isPathClear(fromRow, fromCol, toRow, toCol)
      break

    case 'queen':
      isValidBasicMove = (rowDiff === colDiff || rowDiff === 0 || colDiff === 0) && isPathClear(fromRow, fromCol, toRow, toCol)
      break

    case 'king':
      isValidBasicMove = rowDiff <= 1 && colDiff <= 1
      break

    default:
      return false
  }

  if (!isValidBasicMove) {
    return false
  }

  // Make the move temporarily
  const originalTarget = board.value[toRow][toCol]
  board.value[toRow][toCol] = piece
  board.value[fromRow][fromCol] = null

  // Check if the move leaves the king in check
  const leavesInCheck = isKingInCheck(piece.color)

  // Undo the move
  board.value[fromRow][fromCol] = piece
  board.value[toRow][toCol] = originalTarget

  return !leavesInCheck
}

const getPieceEmoji = (piece) => {
  if (!piece) return ''
  const emojis = {
    // White pieces (actual white colored pieces)
    king: '♔', queen: '♕', bishop: '♗', knight: '♘', rook: '♖', pawn: '♙',
    // Black pieces (actual black colored pieces)
    king_black: '♚', queen_black: '♛', bishop_black: '♝', knight_black: '♞', rook_black: '♜', pawn_black: '♟'
  }
  return piece.color === 'white' ? emojis[piece.type] : emojis[`${piece.type}_black`]
}

const calculateValidMoves = (row, col) => {
  const moves = []
  for (let i = 0; i < 8; i++) {
    for (let j = 0; j < 8; j++) {
      if (isValidMove(row, col, i, j)) {
        moves.push({ row: i, col: j })
      }
    }
  }
  return moves
}

const generateFen = () => {
  let fen = ''
  
  // Board position
  for (let i = 0; i < 8; i++) {
    let emptyCount = 0
    for (let j = 0; j < 8; j++) {
      const piece = board.value[i][j]
      if (piece === null) {
        emptyCount++
      } else {
        if (emptyCount > 0) {
          fen += emptyCount
          emptyCount = 0
        }
        const pieceSymbol = getPieceFenSymbol(piece)
        fen += pieceSymbol
      }
    }
    if (emptyCount > 0) {
      fen += emptyCount
    }
    if (i < 7) fen += '/'
  }

  // Active color
  fen += ' ' + (isWhiteTurn.value ? 'w' : 'b')

  // Castling availability (simplified - assuming no castling for now)
  fen += ' -'

  // En passant target square (simplified - no en passant for now)
  fen += ' -'

  // Halfmove clock (simplified - always 0 for now)
  fen += ' 0'

  // Fullmove number (simplified - always 1 for now)
  fen += ' 1'

  return fen
}

const getPieceFenSymbol = (piece) => {
  const symbols = {
    king: 'k',
    queen: 'q',
    bishop: 'b',
    knight: 'n',
    rook: 'r',
    pawn: 'p'
  }
  const symbol = symbols[piece.type]
  return piece.color === 'white' ? symbol.toUpperCase() : symbol
}

const updateUrl = () => {
  const url = new URL(window.location)
  url.searchParams.set('fen', currentFen.value)
  // Encode the history array to prevent issues with special characters
  url.searchParams.set('history', encodeURIComponent(moveHistory.value.join('|')))
  url.searchParams.set('move', currentMoveIndex.value.toString())
  window.history.replaceState({}, '', url)
}

const loadFromUrl = () => {
  const url = new URL(window.location)
  const historyStr = url.searchParams.get('history')
  const moveIndex = url.searchParams.get('move')
  const fenFromUrl = url.searchParams.get('fen')
  
  if (historyStr) {
    try {
      // Decode the history string and split by the separator
      moveHistory.value = decodeURIComponent(historyStr).split('|')
      currentMoveIndex.value = moveIndex ? parseInt(moveIndex) : moveHistory.value.length - 1
      
      // Ensure the index is valid
      if (currentMoveIndex.value < 0) currentMoveIndex.value = 0
      if (currentMoveIndex.value >= moveHistory.value.length) {
        currentMoveIndex.value = moveHistory.value.length - 1
      }

      // Load the position from history
      if (currentMoveIndex.value >= 0 && currentMoveIndex.value < moveHistory.value.length) {
        parseFen(moveHistory.value[currentMoveIndex.value])
        currentFen.value = moveHistory.value[currentMoveIndex.value]
      }
    } catch (error) {
      console.error('Error loading history:', error)
      // Fallback to initial position if history loading fails
      initializeBoard()
      currentFen.value = generateFen()
      moveHistory.value = [currentFen.value]
      currentMoveIndex.value = 0
    }
  } else if (fenFromUrl) {
    // If no history but FEN exists, start new history with this position
    if (parseFen(fenFromUrl)) {
      currentFen.value = fenFromUrl
      moveHistory.value = [fenFromUrl]
      currentMoveIndex.value = 0
    } else {
      // If FEN is invalid, initialize with starting position
      initializeBoard()
      currentFen.value = generateFen()
      moveHistory.value = [currentFen.value]
      currentMoveIndex.value = 0
    }
  } else {
    // No history or FEN in URL, start with initial position
    initializeBoard()
    currentFen.value = generateFen()
    moveHistory.value = [currentFen.value]
    currentMoveIndex.value = 0
  }
  
  // Always update URL after loading to ensure proper encoding
  updateUrl()
}

const addMoveToHistory = (fen) => {
  // If we're not at the end of the history, truncate the future moves
  if (currentMoveIndex.value < moveHistory.value.length - 1) {
    moveHistory.value = moveHistory.value.slice(0, currentMoveIndex.value + 1)
  }
  moveHistory.value.push(fen)
  currentMoveIndex.value = moveHistory.value.length - 1
  updateUrl()
}

const navigateMove = (index) => {
  if (index >= 0 && index < moveHistory.value.length) {
    currentMoveIndex.value = index
    parseFen(moveHistory.value[index])
    updateUrl()
  }
}

const handleCellClick = (row, col) => {
  if (!isSettingsValid.value) {
    settingsError.value = 'Please save and validate OpenAI settings first'
    return
  }
  if (currentMoveIndex.value < moveHistory.value.length - 1) return
  if (isGameOver.value) return
  if (isWaitingForAI.value) {
    settingsError.value = 'Please wait for AI to finish thinking'
    return
  }

  console.log('Cell clicked:', row, col)
  console.log('Current turn:', isWhiteTurn.value ? 'white' : 'black')
  
  const clickedPiece = board.value[row][col]
  console.log('Clicked piece:', clickedPiece)
  console.log('Currently selected:', selectedPiece.value, 'at position:', selectedPosition.value)
  
  // If no piece is selected and clicked cell has a piece of the current turn's color
  if (!selectedPiece.value && clickedPiece && clickedPiece.color === (isWhiteTurn.value ? 'white' : 'black')) {
    console.log('Selecting piece:', clickedPiece)
    selectedPiece.value = JSON.parse(JSON.stringify(clickedPiece))
    selectedPosition.value = { row, col }
    validMoves.value = calculateValidMoves(row, col)
    return
  }

  // If a piece is selected
  if (selectedPiece.value) {
    // If clicking on the same piece, deselect it
    if (selectedPosition.value.row === row && selectedPosition.value.col === col) {
      console.log('Deselecting piece')
      selectedPiece.value = null
      selectedPosition.value = null
      validMoves.value = []
      return
    }

    // If clicking on a friendly piece, select that piece instead
    if (clickedPiece && clickedPiece.color === selectedPiece.value.color) {
      console.log('Switching selection to:', clickedPiece)
      selectedPiece.value = JSON.parse(JSON.stringify(clickedPiece))
      selectedPosition.value = { row, col }
      validMoves.value = calculateValidMoves(row, col)
      return
    }

    // Check if the move is valid
    if (!isValidMove(selectedPosition.value.row, selectedPosition.value.col, row, col)) {
      console.log('Invalid move')
      return
    }

    // Move the piece
    console.log('Moving piece from', selectedPosition.value, 'to', { row, col })
    const fromRow = selectedPosition.value.row
    const fromCol = selectedPosition.value.col
    
    // Make the move
    board.value[row][col] = { ...selectedPiece.value }
    board.value[fromRow][fromCol] = null
    
    // Reset selection and switch turns
    selectedPiece.value = null
    selectedPosition.value = null
    validMoves.value = []
    isWhiteTurn.value = !isWhiteTurn.value
    
    // Update FEN and history
    currentFen.value = generateFen()
    addMoveToHistory(currentFen.value)

    // Check game status
    checkGameStatus()
  }
}

const isSelected = (row, col) => {
  return selectedPosition.value && 
         selectedPosition.value.row === row && 
         selectedPosition.value.col === col
}

const isValidMoveSquare = (row, col) => {
  return validMoves.value.some(move => move.row === row && move.col === col)
}

const parseFen = (fen) => {
  try {
    // Split FEN into its components
    const [position, turn, castling, enPassant, halfmove, fullmove] = fen.split(' ')
    
    // Parse board position
    const rows = position.split('/')
    if (rows.length !== 8) throw new Error('Invalid FEN: must have 8 ranks')
    
    // Clear the current board
    board.value = Array(8).fill().map(() => Array(8).fill(null))
    
    // Fill the board
    for (let i = 0; i < 8; i++) {
      let j = 0
      for (let char of rows[i]) {
        if (/[1-8]/.test(char)) {
          j += parseInt(char)
        } else {
          const piece = fenCharToPiece(char)
          if (!piece) throw new Error(`Invalid piece character: ${char}`)
          board.value[i][j] = piece
          j++
        }
        if (j > 8) throw new Error('Invalid FEN: rank too long')
      }
      if (j !== 8) throw new Error('Invalid FEN: rank too short')
    }
    
    // Set turn
    isWhiteTurn.value = turn === 'w'
    
    // Reset selection
    selectedPiece.value = null
    selectedPosition.value = null
    validMoves.value = []
    
    // Reset game status
    isGameOver.value = false
    gameStatus.value = ''
    
    // Check initial position's game status
    checkGameStatus()
    
    return true
  } catch (error) {
    console.error('FEN parsing error:', error)
    return false
  }
}

const fenCharToPiece = (char) => {
  const pieceTypes = {
    'k': 'king',
    'q': 'queen',
    'r': 'rook',
    'b': 'bishop',
    'n': 'knight',
    'p': 'pawn'
  }
  
  const isWhite = char === char.toUpperCase()
  const pieceType = pieceTypes[char.toLowerCase()]
  
  if (!pieceType) return null
  
  return {
    type: pieceType,
    color: isWhite ? 'white' : 'black'
  }
}

const handleFenInput = (event) => {
  const newFen = event.target.value
  if (parseFen(newFen)) {
    currentFen.value = newFen
    // Reset history when manually entering a FEN
    moveHistory.value = [newFen]
    currentMoveIndex.value = 0
    updateUrl()
  }
}

// Add after isValidMove function
const isKingInCheck = (color) => {
  // Find the king's position
  let kingRow = -1
  let kingCol = -1
  for (let i = 0; i < 8; i++) {
    for (let j = 0; j < 8; j++) {
      const piece = board.value[i][j]
      if (piece && piece.type === 'king' && piece.color === color) {
        kingRow = i
        kingCol = j
        break
      }
    }
    if (kingRow !== -1) break
  }

  // Check if any opponent's piece can capture the king
  const opponentColor = color === 'white' ? 'black' : 'white'
  for (let i = 0; i < 8; i++) {
    for (let j = 0; j < 8; j++) {
      const piece = board.value[i][j]
      if (piece && piece.color === opponentColor) {
        if (isValidMove(i, j, kingRow, kingCol)) {
          return true
        }
      }
    }
  }
  return false
}

const hasLegalMoves = (color) => {
  // Try every possible move for every piece of the given color
  for (let fromRow = 0; fromRow < 8; fromRow++) {
    for (let fromCol = 0; fromCol < 8; fromCol++) {
      const piece = board.value[fromRow][fromCol]
      if (piece && piece.color === color) {
        for (let toRow = 0; toRow < 8; toRow++) {
          for (let toCol = 0; toCol < 8; toCol++) {
            if (isValidMove(fromRow, fromCol, toRow, toCol)) {
              // Make the move temporarily
              const originalTarget = board.value[toRow][toCol]
              board.value[toRow][toCol] = piece
              board.value[fromRow][fromCol] = null

              // Check if the move leaves or puts the king in check
              const inCheck = isKingInCheck(color)

              // Undo the move
              board.value[fromRow][fromCol] = piece
              board.value[toRow][toCol] = originalTarget

              if (!inCheck) {
                return true // Found at least one legal move
              }
            }
          }
        }
      }
    }
  }
  return false // No legal moves found
}

const checkGameStatus = () => {
  const currentColor = isWhiteTurn.value ? 'white' : 'black'
  const inCheck = isKingInCheck(currentColor)
  const canMove = hasLegalMoves(currentColor)

  if (inCheck && !canMove) {
    isGameOver.value = true
    gameStatus.value = `Checkmate! ${currentColor === 'white' ? 'Black' : 'White'} wins!`
  } else if (!inCheck && !canMove) {
    isGameOver.value = true
    gameStatus.value = 'Stalemate! Game is a draw!'
  } else if (inCheck) {
    gameStatus.value = `${currentColor === 'white' ? 'White' : 'Black'} is in check!`
  } else {
    gameStatus.value = ''
  }
}

const resetBoard = () => {
  // Clear all states
  board.value = Array(8).fill().map(() => Array(8).fill(null))
  selectedPiece.value = null
  selectedPosition.value = null
  validMoves.value = []
  isWhiteTurn.value = true
  isGameOver.value = false
  gameStatus.value = ''
  
  // Initialize new board
  initializeBoard()
  
  // Update FEN and history
  currentFen.value = generateFen()
  moveHistory.value = [currentFen.value]
  currentMoveIndex.value = 0
  
  // Update URL
  updateUrl()
}

const createOpenAIClient = () => {
  return new OpenAI({
    apiKey: openaiKey.value,
    baseURL: openaiEndpoint.value,
    dangerouslyAllowBrowser: true
  })
}

const requestAIMove = async () => {
  if (!openaiKey.value) {
    aiError.value = 'Please enter your OpenAI API key first.'
    return
  }

  isWaitingForAI.value = true
  aiError.value = ''

  try {
    const openai = createOpenAIClient()
    const completion = await openai.chat.completions.create({
      model: openaiModel.value,
      messages: [
        {
          role: 'system',
          content: `You are a chess engine. You will receive the current game state in FEN notation and should respond with a new FEN string representing your move. Only respond with the FEN string, no other text. Make a legal chess move for the current player's turn. The FEN string should indicate that it's white's turn to play after your move.`
        },
        {
          role: 'user',
          content: `Current position: ${currentFen.value}\nMake a move for ${isWhiteTurn.value ? 'white' : 'black'}.`
        }
      ],
      temperature: 0.3,
      max_tokens: 100
    })

    let newFen = completion.choices[0].message.content.trim()
    
    // Ensure the FEN string has the correct turn indicator
    const fenParts = newFen.split(' ')
    if (fenParts.length >= 2) {
      // Replace the turn indicator with 'w' (white to play)
      fenParts[1] = 'w'
      newFen = fenParts.join(' ')
    }
    
    // Validate the FEN string
    if (parseFen(newFen)) {
      // Update game state
      currentFen.value = newFen
      addMoveToHistory(currentFen.value)
      checkGameStatus()
    } else {
      throw new Error('AI suggested an invalid FEN position')
    }
  } catch (error) {
    console.error('AI move error:', error)
    aiError.value = `Error: ${error.message}`
    // Restore the previous position if there was an error
    if (moveHistory.value.length > 0) {
      parseFen(moveHistory.value[moveHistory.value.length - 1])
    }
  } finally {
    isWaitingForAI.value = false
  }
}

const fetchAvailableModels = async () => {
  if (!openaiKey.value) {
    modelError.value = 'Please enter your OpenAI API key first'
    return
  }

  isLoadingModels.value = true
  modelError.value = ''

  try {
    const openai = createOpenAIClient()
    const response = await openai.models.list()
    // Filter for chat completion models and sort them
    availableModels.value = response.data
      .filter(model => model.id.includes('gpt'))
      .sort((a, b) => b.id.localeCompare(a.id))
  } catch (error) {
    console.error('Error fetching models:', error)
    modelError.value = `Error: ${error.message}`
  } finally {
    isLoadingModels.value = false
  }
}

const validateAndSaveSettings = async () => {
  if (!openaiKey.value || !openaiEndpoint.value || !openaiModel.value) {
    settingsError.value = 'All fields are required'
    return
  }

  isTestingSettings.value = true
  settingsError.value = ''

  try {
    const openai = createOpenAIClient()
    const completion = await openai.chat.completions.create({
      model: openaiModel.value,
      messages: [
        {
          role: 'system',
          content: 'This is a test message to validate the API connection.'
        },
        {
          role: 'user',
          content: 'Respond with "ok" if you receive this.'
        }
      ],
      max_tokens: 10
    })

    if (completion.choices && completion.choices[0]) {
      isSettingsValid.value = true
      settingsError.value = ''
      // Save settings to localStorage
      localStorage.setItem('chess_openai_settings', JSON.stringify({
        endpoint: openaiEndpoint.value,
        model: openaiModel.value,
        key: openaiKey.value,
        autoPlayBlack: autoPlayBlack.value
      }))
    } else {
      throw new Error('Invalid API response format')
    }
  } catch (error) {
    console.error('Settings validation error:', error)
    settingsError.value = `Error: ${error.message}`
    isSettingsValid.value = false
  } finally {
    isTestingSettings.value = false
  }
}

const loadSettings = () => {
  const savedSettings = localStorage.getItem('chess_openai_settings')
  if (savedSettings) {
    try {
      const settings = JSON.parse(savedSettings)
      openaiEndpoint.value = settings.endpoint
      openaiModel.value = settings.model
      openaiKey.value = settings.key
      autoPlayBlack.value = settings.autoPlayBlack ?? true // Default to true if not saved
      // Validate loaded settings
      validateAndSaveSettings()
    } catch (error) {
      console.error('Error loading settings:', error)
      settingsError.value = 'Error loading saved settings'
      isSettingsValid.value = false
    }
  }
}

const clearSettings = () => {
  localStorage.removeItem('chess_openai_settings')
  openaiEndpoint.value = 'https://api.openai.com/v1/chat/completions'
  openaiModel.value = 'gpt-4'
  openaiKey.value = ''
  autoPlayBlack.value = true // Reset to default value
  isSettingsValid.value = false
  settingsError.value = ''
}

// Add after the existing watchers
watch(isWhiteTurn, async (newValue) => {
  if (autoPlayBlack.value && !newValue && !isGameOver.value) {
    // It's black's turn and auto-play is enabled
    await requestAIMove()
  }
})

// Initialize board and load from URL if present
onMounted(() => {
  loadSettings()
  loadFromUrl()
  if (!currentFen.value) {
    initializeBoard()
    currentFen.value = generateFen()
    moveHistory.value = [currentFen.value]
    currentMoveIndex.value = 0
    updateUrl()
  }
  checkGameStatus()
})
</script>

<template>
  <div class="chess-container">
    <h1>Chess Game</h1>
    <div class="game-layout">
      <div class="board-section">
        <div class="chess-board">
          <div class="status-indicator" :class="{ 'game-over': isGameOver }">
            <div v-if="gameStatus" class="game-status">{{ gameStatus }}</div>
            <div v-else class="turn-indicator">
              {{ isWhiteTurn ? "White's turn" : "Black's turn" }}
              <span v-if="currentMoveIndex < moveHistory.length - 1" class="history-note">
                (Viewing move {{ currentMoveIndex + 1 }})
              </span>
            </div>
          </div>
          <div v-for="(row, rowIndex) in board" :key="rowIndex" class="board-row">
            <div
              v-for="(cell, colIndex) in row"
              :key="colIndex"
              class="board-cell"
              :class="{
                'white-cell': (rowIndex + colIndex) % 2 === 0,
                'black-cell': (rowIndex + colIndex) % 2 === 1,
                'selected': isSelected(rowIndex, colIndex),
                'white-piece': cell && cell.color === 'white',
                'black-piece': cell && cell.color === 'black',
                'valid-move': isValidMoveSquare(rowIndex, colIndex)
              }"
              @click="handleCellClick(rowIndex, colIndex)"
            >
              {{ getPieceEmoji(cell) }}
              <div v-if="isValidMoveSquare(rowIndex, colIndex)" class="move-indicator"></div>
            </div>
          </div>
        </div>
        <div class="control-panel">
          <div class="history-controls">
            <button 
              class="history-btn" 
              @click="navigateMove(0)" 
              :disabled="currentMoveIndex <= 0"
              title="Go to start position"
            >
              ⏮️ Start
            </button>
            <button 
              class="history-btn" 
              @click="navigateMove(currentMoveIndex - 1)" 
              :disabled="currentMoveIndex <= 0"
              title="Previous move"
            >
              ⏪ Previous
            </button>
            <div class="move-counter">
              Move {{ currentMoveIndex + 1 }} / {{ moveHistory.length }}
            </div>
            <button 
              class="history-btn" 
              @click="navigateMove(currentMoveIndex + 1)" 
              :disabled="currentMoveIndex >= moveHistory.length - 1"
              title="Next move"
            >
              Next ⏩
            </button>
            <button 
              class="history-btn" 
              @click="navigateMove(moveHistory.length - 1)" 
              :disabled="currentMoveIndex >= moveHistory.length - 1"
              title="Go to current position"
            >
              Latest ⏭️
            </button>
          </div>
          <button 
            class="reset-btn" 
            @click="resetBoard"
            title="Reset to starting position"
          >
            🔄 New Game
          </button>
          <div class="ai-controls">
            <div class="ai-controls-row">
              <button 
                class="ai-move-btn" 
                @click="requestAIMove"
                :disabled="!canMakeMove || isWaitingForAI"
                title="Request move from AI"
              >
                🤖 AI Move
                <span v-if="isWaitingForAI" class="loading-indicator">...</span>
              </button>
              <label class="auto-play-toggle">
                <input 
                  type="checkbox" 
                  v-model="autoPlayBlack"
                  :disabled="!isSettingsValid || isWaitingForAI"
                >
                <span class="toggle-label">Auto-play as Black</span>
              </label>
            </div>
            <div v-if="aiError" class="ai-error">{{ aiError }}</div>
          </div>
        </div>
        <div class="fen-display">
          <div class="fen-label">FEN:</div>
          <input 
            type="text" 
            class="fen-input" 
            :value="currentFen"
            @input="handleFenInput"
            placeholder="Enter FEN string..."
          />
        </div>
      </div>
      <div class="right-panels">
        <div class="config-panel">
          <h2>AI Configuration</h2>
          <div class="config-form">
            <div class="config-field">
              <label for="openai-endpoint">OpenAI Endpoint:</label>
              <input 
                id="openai-endpoint"
                type="text" 
                v-model="openaiEndpoint"
                class="config-input"
                :disabled="isTestingSettings"
                placeholder="Enter OpenAI API endpoint..."
              />
            </div>
            <div class="config-field">
              <label for="openai-model">OpenAI Model:</label>
              <div class="model-selector">
                <select 
                  id="openai-model"
                  v-model="openaiModel"
                  class="config-input"
                  :disabled="isTestingSettings || isLoadingModels"
                >
                  <option value="">Select a model...</option>
                  <option v-for="model in availableModels" :key="model.id" :value="model.id">
                    {{ model.id }}
                  </option>
                </select>
                <button 
                  class="refresh-models-btn"
                  @click="fetchAvailableModels"
                  :disabled="isLoadingModels || !openaiKey"
                  title="Refresh available models"
                >
                  🔄
                </button>
              </div>
              <div v-if="isLoadingModels" class="loading-text">Loading models...</div>
              <div v-if="modelError" class="model-error">{{ modelError }}</div>
            </div>
            <div class="config-field">
              <label for="openai-key">OpenAI API Key:</label>
              <input 
                id="openai-key"
                type="password" 
                v-model="openaiKey"
                class="config-input"
                :disabled="isTestingSettings"
                placeholder="Enter your OpenAI API key..."
              />
            </div>
            <div class="settings-buttons">
              <button 
                class="save-settings-btn" 
                @click="validateAndSaveSettings"
                :disabled="isTestingSettings"
              >
                {{ isTestingSettings ? '🔄 Testing...' : (isSettingsValid ? '✅ Settings Saved' : '💾 Save Settings') }}
              </button>
              <button 
                class="clear-settings-btn" 
                @click="clearSettings"
                :disabled="isTestingSettings"
              >
                🗑️ Clear Settings
              </button>
            </div>
            <div v-if="settingsError" class="settings-error">{{ settingsError }}</div>
            <div v-if="!isSettingsValid" class="settings-warning">
              ⚠️ Please save and validate settings before playing
            </div>
          </div>
        </div>
        <div class="fen-history-panel">
          <h2>Move History</h2>
          <div class="fen-history-list">
            <div
              v-for="(fen, index) in moveHistory"
              :key="index"
              class="fen-history-item"
              :class="{ 'active': index === currentMoveIndex }"
              @click="navigateMove(index)"
            >
              <div class="move-number">Move {{ index + 1 }}</div>
              <div class="fen-text">{{ fen }}</div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.chess-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  background-color: #2f2f2f;
  min-height: 100vh;
}

h1 {
  color: #ffffff;
  margin-bottom: 20px;
}

.game-layout {
  display: flex;
  gap: 20px;
  width: 100%;
  max-width: 1400px;
  margin: 0 auto;
}

.board-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.chess-board {
  display: inline-block;
  border: 3px solid #1a1a1a;
  padding: 15px;
  background-color: #1a1a1a;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.status-indicator {
  text-align: center;
  margin-bottom: 15px;
  font-size: 1.2em;
  font-weight: bold;
  color: #ffffff;
  padding: 8px;
  background-color: #3a3a3a;
  border-radius: 4px;
  transition: all 0.3s ease;
}

.game-status {
  color: #ffd700;
}

.status-indicator.game-over {
  background-color: #b71c1c;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0% { opacity: 1; }
  50% { opacity: 0.8; }
  100% { opacity: 1; }
}

.turn-indicator {
  text-align: center;
  margin-bottom: 15px;
  font-size: 1.2em;
  font-weight: bold;
  color: #ffffff;
  padding: 8px;
  background-color: #3a3a3a;
  border-radius: 4px;
}

.board-row {
  display: flex;
}

.board-cell {
  width: 64px;
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 45px;
  cursor: pointer;
  transition: all 0.2s ease;
  position: relative;
}

.white-cell {
  background-color: #e8edf9;
}

.black-cell {
  background-color: #4b7399;
}

.white-piece {
  color: #ffffff;
  text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;
}

.black-piece {
  color: #000000;
}

.selected {
  background-color: #687a8f !important;
  box-shadow: inset 0 0 0 2px #ffeb3b;
}

.move-indicator {
  position: absolute;
  width: 16px;
  height: 16px;
  background-color: rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  pointer-events: none;
}

.valid-move.board-cell:hover {
  background-color: rgba(255, 235, 59, 0.3);
}

.control-panel {
  width: 100%;
  max-width: 600px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-top: 20px;
}

.history-controls {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  margin-top: 0;
  background-color: #1a1a1a;
  padding: 10px;
  border-radius: 8px;
}

.history-btn {
  background-color: #3a3a3a;
  color: #ffffff;
  border: 1px solid #4a4a4a;
  border-radius: 4px;
  padding: 8px 12px;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 14px;
  min-width: 80px;
  height: 36px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  flex-shrink: 0;
}

.history-btn:hover:not(:disabled) {
  background-color: #4a4a4a;
  border-color: #5a5a5a;
}

.history-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.move-counter {
  color: #ffffff;
  font-family: monospace;
  padding: 8px 16px;
  background-color: #3a3a3a;
  border-radius: 4px;
  min-width: 100px;
  text-align: center;
  height: 36px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.history-note {
  font-size: 0.8em;
  opacity: 0.8;
  margin-left: 8px;
}

.reset-btn {
  background-color: #b71c1c;
  color: #ffffff;
  border: none;
  border-radius: 4px;
  padding: 12px;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 16px;
  font-weight: bold;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.reset-btn:hover {
  background-color: #d32f2f;
}

.fen-display {
  margin-top: 10px;
}

.fen-label {
  color: #ffffff;
  font-weight: bold;
  font-size: 1.1em;
  white-space: nowrap;
}

.fen-input {
  flex-grow: 1;
  background-color: #3a3a3a;
  color: #ffffff;
  font-family: monospace;
  padding: 8px 12px;
  border: 1px solid #4a4a4a;
  border-radius: 4px;
  font-size: 14px;
  outline: none;
  transition: border-color 0.2s;
}

.fen-input:focus {
  border-color: #687a8f;
}

.fen-input:hover {
  border-color: #5a5a5a;
}

.right-panels {
  display: flex;
  flex-direction: column;
  gap: 20px;
  width: 400px;
}

.config-panel {
  background-color: #1a1a1a;
  border-radius: 8px;
  padding: 15px;
}

.config-panel h2 {
  color: #ffffff;
  margin: 0 0 15px 0;
  font-size: 1.2em;
  text-align: center;
}

.config-form {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.config-field {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.config-field label {
  color: #ffffff;
  font-size: 0.9em;
  font-weight: bold;
}

.config-input {
  background-color: #3a3a3a;
  color: #ffffff;
  font-family: monospace;
  padding: 8px 12px;
  border: 1px solid #4a4a4a;
  border-radius: 4px;
  font-size: 14px;
  outline: none;
  transition: border-color 0.2s;
  width: 100%;
}

.config-input:focus {
  border-color: #687a8f;
}

.config-input:hover {
  border-color: #5a5a5a;
}

.fen-history-panel {
  width: auto;
}

.fen-history-panel h2 {
  color: #ffffff;
  margin: 0 0 15px 0;
  font-size: 1.2em;
  text-align: center;
}

.fen-history-list {
  flex: 1;
  overflow-y: auto;
  max-height: 600px;
}

.fen-history-item {
  background-color: #3a3a3a;
  border-radius: 4px;
  padding: 10px;
  margin-bottom: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.fen-history-item:hover {
  background-color: #4a4a4a;
}

.fen-history-item.active {
  background-color: #687a8f;
  border: 1px solid #ffeb3b;
}

.move-number {
  color: #ffffff;
  font-weight: bold;
  margin-bottom: 4px;
}

.fen-text {
  color: #cccccc;
  font-family: monospace;
  font-size: 0.9em;
  word-break: break-all;
}

/* Scrollbar styling */
.fen-history-list::-webkit-scrollbar {
  width: 8px;
}

.fen-history-list::-webkit-scrollbar-track {
  background: #2f2f2f;
  border-radius: 4px;
}

.fen-history-list::-webkit-scrollbar-thumb {
  background: #4a4a4a;
  border-radius: 4px;
}

.fen-history-list::-webkit-scrollbar-thumb:hover {
  background: #5a5a5a;
}

.ai-controls {
  margin-top: 10px;
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.ai-controls-row {
  display: flex;
  align-items: center;
  gap: 12px;
}

.ai-move-btn {
  background-color: #2196f3;
  color: #ffffff;
  border: none;
  border-radius: 4px;
  padding: 12px;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 16px;
  font-weight: bold;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.ai-move-btn:hover:not(:disabled) {
  background-color: #1976d2;
}

.ai-move-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.loading-indicator {
  animation: ellipsis 1.4s infinite;
}

.ai-error {
  color: #ff5252;
  font-size: 0.9em;
  text-align: center;
  padding: 8px;
  background-color: rgba(255, 82, 82, 0.1);
  border-radius: 4px;
}

@keyframes ellipsis {
  0% { content: '.'; }
  33% { content: '..'; }
  66% { content: '...'; }
}

.settings-buttons {
  display: flex;
  gap: 10px;
  margin-top: 10px;
}

.save-settings-btn,
.clear-settings-btn {
  flex: 1;
  padding: 10px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.save-settings-btn {
  background-color: #4CAF50;
  color: white;
}

.save-settings-btn:hover:not(:disabled) {
  background-color: #45a049;
}

.clear-settings-btn {
  background-color: #f44336;
  color: white;
}

.clear-settings-btn:hover:not(:disabled) {
  background-color: #da190b;
}

.save-settings-btn:disabled,
.clear-settings-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.settings-error {
  color: #ff5252;
  font-size: 0.9em;
  text-align: center;
  padding: 8px;
  background-color: rgba(255, 82, 82, 0.1);
  border-radius: 4px;
  margin-top: 8px;
}

.settings-warning {
  color: #ffd700;
  font-size: 0.9em;
  text-align: center;
  padding: 8px;
  background-color: rgba(255, 215, 0, 0.1);
  border-radius: 4px;
  margin-top: 8px;
}

.model-selector {
  display: flex;
  gap: 8px;
  align-items: center;
}

.model-selector select {
  flex: 1;
  appearance: none;
  background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6 9 12 15 18 9'%3e%3c/polyline%3e%3c/svg%3e");
  background-repeat: no-repeat;
  background-position: right 8px center;
  background-size: 16px;
  padding-right: 32px;
}

.refresh-models-btn {
  background-color: #3a3a3a;
  color: #ffffff;
  border: 1px solid #4a4a4a;
  border-radius: 4px;
  padding: 8px;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 16px;
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.refresh-models-btn:hover:not(:disabled) {
  background-color: #4a4a4a;
  border-color: #5a5a5a;
}

.refresh-models-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.loading-text {
  color: #ffffff;
  font-size: 0.9em;
  margin-top: 4px;
}

.model-error {
  color: #ff5252;
  font-size: 0.9em;
  margin-top: 4px;
}
</style> 