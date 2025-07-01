<template>
  <div class="background">
    <div class="overlay"></div>
    <div class="main-content">
      <h1 class="title">Tic Tac Toe</h1>
      <div class="board">
        <div
          v-for="(row, rowIndex) in 3"
          :key="rowIndex"
          class="board-row"
        >
          <button
            v-for="(col, colIndex) in 3"
            :key="colIndex"
            class="square"
            :disabled="gameOver || !!board[rowIndex * 3 + colIndex]"
            @click="makeMove(rowIndex * 3 + colIndex)"
            :aria-label="'Cell ' + (rowIndex * 3 + colIndex + 1)"
          >
            <span :class="{
              x: board[rowIndex * 3 + colIndex] === 'X',
              o: board[rowIndex * 3 + colIndex] === 'O'
            }">
              {{ board[rowIndex * 3 + colIndex] }}
            </span>
          </button>
        </div>
      </div>
      <div class="controls">
        <label for="difficulty" class="difficulty-label">Difficulty:</label>
        <select
          id="difficulty"
          v-model="difficulty"
          :disabled="moveHistory.length > 1"
          @change="resetGame"
        >
          <option value="easy">Easy</option>
          <option value="medium">Medium</option>
          <option value="hard">Hard</option>
        </select>
        <button
          class="action-btn"
          @click="undoMove"
          :disabled="moveHistory.length <= (isHumanFirst ? 1 : 2) || gameOver"
        >
          Undo
        </button>
        <button class="action-btn" @click="resetGame">
          Reset
        </button>
      </div>
      <div class="status">
        <span v-if="winner === null && !gameOver">
          Turn: 
          <span :class="boardClass">{{ isHumanTurn ? 'You (X)' : 'Computer (O)' }}</span>
        </span>
        <span v-else-if="winner === 'draw'"><b>Draw!</b></span>
        <span v-else-if="winner === 'X'"><b>You Win!</b></span>
        <span v-else-if="winner === 'O'"><b>Computer Wins!</b></span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
// PUBLIC_INTERFACE
/**
 * Main page component for the Tic Tac Toe game.
 * Features: human vs. AI play, difficulty levels (easy/medium/hard),
 * undo, reset, modern styling with a local background image and overlay.
 */
import { ref, computed, watch, onMounted } from 'vue'

const board = ref<(string | null)[]>(Array(9).fill(null))
const moveHistory = ref<Array<(string | null)[]>>([Array(9).fill(null)])
const currentMove = ref(0)

const winner = ref<null | string>('') // 'X', 'O', 'draw', or null
const difficulty = ref<'easy' | 'medium' | 'hard'>('medium')

const isHumanFirst = true // always X is human
const isHumanTurn = computed(() => {
  // Even move = X (human) turn
  return currentMove.value % 2 === 0
})

const gameOver = computed(() => winner.value === 'X' || winner.value === 'O' || winner.value === 'draw')

const boardClass = computed(() => ({
  'status-x': isHumanTurn.value,
  'status-o': !isHumanTurn.value,
}))

watch(
  () => moveHistory.value,
  () => {
    board.value = [...moveHistory.value[currentMove.value]]
  },
  { deep: true }
)

watch(currentMove, () => {
  board.value = [...moveHistory.value[currentMove.value]]
})

// AI Play after human move
watch(
  () => board.value,
  () => {
    winner.value = getWinner(board.value)
    if (!isHumanTurn.value && !gameOver.value) {
      setTimeout(() => {
        aiPlay()
      }, 350)
    }
  },
  { immediate: false }
)

// PUBLIC_INTERFACE
function makeMove(idx: number) {
  if (board.value[idx] || gameOver.value || !isHumanTurn.value) return
  const newBoard = [...board.value]
  newBoard[idx] = 'X'
  moveHistory.value = moveHistory.value.slice(0, currentMove.value + 1).concat([newBoard])
  currentMove.value++
  board.value = newBoard
  winner.value = getWinner(board.value)
}

function aiPlay() {
  if (gameOver.value) return
  const aiMove = getAIMove(board.value, difficulty.value)
  if (aiMove !== null) {
    const newBoard = [...board.value]
    newBoard[aiMove] = 'O'
    moveHistory.value = moveHistory.value.slice(0, currentMove.value + 1).concat([newBoard])
    currentMove.value++
    board.value = newBoard
    winner.value = getWinner(board.value)
  }
}

// PUBLIC_INTERFACE
function undoMove() {
  if (
    moveHistory.value.length > (isHumanFirst ? 1 : 2) &&
    currentMove.value > 0 &&
    !gameOver.value
  ) {
    // If it was computer's turn, undo two moves (AI + human's move)
    const stepsBack = isHumanTurn.value ? 2 : 1
    currentMove.value = Math.max(0, currentMove.value - stepsBack)
    board.value = [...moveHistory.value[currentMove.value]]
    winner.value = getWinner(board.value)
    moveHistory.value = moveHistory.value.slice(0, currentMove.value + 1)
  }
}

// PUBLIC_INTERFACE
function resetGame() {
  board.value = Array(9).fill(null)
  moveHistory.value = [Array(9).fill(null)]
  currentMove.value = 0
  winner.value = null
}

onMounted(() => {
  resetGame()
})

// --------- GAME WINNER DETECTION ---------
function getWinner(brd: (string|null)[]) : 'X'|'O'|'draw'|null {
  const wins = [
    [0,1,2], [3,4,5], [6,7,8],
    [0,3,6], [1,4,7], [2,5,8],
    [0,4,8], [2,4,6],
  ]
  for (const [a,b,c] of wins) {
    if (brd[a] && brd[a] === brd[b] && brd[a] === brd[c]) {
      return brd[a] as 'X'|'O'
    }
  }
  if (brd.every(cell => cell)) return 'draw'
  return null
}

