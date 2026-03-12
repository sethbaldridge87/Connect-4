<script setup>
import {ref} from 'vue'
// Initial setup of global values
const xAxis = ref([1,2,3,4,5,6,7]);
const yAxis = ref([6,5,4,3,2,1]);
const player1 = ref(true);
const gameOver = ref(false);
const waiting = ref(false);
let remainingTurns = 42;
let playerColor;
let winner;

const dropCoin = (xAxis) => {
  // The function that handles the coin drop, coloring in the corresponding cell and switching the player turn
  player1.value === true ? playerColor = 'playerOne' : playerColor = 'playerTwo';
  const cellsWithxAxis = document.querySelectorAll('td[data-x="' + xAxis +'"]');
  let thisCoin = {};
  let nextPlayer;
  let waitTime;
  waiting.value = true;
  for (let i = cellsWithxAxis.length - 1; i >= 0; i--) {
    const thisCell = cellsWithxAxis[i];
    if (thisCell.classList.length === 0) {
      thisCell.classList.add(playerColor);
      const getPosition = thisCell.getBoundingClientRect();
      const coinDrop = thisCell.querySelector('.coin');
      const topPosition = getPosition.top - 150;
      coinDrop.animate([
        { top: '-' + topPosition + 'px' },
        { top: '10px' }
      ], {
        duration: topPosition, // duration in milliseconds
        iterations: 1 // number of times to repeat, or a number
      });
      thisCoin = {
        x: Number(thisCell.getAttribute('data-x')),
        y: Number(thisCell.getAttribute('data-y'))
      }
      nextPlayer = true;
      waitTime = topPosition;
      break
    }
  }
  setTimeout(function() {
    waiting.value = false;
    // Prevents error when attempting to click a full column
    if (nextPlayer === true) {
      checkBoard(thisCoin, playerColor);
      remainingTurns--;
      player1.value = !player1.value;
    }
    if (remainingTurns === 0) {
      endGame();
    }
  }, waitTime);
}

const streakCheck = (cellGroup, cellColor) => {
  // Here we take an array of cells and check to see if we have 4 in a row of a certain color
  let streakArray = [];
  for (const cell of cellGroup) {
    if (cell.classList.contains(cellColor)) {
      streakArray.push(cell);
    } else {
      streakArray = [];
    }
    if (streakArray.length >= 4) {
      for (const cell of streakArray) {
        cell.classList.add('winner');
      }
      endGame(cellColor);
    }
  }
}

const verticalCheck = (coinData, coinColor) => {
  // Check for vertical win by checking the classes of the 3 coins below this one
  const verticalArray = [];
  for (let i = coinData.y; i > 0; i--) {
    const adjacentCell = document.querySelectorAll('td[data-x="' + coinData.x + '"][data-y="' + i + '"]');
    verticalArray.push(adjacentCell);
    if (verticalArray.length === 4) {
      const allHaveClass = verticalArray.every(element => {
        return element[0].classList.contains(coinColor);
      });
      if (allHaveClass) {
        for (const cell of verticalArray) {
          cell[0].classList.add('winner');
        }
        endGame(coinColor);
      }
    }
  }
}

const horizontalCheck = (coinData, coinColor) => {
  // Check for horizontal win by checking for a streak of classes among the coins in this row
  const thisRow = document.querySelectorAll('tr[data-y="' + coinData.y + '"]');
  const cellArray = [];
  for (const cell of thisRow[0].cells) {
    cellArray.push(cell);
  }
  streakCheck(cellArray, coinColor);
}

