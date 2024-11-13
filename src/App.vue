<template>
  <div class="app">
    <div class="sidebar" style="width: 400px; text-align:center">
      <div class="lives" style="margin-left:auto; margin-right:auto">
        <span v-for="n in lives" :key="'life-' + n" class="heart">❤️</span>
      </div>
      <div class="game-description">
        <p>Guess the island with the highest average height! You have 3 lives. Click on an island to make a guess.</p>
      </div>
      <div class="result-message">
        <p v-if="gameOver" :class="gameResultClass">{{ gameResultMessage }}</p>
      </div>
      <button style="margin-left:auto; margin-right:auto" v-if="gameOver" @click="restartGame" class="restart-btn">Restart Game</button>
    </div>
    
    <div class="matrix-container">
      <div class="row" v-for="(row, rowIndex) in matrixFinal" :key="'row-' + rowIndex">
        <div class="cell" 
          v-for="(cell, cellIndex) in row" 
          :key="'cell-' + rowIndex + '-' + cellIndex"
          :style="{ backgroundColor: getTerrainColor(cell) }"
          :class="{ 'highlighted': isHighlighted(rowIndex, cellIndex) }"
          @mouseover="highlightIsland(rowIndex, cellIndex)"
          @mouseleave="removeHighlight"
          @click="pickIsland(rowIndex, cellIndex)">
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref } from 'vue';
import chroma from 'chroma-js';

// Matrix and Islands
const matrixFinal = ref([]);
const islandCells = ref([]); // Store island coordinates
const highlightedIsland = ref<Set<string>>(new Set()); // Highlighted island cells
const lives = ref(3); // Number of lives (hearts)
const gameOver = ref(false); // Game over status
const gameResultMessage = ref(''); // Message to show when game ends
const gameResultClass = ref(''); // Class to style the result message

// Function to map terrain values to colors
function getTerrainColor(value: number): string {
  const scale = chroma.scale(['blue', '#07b893', '#8fe480', '#c9bf7f', '#84655c', '#ecece4'])
    .domain([0, 200, 500, 800, 1000]);
  return scale(value).hex();
}

// Calculate and check the picked island
function pickIsland(row: number, col: number) {
  if (gameOver.value) return; // Prevent actions if game is over
  
  const islandId = islandCells.value.findIndex(island => island.includes(cellId(row, col)));
  
  if (islandId !== -1) {
    const island = islandCells.value[islandId];

    const islandHeights = island.map(cell => {
      const [r, c] = cell.split('-').map(Number);
      return matrixFinal.value[r][c];
    });

    const averageHeight = islandHeights.reduce((sum, height) => sum + height, 0) / islandHeights.length;

    const highestIsland = islandCells.value.reduce((highest, currentIsland) => {
      const currentIslandHeights = currentIsland.map(cell => {
        const [r, c] = cell.split('-').map(Number);
        return matrixFinal.value[r][c];
      });
      const currentIslandAvgHeight = currentIslandHeights.reduce((sum, height) => sum + height, 0) / currentIslandHeights.length;
      
      if (currentIslandAvgHeight > highest.avgHeight) {
        return { avgHeight: currentIslandAvgHeight, islandId: currentIsland };
      }
      return highest;
    }, { avgHeight: -Infinity, islandId: [] });

    if (averageHeight === highestIsland.avgHeight) {
      console.log(`YES, Height: ${averageHeight}`);
      gameResultMessage.value = `Congratulations! You guessed correctly.`;
      gameResultClass.value = 'win';
      gameOver.value = true;
    } else {
      console.log(`NO, Average Height: ${averageHeight}`);
      decreaseLife(); // Decrease life if the guess is incorrect
    }
  }
}

// Decrease a life when the guess is incorrect
function decreaseLife() {
  if (lives.value > 0) {
    lives.value--;
  }

  if (lives.value === 0) {
    gameResultMessage.value = 'Game Over! You ran out of lives.';
    gameResultClass.value = 'lose';
    gameOver.value = true;
  }
}