// --------- AI LOGIC ---------
/**
 * Returns next AI move index.
 * PUBLIC_INTERFACE
 */
function getAIMove(brd: (string|null)[], level: string): number|null {
  const available = brd.map((cell, idx) => cell ? null : idx).filter(idx => idx !== null) as number[]
  if (available.length === 0) return null

  // EASY - random move
  if (level === 'easy') {
    return available[Math.floor(Math.random() * available.length)]
  }

  // MEDIUM - win or block, otherwise random
  if (level === 'medium') {
    // Try to win
    for (const idx of available) {
      const test = [...brd]; test[idx] = 'O'
      if (getWinner(test) === 'O') return idx
    }
    // Try to block
    for (const idx of available) {
      const test = [...brd]; test[idx] = 'X'
      if (getWinner(test) === 'X') return idx
    }
    // Otherwise random
    return available[Math.floor(Math.random() * available.length)]
  }

  // HARD - Minimax
  let bestScore = -Infinity, bestMove = null
  for (const idx of available) {
    const test = [...brd]; test[idx] = 'O'
    const score = minimax(test, false)
    if (score > bestScore) {
      bestScore = score
      bestMove = idx
    }
  }
  return bestMove
}

// Minimax for unbeatable AI on hard
function minimax(brd: (string|null)[], isMax: boolean): number {
  const win = getWinner(brd)
  if (win === 'O') return 1
  if (win === 'X') return -1
  if (win === 'draw') return 0

  const available = brd.map((cell, idx) => cell ? null : idx).filter(idx => idx !== null) as number[]
  if (isMax) {
    let maxEval = -Infinity
    for (const idx of available) {
      const test = [...brd]; test[idx] = 'O'
      maxEval = Math.max(maxEval, minimax(test, false))
    }
    return maxEval
  } else {
    let minEval = Infinity
    for (const idx of available) {
      const test = [...brd]; test[idx] = 'X'
      minEval = Math.min(minEval, minimax(test, true))
    }
    return minEval
  }
}
</script>

<style scoped>
.background {
  position: fixed;
  top: 0; left: 0;
  width: 100vw; height: 100vh;
  background: url('/bg.jpg') center/cover no-repeat;
  z-index: 0;
  min-height: 100vh;
}
.overlay {
  position: fixed;
  top: 0; left: 0;
  width: 100vw; height: 100vh;
  background-color: rgba(34, 38, 41, 0.63); /* dark primary overlay */
  z-index: 1;
  pointer-events: none;
}
.main-content {
  position: relative;
  z-index: 2;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
.title {
  color: #f8b229;
  font-size: 2.5rem;
  letter-spacing: 1px;
  margin-bottom: 1.5rem;
  font-weight: bold;
  text-shadow: 1px 2px 4px #222629cc;
}
.board {
  display: flex;
  flex-direction: column;
  margin-bottom: 1.3rem;
  background: rgba(246,246,246,0.96);
  border-radius: 1rem;
  box-shadow: 0 8px 32px 0 #22262925;
  padding: 1.2rem;
}
.board-row {
  display: flex;
}
.square {
  width: 72px;
  height: 72px;
  margin: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #fff;
  border-radius: 0.6rem;
  border: 2px solid #222629;
  font-size: 2.1rem;
  font-family: 'Poppins', Arial, sans-serif;
  font-weight: 600;
  color: #222629;
  transition: background 0.18s, border 0.22s;
  outline: none;
}
.square:disabled {
  background: #6ba36822;
  border-color: #6ba36877;
  cursor: not-allowed;
}
.square .x {
  color: #f8b229;
  text-shadow: 0 1px 2px #22262944;
}
.square .o {
  color: #6ba368;
}
.controls {
  margin-top: 0.5rem;
  display: flex;
  gap: 0.8rem;
  align-items: center;
  flex-wrap: wrap;
}
.difficulty-label {
  color: #222629;
  font-weight: 600;
  margin-right: 0.2rem;
}
select {
  background: #f8b229;
  color: #222629;
  border: 1px solid #f8b229;
  border-radius: 4px;
  font-weight: 600;
  padding: 0.33em 0.6em;
  margin-right: 0.15rem;
  outline: none;
}
.action-btn {
  background: #6ba368;
  color: #fff;
  border: none;
  border-radius: 5px;
  padding: 0.43em 1.2em;
  font-weight: 700;
  font-size: 1rem;
  box-shadow: 0 1px 6px #22262933;
  cursor: pointer;
  transition: background 0.18s, box-shadow 0.2s;
}
.action-btn:disabled {
  background: #22262944;
  color: #ccc;
  cursor: not-allowed;
}
.status {
  margin-top: 1.3rem;
  font-size: 1.3rem;
  color: #222629;
  min-height: 38px;
  font-weight: 500;
  text-align: center;
}
.status-x {
  color: #f8b229;
  font-weight: bold;
}
.status-o {
  color: #6ba368;
  font-weight: bold;
}
@media (max-width: 600px) {
  .main-content, .board {
    padding: 0.48rem;
  }
  .board-row {
    margin: 0 -2px;
    gap: 0;
  }
  .square {
    width: 54px;
    height: 54px;
    font-size: 1.23rem;
    margin: 1.8px;
  }
}
</style>
