<script setup>
import { ref, reactive, computed, onMounted } from 'vue'

const { value: point } = ref([0, 0])
const markedList = reactive(new Map()) // 标注为🚩的格子
const openList = reactive(new Map()) // 打开的格子
const grid = ref([]) // 棋盘
const time = ref('') // 计时器
const curDiff = ref(0) // 当前难度
let num = 0

const state = reactive({
  playing: false,
  isOver: false,
  isWin: null
})

const Game = {
  start() {
    if (state.playing) return
    this.init()
    state.playing = true
    this.timeing.start()
  },
  init() {
    state.playing = false
    state.isOver = false
    state.isWin = null
    this.timeing.end().clear()
    openList.clear()
    markedList.clear()
    return this
  },
  over() {
    state.isWin = false
  },
  win() {
    state.isWin = true
  },
  end() {
    if (!state.playing) return
    state.playing = false
    state.isOver = true
    this.timeing.end()
    return this
  },
  timeing: {
    timer: null,
    start() {
      if (this.timer) this.end()
      time.value = 0
      this.timer = setInterval(() => {
        time.value++
      }, 1000)
    },
    end() {
      this.timer && clearInterval(this.timer)
      return this
    },
    clear() {
      time.value = ''
    }
  }
}

const landmines = new Map()

function randomPoint(max) {
  return [parseInt(Math.random() * max), parseInt(Math.random() * max)]
}

function randomPointInGrid(max, count) {
  landmines.clear()
  for (let i = 0; i < count; i++) {
    const minesPoint = randomPoint(max)
    const key = '' + minesPoint[0] + minesPoint[1]
    if (landmines.has(key)) {
      i--
      continue
    }
    landmines.set(key, minesPoint)
  }
}

function fillGridWithLandmines() {
  randomPointInGrid(num, num)
  
  landmines.forEach(function(minePoint) {
    grid.value[minePoint[0]][minePoint[1]] = 'M'
  })
}

// 棋盘数量
const diffs = [10, 15, 30]

const isMostDiff = computed(() => curDiff.value === (diffs.length - 1))

function switchGrid(type) {
  Game.init()
  curDiff.value = type
  num = diffs[type]
  point[0] = point[1] = 0

  grid.value.length = 0
  const arr = new Array(Math.pow(num, 2))
  while (arr.length) {
    grid.value.push(arr.splice(0, num))
  }

  fillGridWithLandmines()
}

const Move = {
  keys: ['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'],
  ArrowUp() {
    if (point[0] === 0 && point[1] !== 0) {
      point[0] = num - 1
      point[1]--
      return
    }
    (point[0] !== 0) && point[0]--
  },
  ArrowDown() {
    if (point[0] === num - 1 && point[1] !== num - 1) {
      point[0] = 0
      point[1]++
      return
    }
    (point[0] < num - 1) && point[0]++
  },
  ArrowLeft() {
    if (point[1] === 0 && point[0] !== 0) {
      point[1] = num - 1
      point[0]--
      return
    }
    (point[1] !== 0) && point[1]--
  },
  ArrowRight() {
    if (point[1] === num - 1 && point[0] !== num - 1) {
      point[1] = 0
      point[0]++
      return
    }
    (point[1] < num - 1) && point[1]++
  }
}

function move(key) {
  if (!num) return
  if (!Move.keys.includes(key.key)) return
  Move[key.key]()
}

// function getRandomNumber(min ,max) {
//   return Math.floor(Math.random() * (max - min + 1)) + min
// }

// 搜索附近地雷 简单逻辑  有很大优化空间
function search([cx, cy]) {
  const dx = [1, 1, 1, -1, -1, -1, 0, 0]
  const dy = [1, 0, -1, 1, 0, -1, 1, -1]
  const isBound = (x, y) => x >= 0 && y >= 0 && x < grid.value.length && y < grid.value[0].length

  // 限制打开格子数量（如果不限制  首次会打开大部分安全格子
  // 🌚 当前会斜着打开格子
  // let max = getRandomNumber(1, 5)
  // let searchCount = 0

  const update = (x, y) => {
    // if (searchCount === max) return

    const key = '' + x + y
    if (!isBound(x, y) || openList.has(key)) return

    let count = 0
    for (let i = 0; i < 8; i++) {
      const nx = x + dx[i]
      const ny = y + dy[i]
      if (isBound(nx, ny) && grid.value[nx][ny] === 'M') {
        count++
      }
    }

    // 未查询到地雷
    if (!count) {
      openList.set(key, [x, y])
      for (let i = 0; i < 8; i++) {
        update(x + dx[i], y + dy[i])
      }
    } else {
      // searchCount++
      grid.value[x].splice(y, 1, count) // 标注附近地雷数量
      openList.set(key, [x, y]) // 打开格子
    }
  }

  update(cx, cy)
  return grid
}