// Helper function to generate a unique ID for a cell
function cellId(row: number, col: number): string {
  return `${row}-${col}`;
}

// Find and group islands in the matrix
function findIslands(matrix) {
  const rows = matrix.length;
  const cols = matrix[0].length;
  const visited = Array.from({ length: rows }, () => Array(cols).fill(false));
  const islands = [];

  function dfs(x, y, islandId) {
    if (x < 0 || x >= rows || y < 0 || y >= cols || matrix[x][y] === 0 || visited[x][y]) return;
    visited[x][y] = true;
    islands[islandId].push(cellId(x, y));

    const directions = [
      [1, 0], [-1, 0], [0, 1], [0, -1]
    ];
    for (const [dx, dy] of directions) {
      dfs(x + dx, y + dy, islandId);
    }
  }

  let islandId = 0;
  for (let i = 0; i < rows; i++) {
    for (let j = 0; j < cols; j++) {
      if (matrix[i][j] !== 0 && !visited[i][j]) {
        islands.push([]);
        dfs(i, j, islandId);
        islandId++;
      }
    }
  }

  return islands;
}

// Parse the matrix data
function parseMatrixData(dataString) {
  const dataArray = dataString.trim().split(/\s+/).map(Number);
  if (dataArray.length !== 900) {
    throw new Error('Data does not contain exactly 900 elements.');
  }

  const matrix = [];
  for (let i = 0; i < 900; i += 30) {
    matrix.push(dataArray.slice(i, i + 30));
  }

  return matrix;
}

// Fetch the matrix data
async function getMatrixData() {
  const targetUrl = 'https://cors-anywhere.herokuapp.com/https://jobfair.nordeus.com/jf24-fullstack-challenge/test';
  try {
    const response = await fetch(targetUrl);
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const responseBody = await response.text();
    const matrix = parseMatrixData(responseBody);
    matrixFinal.value = matrix;
    islandCells.value = findIslands(matrix);
  } catch (error) {
    console.error('Error fetching matrix data:', error);
  }
}

// Highlight the island cells when hovered over
function highlightIsland(row: number, col: number) {
  const islandId = islandCells.value.findIndex(island => island.includes(cellId(row, col)));
  if (islandId !== -1) {
    islandCells.value[islandId].forEach(cell => highlightedIsland.value.add(cell));
  }
}

// Remove highlight when mouse leaves
function removeHighlight() {
  highlightedIsland.value.clear();
}

// Check if the cell is highlighted
function isHighlighted(row: number, col: number) {
  return highlightedIsland.value.has(cellId(row, col));
}

// Restart the game
function restartGame() {
  lives.value = 3;
  gameOver.value = false;
  gameResultMessage.value = '';
  gameResultClass.value = '';
  getMatrixData(); // Fetch new matrix data
}

getMatrixData();
</script>

<style>
.app {
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 100vh;
  font-family: 'Arial', sans-serif;
  background: linear-gradient(135deg, #f1f1f1, #c9c9c9); /* Subtle gradient background */
}

.sidebar {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  padding: 20px;
  max-width: 250px;
}

.lives {
  display: flex;
  margin-bottom: 20px;
  font-size: 1.5rem;
}

.heart {
  margin-right: 10px;
}

.game-description {
  margin-bottom: 20px;
  font-size: 1.2rem;
}

.result-message {
  font-size: 1.3rem;
  margin-bottom: 20px;
}

.win {
  color: green;
}

.lose {
  color: red;
}

.restart-btn {
  padding: 10px;
  background-color: #2e8b57;
  color: white;
  border: none;
  cursor: pointer;
  border-radius: 5px;
}

.matrix-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 20px;
}

.row {
  display: flex;
}

.cell {
  width: 30px;
  height: 30px;
  border: 1px solid #ccc;
  transition: background-color 0.3s ease;
}

.highlighted {
  border: 2px solid yellow;
}

.selected {
  border: 2px solid red;
}
</style>
