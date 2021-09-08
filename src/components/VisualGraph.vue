<template>
  <div>
    <img class="logo" src="../assets/logo.jpg" alt="Logo">
    <h1>מחשבון המסלול הקצר ביותר</h1>
    <h3>ע״פ האלגוריתם של דייקסטרה</h3>
    <div class="canvas-wrapper">
      <canvas @mousedown="paint" @touchstart="paint" ref="canv"></canvas>
    </div>
    <div class="table-wrapper">
      <table v-if="renderList">
        <tr v-for="(dot, dotIndex) in dots" v-bind:key="dotIndex">
          <td><span class="dot-name">{{ dot.name }}</span></td>
          <td>
            <button type="button" @click="moveDot(dotIndex)" :class="{'active':dotIndex === dotIndexToMove}">הזזה
            </button>
          </td>
          <td>
            <button type="button" @click="deleteDot(dotIndex)">מחיקה</button>
          </td>
          <td v-for="(distance, distIndex) in dotDistances(dotIndex)" v-bind:key="distIndex" class="dist-col">
            <label>{{ names[distIndex] }}</label>&nbsp;
            <input :disabled="dotIndex === distIndex" oninput="validity.valid||(value='');" type="number" min="0" v-model.number="distances[dotIndex][distIndex]"
                   @change="connectDots(dotIndex, distIndex)">
          </td>
        </tr>
      </table>
    </div>
    <div class="src-dist-inputs">
      <div class="form-group">
        <label for="src">מקור</label>
        <select v-if="renderList" name="src" id="src" v-model="srcIndex">
          <option v-for="(dot, dotIndex) in dots" :value="dotIndex" v-bind:key="dotIndex">{{ dot.name }}</option>
        </select>
      </div>
      <div class="form-group">
        <label for="dist">יעד</label>
        <select v-if="renderList" name="dist" id="dist" v-model="distIndex">
          <option v-for="(dot, dotIndex) in dots" :value="dotIndex" v-bind:key="dotIndex">{{ dot.name }}</option>
        </select>
      </div>
    </div>
    <div class="action-buttons">
      <button class="calc-btn" type="button" @click="calculate">חשב</button>
      <button class="reset-btn" type="button" @click="reset">איפוס</button>
    </div>
    <!--<button type="button" @click="redraw">צייר מחדש</button>-->
    <div class="results-path" v-if="resultPath.length">
      <div>
          <label>המסלול הקצר ביותר הוא:</label>&nbsp;
          <span>{{ resultPath.join(' => ') }}</span>
      </div>
    </div>
    <div class="results" v-if="result">
        <label>ואורכו הוא:</label>&nbsp;
        <span>{{result}}</span>
    </div>
  </div>
</template>

<script>
let NO_PARENT = -1;

