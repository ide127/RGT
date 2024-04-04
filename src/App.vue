<template>
  <div id="app">
    <h1>누가 누가 더 빨리 찾나</h1>
    <div class="controls">
      <div class="controlBox">
        <div class="control">
          <label for="gridWidth">Grid Width:</label>
          <input type="number" id="gridWidth" v-model.number="gridWidth" min="1" />
        </div>
        <div class="control">
          <label for="gridHeight">Grid Height:</label>
          <input type="number" id="gridHeight" v-model.number="gridHeight" min="1" />
        </div>
      </div>
      <div class="controlBox">
        <div class="control">
          <label for="startX">Start X:</label>
          <input type="number" id="startX" v-model.number="startX" min="0" :max="gridWidth - 1" />
        </div>
        <div class="control">
          <label for="startY">Start Y:</label>
          <input type="number" id="startY" v-model.number="startY" min="0" :max="gridHeight - 1" />
        </div>
      </div>
      <div class="controlBox">
        <div class="control">
          <label for="goalX">Goal X:</label>
          <input type="number" id="goalX" v-model.number="goalX" min="0" :max="gridWidth - 1" />
        </div>
        <div class="control">
          <label for="goalY">Goal Y:</label>
          <input type="number" id="goalY" v-model.number="goalY" min="0" :max="gridHeight - 1" />
        </div>
      </div>
      <button @click="createGrid">Update Grid</button>
      <button @click="findPaths">Find Paths</button>
    </div>
    <div class="table-wrapper">
      <div v-for="finder in finders" :key="finder.name" class="grid-wrapper">
        <h3>{{ finder.name }}</h3>
        <!-- <div>{{ finder }}</div> -->
        <div v-if="finder.grid" class="grid-container">
          <table @click="handleCellClick">
            <tr v-for="(row, rowIndex) in finder.grid.nodes" :key="rowIndex">
              <td
                v-for="(cell, colIndex) in row"
                :key="colIndex"
                :class="{
                  obstacle: !cell.walkable,
                  path: isPath(finder, colIndex, rowIndex),
                  start: isStart(colIndex, rowIndex),
                  goal: isGoal(colIndex, rowIndex),
                  process: isProcess(finder, colIndex, rowIndex),
                  layoverPoints: isLayover(colIndex, rowIndex)
                }"
              ></td>
            </tr>
          </table>
        </div>
      </div>
    </div>

    <div class="dashboard">
      <h2>How much ms each robot took?</h2>
      <div id="chart"></div>
    </div>
  </div>
</template>

<script>
import { Grid, IDAStarFinder, Heuristic } from 'pathfinding'
import * as d3 from 'd3'

