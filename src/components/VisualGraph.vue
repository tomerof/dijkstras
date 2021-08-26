<template>
  <div>
    <canvas @mousedown="paint" ref="canv"></canvas>
    <table>
      <tr v-for="(dot, dotIndex) in dots" v-bind:key="dotIndex">
        <td>{{ dot.name }}</td>
        <td>
          <button type="button" @click="moveDot(dotIndex)" :class="{'active':dotIndex === dotIndexToMove}">Move</button>
        </td>
        <td>
          <button type="button" @click="deleteDot(dotIndex)">Delete</button>
        </td>
        <td v-for="(distance, distIndex) in dotDistances(dotIndex)" v-bind:key="distIndex">
          <label>{{ names[distIndex] }}</label>
          <input type="tel" v-model="distances[dotIndex][distIndex]" @change="connectDots(dotIndex, distIndex)">
        </td>
      </tr>
    </table>
    <button type="button" @click="calculate">חשב</button>
    <button type="button" @click="reset">איפוס</button>
    <!--<button type="button" @click="redraw">צייר מחדש</button>-->
    <div class="results" v-html="result"></div>
  </div>
</template>

<script>
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
      result: ''
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
      if (this.dotIndexToMove !== null) {
        this.dots[this.dotIndexToMove].x = event.clientX
        this.dots[this.dotIndexToMove].y = event.clientY
        this.dotIndexToMove = null
        this.redraw()
      } else {
        const rect = this.canvas.getBoundingClientRect();
        this.paintDot(event.clientX - rect.left, event.clientY - rect.top)
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
      if (this.dots[fromDotIndex] && this.dots[toDotIndex] && fromDotIndex !== toDotIndex) {
        if (distanceWeight > 0) {
          this.paintLine(this.dots[fromDotIndex], this.dots[toDotIndex], distanceWeight)
        }
      }
    },
    redraw() {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height)

      if (Array.isArray(this.dots)) {
        this.dots.forEach((dot) => {
          this.paintDot(dot.x, dot.y, dot.name, false)
        })

        this.dots.forEach((dot, dotIndex) => {
          if (this.distances[dotIndex]) {
            this.distances[dotIndex].forEach((dist, distIndex) => {
              this.connectDots(dotIndex, distIndex)
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
    // A utility function to find the
    // vertex with minimum distance
    // value, from the set of vertices
    // not yet included in shortest
    // path tree
    minDistance(dist, sptSet) {

      // Initialize min value
      let min = Number.MAX_VALUE
      let min_index = -1

      for (let v = 0; v < this.dots.length; v++) {
        if (sptSet[v] === false && dist[v] <= min) {
          min = dist[v];
          min_index = v;
        }
      }
      return min_index;
    },

    // A utility function to print
    // the constructed distance array
    printSolution(dist) {

      this.result = "Vertex \t\t Distance from Source<br>";
      for (let i = 0; i < this.dots.length; i++) {
        this.result += i + " \t\t " + dist[i] + "<br>"
      }
    },

    // Function that implements Dijkstra's
    // single source shortest path algorithm
    // for a graph represented using adjacency
    // matrix representation
    dijkstra(graph, src) {
      const V = this.dots.length

      let dist = new Array(V)
      let sptSet = new Array(V)

      // Initialize all distances as
      // INFINITE and stpSet[] as false
      for (let i = 0; i < V; i++) {
        dist[i] = Number.MAX_VALUE
        sptSet[i] = false
      }

      // Distance of source vertex
      // from itself is always 0
      dist[src] = 0

      // Find shortest path for all vertices
      for (let count = 0; count < V - 1; count++) {

        // Pick the minimum distance vertex
        // from the set of vertices not yet
        // processed. u is always equal to
        // src in first iteration.
        let u = this.minDistance(dist, sptSet)

        // Mark the picked vertex as processed
        sptSet[u] = true

        // Update dist value of the adjacent
        // vertices of the picked vertex.
        for (let v = 0; v < V; v++) {

          // Update dist[v] only if is not in
          // sptSet, there is an edge from u
          // to v, and total weight of path
          // from src to v through u is smaller
          // than current value of dist[v]
          if (!sptSet[v] && graph[u][v] !== 0 &&
              dist[u] !== Number.MAX_VALUE &&
              dist[u] + graph[u][v] < dist[v]) {
            dist[v] = dist[u] + graph[u][v]
          }
        }
      }

      // Print the constructed distance array
      this.printSolution(dist)
    },
    calculate(){
      const graph = []
      for(let dotIndex in this.distances){
        graph.push(this.distances[dotIndex])
      }
      this.dijkstra(graph, 3)
    },
    reset(){
      this.nextDotIndex = 0
      this.dots = []
      this.distances = []
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height)
      localStorage.removeItem('dots')
      localStorage.removeItem('distances')
    },
    loadFromLocalStorage(){
      const dotsJson = localStorage.getItem('dots')
      if(dotsJson){
        this.dots = JSON.parse(dotsJson)
      }
      const distancesJson = localStorage.getItem('distances')
      if(distancesJson){
        this.distances = JSON.parse(distancesJson)
      }

      this.nextDotIndex = this.dots.length

      this.redraw()
    }
  },
  mounted() {
    this.canvas = this.$refs.canv
    this.ctx = this.canvas.getContext('2d')
    this.canvas.height = 500
    this.canvas.width = 500

    this.loadFromLocalStorage()
  }
}
</script>

<style scoped>
canvas {
  width: 500px;
  max-width: 100%;
  border: solid 1px #000;
}

input[type="tel"] {
  width: 40px;
  padding: 0;
}

button.active {
  background-color: #f00;
}
</style>