export default {
  name: "VisualGraph",
  data() {
    return {
      canvas: null,
      ctx: null,
      dots: [],
      distances: {},
      names: ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z'],
      dotIndexToMove: null,
      nextDotIndex: 0,
      result: '',
      resultPath: [],
      lastPaintTime: 0,
      renderList: true,
      srcIndex: 0,
      distIndex: 0
    }
  },
  computed: {
    isTouchEnabled() {
      return ('ontouchstart' in window) ||
          (navigator.maxTouchPoints > 0) ||
          (navigator.msMaxTouchPoints > 0);
    }
  },
  methods: {
    dotDistances(dotIndex) {
      if (this.distances[dotIndex]) {
        if (this.distances[dotIndex].length < this.dots.length) {
          for (let i = 0; i < this.dots.length - this.distances[dotIndex].length; i++) {
            this.distances[dotIndex].push(0)
          }
        }
      } else {
        this.distances[dotIndex] = []
        for (let i = 0; i < this.dots.length; i++) {
          this.distances[dotIndex].push(0)
        }
      }
      return this.distances[dotIndex]
    },
    paint(event) {
      let e = event;

      const now = new Date().getTime();

      //prevent double dot creation on touch & click supported devices
      if (this.lastPaintTime !== 0 && now - this.lastPaintTime < 250) {
        return false;
      }

      this.lastPaintTime = now;

      if (event instanceof TouchEvent && event.touches[0]) {
        e = event.touches[0]
      }

      const rect = this.canvas.getBoundingClientRect();
      if (this.dotIndexToMove !== null) {
        this.dots[this.dotIndexToMove].x = e.clientX - rect.left
        this.dots[this.dotIndexToMove].y = e.clientY - rect.top
        this.dotIndexToMove = null
        this.redraw()
      } else {
        const dotName = prompt("Type Name", this.names[this.nextDotIndex])
        if (dotName) {
          this.names[this.nextDotIndex] = dotName.toUpperCase()
          localStorage.setItem('names', JSON.stringify(this.names))
        }
        this.paintDot(e.clientX - rect.left, e.clientY - rect.top)
      }
    },
    paintDot(x, y, label, pushToArray = true) {
      this.ctx.fillStyle = "#000000";
      this.ctx.beginPath();
      this.ctx.arc(x, y, 5, 0, 2 * Math.PI);
      this.ctx.fill();
      this.ctx.font = "15px Arial";
      if (label) {
        this.ctx.fillText(label, x - 5, y - 10);
      } else {
        this.ctx.fillText(this.names[this.nextDotIndex], x - 5, y - 10);
      }

      this.ctx.textAlign = "center";

      if (pushToArray) {
        this.dots.push({x, y, name: this.names[this.nextDotIndex]})
        this.nextDotIndex++
      }

      localStorage.setItem('dots', JSON.stringify(this.dots))
    },
    paintLine(fromDot, toDot, label) {
      this.ctx.beginPath()
      this.ctx.moveTo(fromDot.x, fromDot.y)
      this.ctx.lineTo(toDot.x, toDot.y)
      this.ctx.stroke()

      const midX = fromDot.x + (toDot.x - fromDot.x) * 0.50
      const midY = fromDot.y + (toDot.y - fromDot.y) * 0.50

      this.ctx.font = "15px Arial";
      this.ctx.fillText(label, midX - 10, midY - 10)

      localStorage.setItem('distances', JSON.stringify(this.distances))
    },
    connectDots(fromDotIndex, toDotIndex) {
      const distanceWeight = this.distances[fromDotIndex][toDotIndex]
      this.distances[toDotIndex][fromDotIndex] = distanceWeight
      if (this.dots[fromDotIndex] && this.dots[toDotIndex] && fromDotIndex !== toDotIndex) {
        this.redraw()
        this.forceRerenderList()
      }
    },
    redraw() {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height)
      this.ctx.fillStyle = "rgb(215,215,215)"
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);

      if (Array.isArray(this.dots)) {
        this.dots.forEach((dot) => {
          this.paintDot(dot.x, dot.y, dot.name, false)
        })

        this.dots.forEach((dot, dotIndex) => {
          if (this.distances[dotIndex]) {
            this.distances[dotIndex].forEach((dist, distIndex) => {
              if (dist) {
                this.paintLine(this.dots[dotIndex], this.dots[distIndex], dist)
              }
            })
          }
        })
      }
    },
    moveDot(dotIndex) {
      this.dotIndexToMove = dotIndex
    },
    deleteDot(index) {
      this.dots.forEach((dot, dotIndex) => {
        if (this.distances[dotIndex]) {
          this.distances[dotIndex].splice(index, 1)
        }
      })
      if (this.distances[index]) {
        this.$delete(this.distances, index)
      }
      this.dots.splice(index, 1)

      this.redraw()
    },
    calculate() {
      const graph = []
      for (let dotIndex in this.distances) {
        graph.push(this.distances[dotIndex])
      }
      this.dijkstra(graph, this.srcIndex)
    },
    reset() {
      this.nextDotIndex = 0
      this.dots = []
      this.distances = []
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height)
      localStorage.removeItem('dots')
      localStorage.removeItem('distances')
    },
    loadFromLocalStorage() {
      const dotsJson = localStorage.getItem('dots')
      if (dotsJson) {
        this.dots = JSON.parse(dotsJson)
      }

      const distancesJson = localStorage.getItem('distances')
      if (distancesJson) {
        this.distances = JSON.parse(distancesJson)
      }

      const namesJson = localStorage.getItem('names')
      if (namesJson) {
        this.names = JSON.parse(namesJson)
      }

      this.nextDotIndex = this.dots.length

      this.redraw()
    },
    forceRerenderList() {
      // Remove my-component from the DOM
      this.renderList = false;

      // If you like promises better you can
      // also use nextTick this way
      this.$nextTick().then(() => {
        // Add the component back in
        this.renderList = true;
      });
    },
    dijkstra(adjacencyMatrix, startVertex) {
      let nVertices = adjacencyMatrix[0].length;

      // shortestDistances[i] will hold the
      // shortest distance from src to i
      let shortestDistances = new Array(nVertices);

      // added[i] will true if vertex i is
      // included / in shortest path tree
      // or shortest distance from src to
      // i is finalized
      let added = new Array(nVertices);

      // Initialize all distances as
      // INFINITE and added[] as false
      for (let vertexIndex = 0; vertexIndex < nVertices;
           vertexIndex++) {
        shortestDistances[vertexIndex] = Number.MAX_VALUE;
        added[vertexIndex] = false;
      }

      // Distance of source vertex from
      // itself is always 0
      shortestDistances[startVertex] = 0;

      // Parent array to store shortest
      // path tree
      let parents = new Array(nVertices);

      // The starting vertex does not
      // have a parent
      parents[startVertex] = NO_PARENT;

      // Find shortest path for all
      // vertices
      for (let i = 1; i < nVertices; i++) {

        // Pick the minimum distance vertex
        // from the set of vertices not yet
        // processed. nearestVertex is
        // always equal to startNode in
        // first iteration.
        let nearestVertex = -1;
        let shortestDistance = Number.MAX_VALUE;
        for (let vertexIndex = 0;
             vertexIndex < nVertices;
             vertexIndex++) {
          if (!added[vertexIndex] &&
              shortestDistances[vertexIndex] <
              shortestDistance) {
            nearestVertex = vertexIndex;
            shortestDistance = shortestDistances[vertexIndex];
          }
        }

        // Mark the picked vertex as
        // processed
        added[nearestVertex] = true;

        // Update dist value of the
        // adjacent vertices of the
        // picked vertex.
        for (let vertexIndex = 0;
             vertexIndex < nVertices;
             vertexIndex++) {
          let edgeDistance = adjacencyMatrix[nearestVertex][vertexIndex];

          if (edgeDistance > 0
              && ((shortestDistance + edgeDistance) <
                  shortestDistances[vertexIndex])) {
            parents[vertexIndex] = nearestVertex;
            shortestDistances[vertexIndex] = shortestDistance +
                edgeDistance;
          }
        }
      }

      this.printSolution(startVertex, shortestDistances, parents);
    },
    printSolution(startVertex, distances, parents) {
      this.resultPath = [];
      this.result = `${distances[this.distIndex]}`;
      //this.result += ("<br>" + this.names[this.distIndex] + "        ");
      //this.result += (distances[this.distIndex] + "      ");
      this.printPath(this.distIndex, parents)
    },
    printPath(currentVertex, parents) {
      // Base case : Source node has
      // been processed
      if (currentVertex === NO_PARENT) {
        return;
      }

      this.printPath(parents[currentVertex], parents)
      this.resultPath.push(this.names[currentVertex])
    }
  },
  mounted() {
    this.canvas = this.$refs.canv
    this.ctx = this.canvas.getContext('2d')
    if (window.innerWidth < 767) {
      this.canvas.width = window.innerWidth
      this.canvas.height = window.innerWidth
    } else {
      this.canvas.width = 767
      this.canvas.height = 431
    }

    this.loadFromLocalStorage()
  }
}
</script>