export default {
  name: 'App',
  data() {
    return {
      gridWidth: 5,
      gridHeight: 5,
      startX: 1,
      startY: 1,
      goalX: 3,
      goalY: 3,
      layoverPoints: [],
      cellStatus: [],
      finders: [
        {
          grid: null,
          name: 'Robot manhattan',
          instance: new IDAStarFinder({
            heuristic: Heuristic.manhattan
          }),
          timeCost: null,
          paths: [],
          processCells: []
        },
        {
          grid: null,
          name: 'Robot euclidean',
          instance: new IDAStarFinder({
            heuristic: Heuristic.euclidean
          }),
          timeCost: null,
          paths: [],
          processCells: []
        },
        {
          grid: null,
          name: 'Robot octile',
          instance: new IDAStarFinder({
            heuristic: Heuristic.octile
          }),
          timeCost: null,
          paths: [],
          processCells: []
        },
        {
          grid: null,
          name: 'Robot chebyshev',
          instance: new IDAStarFinder({
            heuristic: Heuristic.chebyshev
          }),
          timeCost: null,
          paths: [],
          processCells: []
        }
      ],
      results: []
    }
  },
  methods: {
    createGrid() {
      const grid = new Grid(this.gridWidth, this.gridHeight)
      for (const finder of this.finders) {
        finder.grid = grid.clone()
        finder.processCells = []
        finder.paths = []
      }
      this.layoverPoints = []
      this.cellStatus = []
    },
    handleCellClick(event) {
      if (!this.finders[0].grid) return
      const cell = event.target.closest('td')
      if (cell) {
        const rowIndex = cell.parentNode.rowIndex
        const colIndex = cell.cellIndex

        if (
          (rowIndex === this.startX && colIndex === this.startY) ||
          (rowIndex === this.goalX && colIndex === this.goalY)
        ) {
          return
        }

        if (!this.cellStatus[rowIndex]) {
          this.cellStatus[rowIndex] = []
        }
        if (this.cellStatus[rowIndex][colIndex] === undefined) {
          this.cellStatus[rowIndex][colIndex] = 0
        }

        // 셀 상태 사이클 로직
        this.cellStatus[rowIndex][colIndex] = (this.cellStatus[rowIndex][colIndex] + 1) % 3

        for (const finder of this.finders) {
          switch (this.cellStatus[rowIndex][colIndex]) {
            case 1: // Layover
              this.layoverPoints.push({ x: colIndex, y: rowIndex })
              finder.grid.setWalkableAt(colIndex, rowIndex, true)
              break
            case 2: // Obstacle
              this.layoverPoints = this.layoverPoints.filter(
                (point) => point.x !== colIndex || point.y !== rowIndex
              )
              finder.grid.setWalkableAt(colIndex, rowIndex, false)
              break
            case 0: // Nothing
              this.layoverPoints = this.layoverPoints.filter(
                (point) => point.x !== colIndex || point.y !== rowIndex
              )
              finder.grid.setWalkableAt(colIndex, rowIndex, true)
              break
          }
        }
      }
    },
    findPaths() {
      this.results = []
      for (let finder of this.finders) {
        this.findPath(finder)
      }
    },
    findPath(finder) {
      if (!finder.grid) return
      const startTime = performance.now()
      this.visualizeFinder(finder)
      const endTime = performance.now()
      finder.timeCost = endTime - startTime
      this.results.push({ finder: finder.name, time: finder.timeCost })
    },
    visualizeFinder(finder) {
      const startX = this.startX
      const startY = this.startY
      const goalX = this.goalX
      const goalY = this.goalY
      finder.processCells = []
      const finderInstance = finder.instance
      const path = []
      let currentX = startX
      let currentY = startY
      for (const layoverPoint of this.layoverPoints) {
        const partialPath = finderInstance.findPath(
          currentX,
          currentY,
          layoverPoint.x,
          layoverPoint.y,
          finder.grid
        )
        path.push(...partialPath)
        currentX = layoverPoint.x
        currentY = layoverPoint.y
      }
      const finalPath = finderInstance.findPath(currentX, currentY, goalX, goalY, finder.grid)
      path.push(...finalPath)
      finder.paths = path
    },
    isLayover(colIndex, rowIndex) {
      return this.layoverPoints.some(({ x, y }) => x === colIndex && y === rowIndex)
    },
    isPath(finder, colIndex, rowIndex) {
      const path = finder.paths
      return path && path.some(([x, y]) => x === colIndex && y === rowIndex)
    },
    isStart(colIndex, rowIndex) {
      return colIndex === this.startX && rowIndex === this.startY
    },
    isGoal(colIndex, rowIndex) {
      return colIndex === this.goalX && rowIndex === this.goalY
    },
    isProcess(finder, colIndex, rowIndex) {
      return finder.processCells.some(({ x, y }) => x === colIndex && y === rowIndex)
    },

    createBarChart() {
      const margin = { top: 20, right: 20, bottom: 30, left: 40 }
      const width = 400 - margin.left - margin.right
      const height = 300 - margin.top - margin.bottom

      const svg = d3
        .select('#chart')
        .append('svg')
        .attr('width', width + margin.left + margin.right)
        .attr('height', height + margin.top + margin.bottom)
        .append('g')
        .attr('transform', `translate(${margin.left}, ${margin.top})`)

      const maxTime = d3.max(this.results, (d) => d.time)
      const x = d3
        .scaleBand()
        .range([0, width])
        .padding(0.1)
        .domain(this.results.map((d) => d.finder))

      const y = d3.scaleLinear().range([height, 0]).domain([0, maxTime])

      svg.append('g').attr('transform', `translate(0, ${height})`).call(d3.axisBottom(x))

      svg.append('g').call(d3.axisLeft(y))

      svg
        .selectAll('.bar')
        .data(this.results)
        .enter()
        .append('rect')
        .attr('class', 'bar')
        .attr('x', (d) => x(d.finder))
        .attr('width', x.bandwidth())
        .attr('y', (d) => y(d.time))
        .attr('height', (d) => height - y(d.time))
        .append('title')
        .text((d) => `${d.finder}: ${d.time.toFixed(2)} ms`)
    }
  },
  watch: {
    results() {
      d3.select('#chart').html('')
      this.createBarChart()
    }
  }
}
</script>

<style>
#app {
  font-family: Arial, sans-serif;
  max-width: 1000px;
  margin: 0 auto;
  padding: 20px;
}

.controls {
  flex-wrap: wrap;
  gap: 10px;
  margin-bottom: 20px;
}

.controlBox {
  display: flex;
  gap: 10px;
}

.control {
  display: flex;
  align-items: center;
}

.table-wrapper {
  display: flex;
  gap: 20px;
}

table {
  border-collapse: collapse;
  margin-top: 20px;
}

td {
  border: 1px solid black;
  width: 30px;
  height: 30px;
  text-align: center;
  cursor: pointer;
}

.obstacle {
  background-color: #ff0000;
}

.path {
  background-color: #8bc34a;
}

.start {
  background-color: #2196f3;
}

.goal {
  background-color: #2196f3;
}
.process {
  background-color: #9c27b0;
}
.layoverPoints {
  background-color: #9c27b0;
}

.dashboard {
  margin-top: 20px;
}

.dashboard table {
  width: 100%;
  border-collapse: collapse;
}

.dashboard th,
.dashboard td {
  border: 1px solid black;
  padding: 8px;
  text-align: left;
}
.grid-container {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
}

.grid-wrapper {
  flex: 1;
}

.bar {
  fill: steelblue;
}

.bar:hover {
  fill: brown;
}
</style>