function check(row, col) {
  if (!state.playing) return

  const key = '' + row + col
  if (markedList.has(key) || openList.has(key)) return
  if (landmines.has(key)) {
    Game.end().over()
    return
  }

  search([row, col])
  if (openList.size === grid.value.length * grid.value[0].length - landmines.size) {
    Game.end().win()
  }
}

function mark(row, col) {
  if (!state.playing) return

  const key = '' + row + col
  if (markedList.has(key)) {
    markedList.delete(key)
    return
  }
  markedList.set(key, [row, col])
}

function isMarked(row, col) {
  const key = '' + row + col
  return !!markedList.has(key)
}

function isSafe(row, col) {
  const key = '' + row + col
  return !!openList.has(key)
}

function getContent(row, col) {
  const val = grid.value[row][col]
  if (!state.isOver) {
    return val === 'M' ? '' : val
  }
  return val === 'M' ? (state.isWin ? '' : '💩') : val
}

function restart(isMore) {
  if (isMore) curDiff.value++

  switchGrid(curDiff.value)
  Game.start()
}

onMounted(function() {
  switchGrid(curDiff.value)
})
</script>

<template>
  <div class="btn-group">
    <p>
      <a @click.prevent="switchGrid(0)">Small</a>
      <a @click.prevent="switchGrid(1)">Mid</a>
      <a @click.prevent="switchGrid(2)">Large</a>
    </p>
    
    <p>
      <a v-show="!state.playing" @click.prevent="Game.start">Start</a>
      <a v-show="state.playing" @click.prevent="Game.end">End</a>
    </p>

    <p v-if="state.playing || state.isOver" class="time">{{ time }}</p>
  </div>

  <div class="battlefield">
    <table v-if="grid.length" class="table" border="1" :style="{ '--num': num }">
      <tr v-for="(row, rowI) in grid" :key="'row_' + rowI" class="row">
        <td
          v-for="(col, colI) in row"
          :key="'col_' + colI"
          class="col"
          @click="check(rowI, colI)"
          @click.right.prevent="mark(rowI, colI)"
        >
          <div
            class="col-box"
            :class="{ marked: isMarked(rowI, colI), safe: isSafe(rowI, colI) }"
          >{{ getContent(rowI, colI) }}</div>
        </td>
      </tr>
    </table>

    <!-- 结算窗口 -->
    <div v-if="state.isOver" :class="['game-end-box', state.isWin ? 'is-win' : 'is-lose']">
      <h1>{{ state.isWin ? '🥳' : '☹️' }}</h1>

      <p class="game-end-title">{{ state.isWin ? 'You Win！！！' : 'You lose.' }}</p>
      <p>
        <a v-if="!state.isWin" @click.prevent="restart(false)">Try again</a>
        <a v-else-if="!isMostDiff" @click.prevent="restart(true)">Try a higher difficulty</a>
        <a v-else @click.prevent="restart(false)">Wow! It's already the highest difficulty</a>
      </p>
    </div>
  </div>
</template>

<style scoped>
a {
  padding: 5px;
  color: #333;
  cursor: pointer;
  text-decoration: underline;
}

.btn-group {
  padding: 2em 0;
}
.time {
  padding-top: 10px;
}
.table {
  width: 300px;
  height: 300px;
  border-collapse: collapse;
  border-spacing: 0;
}
.table .col {
  border: 1px solid;
  transition: all .1s;
  padding: 0;
  background: #ccc;
  user-select: none;
}
.table .col:hover {
  background: #ededed;
}
.table .col .col-box {
  width: calc((300px - 11px) / var(--num));
  height: calc((300px - 11px) / var(--num));
  user-select: none;
}
.table .col .col-box.safe {
  background: #fff;
}
.table .col .col-box.marked {
  position: relative;
  user-select: none;
}
.table .col .col-box.marked::before {
  content: '🚩';
  display: block;
  width: 100%;
  height: 100%;
  position: absolute;
}

.battlefield {
  position: relative;
  width: 300px;
  height: 300px;
  overflow: hidden;
}
.battlefield .game-end-box {
  display: flex;
  flex-direction: column;
  -ms-flex-direction: column;
  justify-content: center;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  margin: auto;
  background: rgba(255, 255, 255, .7);
  z-index: 2;
}
.game-end-box h1 {
  margin: 0;
}
.game-end-title {
  font-weight: normal;
  font-size: 24px;
  margin: 10px 0 0;
}
.game-end-box.is-win .game-end-title {
  color: #0c7d57;
}
.game-end-box.is-lose .game-end-title {
  color: rgba(163. 35, 35);
}
</style>