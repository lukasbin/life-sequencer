<template>
<div class="container" v-if="instrument">
  <div>
    <div class="cells" v-if="cells">
      <div class="row" v-for="(row, index) of cells" :key="index">
        <Cell v-for="cell of row" :key="cell.note" :cell="cell" @toggle="toggle(cell)"/>
      </div>
    </div>
    <div class="description">
      <strong>Conway's Game of Life Sequencer</strong> v0.0.1<br>
      Touch a few cells and click "Run"<br>
      White cells: dead, grey cells: alive, black cells: newborn. A sound is played only when cells are born.<br>
      Made by Lukas Bindeus in 2020. <a href="https://github.com/lukasbin/life-sequencer">Source</a>
    </div>
  </div>
  <div class="right">
    <button v-if="started && !stopped" @click="stopped = true">Stop</button>
    <button v-if="!started" @click="run">Run</button>
    <button v-if="stopped" @click="$emit('reload')">Start over (reload app)</button>
    <span v-if="stopped && completed">Completed!</span>
    <span v-else-if="stopped">Stopped manually</span>
    <span v-if="started || stopped">Iterations: {{ iterations }}</span>
    <span class="label" v-if="starter.length">Pattern</span>
    <div class="thumbnail" v-if="starter.length">
      <div class="row" v-for="(row, x) of starter" :key="x">
        <span v-for="(cell, y) of row" :key="y">
          <span v-if="cell">■</span>
          <span v-else style="color: white;">□</span>
        </span>
      </div>
    </div>
    <span class="label" v-if="started">Live thumbnail</span>
    <div class="thumbnail" v-if="started">
      <div class="row" v-for="(row, index) of cells" :key="index">
        <span v-for="cell of row" :key="cell.note">
          <span v-if="cell.active">■</span>
          <span v-else style="color: white;">□</span>
        </span>
      </div>
    </div>
  </div>
</div>
</template>
<script>
import Cell from './Cell'
import Soundfont from 'soundfont-player'

const ctx = new AudioContext()