<style scoped>
body, h1, h2, h3, h4, h5, h6, table, input, label, button,p {
  font-family: Arial, sans-serif;
}

h1 {
    margin-bottom: 5px;
}

h3 {
    margin-top: 5px;
}

.canvas-wrapper {
  text-align: center;
}

canvas {
  width: 767px;
  max-width: 100%;
  border: solid 1px #000;
}

input[type="number"] {
  width: 40px;
  padding: 0;
}

button.active {
  background-color: #f00;
}

.table-wrapper {
  max-width: 100%;
  overflow-x: auto;
}

.table-wrapper table {
  margin-left: auto;
  margin-right: auto;
}

table td {
  white-space: nowrap;
  text-align: left;
}

td.dist-col {
  border: solid 1px;
}

td.dist-col input {
  border: none;
}

td.dist-col input[disabled] {
  opacity: .4;
  cursor: not-allowed;
}

.dot-name {
  font-size: 20px;
  font-weight: bold;
}

.calc-btn {
  margin-left: 10px;
}

h1, h3 {
  text-align: center;
}

.src-dist-inputs {
  margin-top: 10px;
}

.src-dist-inputs label {
  margin-left: 10px;
}

.form-group {
  margin-bottom: 10px;
}

.results, .results-path {
  margin-top: 10px;
}

.action-buttons, .src-dist-inputs, .results, .results-path {
  direction: rtl;
  text-align: center;
}

.logo {
    margin-left: auto;
    margin-right: auto;
    display: block;
    width: 200px;
    margin-bottom: 10px;
}

.results label, .results-path label {
    font-size: 18px;
    font-weight: bold;
}

</style>