const diagonalCheck = (coinData, coinColor) => {
  // These functions get the x and y values of the cells diagonal of the last filled cell and adds them to arrays
  const lowerData = (data, axis) => {
    const array = [];
    for (let i = data; i > 0; i--) {
      const position = {};
      position[axis] = i;
      array.push(position);
    }
    return array;
  }

  const upperData = (data, axis) => {
    const array = [];
    for (let i = data; i <= 7; i++) {
      const position = {};
      position[axis] = i;
      array.push(position);
    }
    return array;
  }
  
  const ascendingDiagonalArrayX1 = lowerData(coinData.x, 'x');
  const ascendingDiagonalArrayY1 = lowerData(coinData.y, 'y');
  const ascendingDiagonalArrayX2 = upperData(coinData.x, 'x');
  const ascendingDiagonalArrayY2 = upperData(coinData.y, 'y');
  
  const descendingDiagonalArrayX1 = lowerData(coinData.x, 'x');
  const descendingDiagonalArrayY1 = upperData(coinData.y, 'y');
  const descendingDiagonalArrayX2 = upperData(coinData.x, 'x');
  const descendingDiagonalArrayY2 = lowerData(coinData.y, 'y');

  // Here we determine the shortest lengths of the x and y data so we can iterate only through affected cells, then combine the x and y data together based on their respective indexes, so they are complete sets of locational data
  const combineCoords = (array1, array2) => {
    let smallestArray;
    let biggestArray;
    if (array1.length <= array2.length) {
      biggestArray = array2;
      smallestArray = array1;
    } else {
      biggestArray = array1;
      smallestArray = array2;
    }
    const dataList = [];
    for (let i = 0; i < smallestArray.length; i++) {
      const coord = {};
      Object.assign(coord, smallestArray[i]);
      Object.assign(coord, biggestArray[i]);
      dataList.push(coord);
    }
    return dataList;
  }

  const dataList1 = combineCoords(ascendingDiagonalArrayX1, ascendingDiagonalArrayY1);
  const dataList2 = combineCoords(ascendingDiagonalArrayX2, ascendingDiagonalArrayY2);
  const dataList3 = combineCoords(descendingDiagonalArrayX1, descendingDiagonalArrayY1);
  const dataList4 = combineCoords(descendingDiagonalArrayX2, descendingDiagonalArrayY2);

  // Removes the duplicate coordinate data of the filled cell from 2 of the arrays
  dataList1.shift();
  dataList3.shift();

  // Sorting these arrays of data ensures that the cells will be arranged in order
  dataList1.sort((a, b) => a.x - b.x);
  dataList3.sort((a, b) => a.x - b.x);

  // We then apply that locational data to the corresponding cells in the table, and get those cells as an array
  const markCells = (dataArray1, dataArray2) => {
    const cellArray = [];
    const targetCells = (currentArray) => {
      for (const cell of currentArray) {
        const diagonalCell = document.querySelector('td[data-x="' + cell.x + '"][data-y="' + cell.y + '"]');
        if (diagonalCell) {
          cellArray.push(diagonalCell);
        }
      }
    }
    targetCells(dataArray1);
    targetCells(dataArray2);
    return cellArray;
  }

  const cellListA = markCells(dataList1, dataList2);
  const cellListB = markCells(dataList3, dataList4);

  // Finally, we check for a 4 coin streak in both diagonal rows
  streakCheck(cellListA, coinColor);
  if (!gameOver.value) {
    streakCheck(cellListB, coinColor);
  }
}

const checkBoard = (coinData, coinColor) => {
  // This function checks all the possible ways in which the last dropped coin might score a streak. Each one will run only if no streak has yet been achieved.
  diagonalCheck(coinData, coinColor);
  if (!gameOver.value) {
    horizontalCheck(coinData, coinColor);
  }
  if (!gameOver.value) {
    verticalCheck(coinData, coinColor);
  }
}

const endGame = (winnerColor) => {
  // This function determines if there was a winner or a tie
  waiting.value = true;
  setTimeout(function() {
    gameOver.value = true;
    if (winnerColor) {
      winner = winnerColor
    }
  }, 2000);
}

const clearBoard = () => {
  // This function clears the board and resets the game
  const allCells = document.querySelectorAll('td');
  allCells.forEach((element) => {
    element.classList.remove('playerOne', 'playerTwo', 'winner');
  });
  waiting.value = false;
  gameOver.value = false;
  player1.value = true;
  winner = '';
  remainingTurns = 42;
}

</script>

<template>
  <main>
    <div v-if="gameOver" class="modal-background"></div>
    <div v-if="gameOver" class="modal">
      <h2>The game is over!
        <span v-if="winner" :class="winner">
          Player 
          <span v-if="winner === 'playerOne'">1</span>
          <span v-else-if="winner === 'playerTwo'">2</span>
          wins!
        </span>
        <span v-else>Tie Game!</span>
      </h2>
      <button v-on:click="clearBoard()">Reset Game</button>
    </div>
    <header>
      <h1>Connect 4</h1>
      <h2>Remaining turns: {{ remainingTurns }}</h2>
      <div class="turn-message">
        <h2 v-if="!waiting" :class="{ 
          'playerOne': player1,
          'playerTwo': !player1
        }">Player 
          <span v-if="player1">1</span>
          <span v-else>2</span>, it's your turn!
        </h2>
      </div>
    </header>
    <table :class="{'disable': waiting}">
      <tbody>
        <tr v-for="yValue in yAxis" :data-y="yValue">
          <td :data-x="xValue" :data-y="yValue" v-for="xValue in xAxis" v-on:click="dropCoin(xValue)">
            <div class="circle">
              <div class="coin"></div>
            </div>
          </td>
        </tr>
      </tbody>
    </table>
  </main>
</template>