export default {
  name: 'Container',
  components: { Cell },
  data: () => ({
    scale: ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'],
    firsts: ['C#6', 'G#5', 'D#5', 'A#4', 'F4', 'C4', 'G3', 'D3', 'A2', 'E2', 'B1', 'F#1', 'C#1', 'G#0', 'D#0'],
    cells: null,
    starter: [],
    rowLength: 25,
    instrument: null,
    iterations: 0,
    started: false,
    stopped: false,
    completed: false
  }),
  computed: {
    activeCells () {
      const arr = []
      for (const row of this.cells) {
        for (const cell of row) {
          if (cell.active) arr.push(cell)
        }
      }
      return arr
    },
    activeNotes () {
      return Array.from(new Set(this.activeCells.filter(c => !c.silent && c.voiced).map(c => c.note)))
    },
    silentInactiveCells () {
      const arr = []
      for (const row of this.cells) {
        for (const cell of row) {
          if (!cell.active && cell.silent) arr.push(cell)
        }
      }
      return arr
    },
    allCellsDead () {
      for (const row of this.cells) {
        for (const cell of row) {
          if (cell.active) {
            return false
          }
        }
      }
      return true
    }
  },
  async mounted () {
    this.instrument = await Soundfont.instrument(ctx, 'acoustic_grand_piano')
    this.cells = this.createCells()
  },
  methods: {
    createCells () {
      const cells = []
      for (let i = 0; i < this.firsts.length; i += 1) { 
        let row = []
        const first = this.firsts[i]
        const firstWithoutOctave = first.slice(0, -1)
        let octave = first.slice(-1)
        const index = this.scale.indexOf(firstWithoutOctave)
        for (let j = 0; j <= this.rowLength; j += 1) {
          const newIndex = (index + j) % 12
          const newOctave = parseInt(octave) + Math.floor((index + j) / 12)
          const voiced = true
          row.push({ active: false, voiced, silent: false, isC: this.scale[newIndex] === 'C', note: this.scale[newIndex] + newOctave, x: i, y: j })
        }
        cells.push(row)
      }
      return cells 
    },
    toggle (cell) {
      if (!this.started) {
        cell.active = !cell.active
      }
      this.play(cell)
    },
    async run () {
      // Create copy of cells for starter image
      for (const row of this.cells) {
        const arr = []
        for (const cell of row) {
          arr.push(cell.active ? true : false)
        }
        this.starter.push(arr)
      }
      this.started = true
      while (!this.stopped) {
        if (this.stopped) return
        if (this.allCellsDead) {
          this.completed = true
          this.stopped = true
          /*
          for (const row of this.cells) {
            for (const cell of row) {
              cell.active 
            }
          }
          */
          return
        }
        this.iterations += 1
        for (const cell of this.activeCells) {
          this.play(cell)
          cell.silent = true
        }
        this.getNextGen()
        await new Promise(resolve => setTimeout(resolve, 400))
      }
    },
    play (cell) {
      if (cell.silent || !cell.voiced) return
      this.instrument.play(cell.note, ctx.currentTime, { duration: 0.5 })
    },
    getNextGen() {
      const kill = []
      const birth = []
      // Calculate neighbors of all live cells
      let allNeighbors = new Set()
      for (const cell of this.activeCells) {
        this.findNeighbors(cell.x, cell.y).forEach(allNeighbors.add, allNeighbors)
        cell.silent = false
      }
      // Separate into dead/live neighbors
      const deadNeighbors = []
      const liveNeighbors = []
      for (const cell of allNeighbors) {
        if (cell.active) liveNeighbors.push(cell)
        else deadNeighbors.push(cell)
      }
      // Check live neighbor count of dead neighbors
      for (const cell of deadNeighbors) {
        // Rule 1: Dead cells with exactly three living neighbors are reborn
        if (this.countLiveNeighbors(cell) === 3) {
          birth.push(cell)
        }
      }

      for (const cell of liveNeighbors) {
        const count = this.countLiveNeighbors(cell) 
        // Rule 2: Live cells with less than two living neighbors die of loneliness
        // Rule 3: A living cell with two or three living neighbors remains alive (but sound should not be triggered)
        // Rule 4: Live cells with more than three living neighbors die of overpopulation
        if ([2, 3].includes(count)) {
          /* cell.silent = true */
          birth.push(cell)
        } else {
          kill.push(cell)
        }
      }

      for (const cell of this.silentInactiveCells) {
        cell.silent = false 
      }

      for (const cell of this.activeCells) {
        cell.active = false
        cell.silent = true
      }

      for (const cell of kill) {
        cell.active = false
        cell.silent = false
      }

      for (const cell of birth) {
        cell.active = true
      }
    },
    findNeighbors (x, y) {
      let criteria = [[0, 1], [1, 0], [0, -1], [-1, 0], [-1, -1], [-1, 1], [1, 1], [1, -1]]
      return criteria.filter(([h, j]) => h + x >= 0 && h + x < this.cells.length && j + y >= 0 && j + y < this.cells[0].length)
        .map(([h, j]) => this.cells[h + x][j + y])
    },
    countLiveNeighbors (cell) {
      return this.findNeighbors(cell.x, cell.y).filter(n => n && n.active).length
    }
  }
}
</script>
<style scoped>
.container {
  display: flex;
  flex-flow: row;
}
.cells {
  border-right: 1px solid #303;
  border-top: 1px solid #303;
}
.row {
  display: flex;
  flex-flow: row;
}
.description {
  color: #303;
}
.thumbnail {
  font-family: monospace;
  font-size: 15px;
  letter-spacing: 0px;
  line-height: 9px;
  padding-top: 3px;
  border: 1px solid black;
}
.right {
  display: flex;
  flex-flow: column;
  margin-left: 1em;
}
.right > * {
  margin-bottom: 1em;
}
.label {
  margin-bottom: 0;
}
.description {
  margin-top: 0.5em;
}
button {
  background: transparent;
  padding: 0.5em 1.5em;
  border-radius: 3px;
  border: 1px solid #303;
  color: #303;
}
button:hover {
  cursor: pointer;
  background: white;
}
</style>
