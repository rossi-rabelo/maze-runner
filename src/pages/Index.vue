<template>
  <q-page class="flex flex-center">
    <div class="row full-width justify-center text-h6 text-negative">
      <div class="row full-width justify-center text-center q-mb-sm">
        <span v-if="state === 1">
          Marque a posição inicial do agente:
        </span>
        <span v-if="state === 2">
          Marque a posição objetivo do agente:
        </span>
        <div v-if="state === 3">
          <span>Quantos obstáculos deseja inserir no campo?</span>
          <div class="row full-width justify-center">
            <q-input
              v-model.number="obstaclesNumber"
              type="number"
              :disable="this.searchedPath"
              outlined
              dense
              :maxlength="(rows*cols)/2"
              style="max-width: 200px"
            />
          </div>
        </div>
      </div>
      <div class="row full-width justify-center">
        <q-btn :disable="this.findingPath || this.searchedPath" class="q-mx-sm" outline no-caps rounded color="positive" @click="confirmState()" label="Confirmar" />
        <q-btn :disable="this.findingPath" class="q-mx-sm" outline no-caps rounded color="secondary" @click="reset()" label="Resetar" />
        <q-btn :disable="this.findingPath" v-if="state === 3" class="q-mx-sm" outline no-caps rounded color="negative" @click="findPath()" label="Procurar Caminho Euclidiano" />
        <q-btn :disable="this.findingPath" v-if="state === 3" class="q-mx-sm" outline no-caps rounded color="negative" @click="findPathManhattan()" label="Procurar Caminho Manhattan" />
      </div>
    </div>
    <table>
      <tr v-for="(row, indexR) in rows" :key="indexR">
        <td v-for="(col, indexC) in cols" :key="indexC">
          <div
            @click="doStateAction(indexR, indexC)"
            :class="`cel
              ${(agentPosition.row === indexR && agentPosition.col === indexC) ? 'initialCel' : ''}
              ${(goalPosition.row === indexR && goalPosition.col === indexC) ? 'goalCel' : ''}
              ${obstacles.find(element => (element.row === indexR && element.col === indexC)) ? 'obstacleCel' : ''}
              ${resolutionPath.find(element => (element.row === indexR && element.col === indexC)) ? 'resolutionPath' : ''}
              ${resolutionPathManhattan.find(element => (element.row === indexR && element.col === indexC)) ? 'resolutionPathManhattan' : ''}
              ${resolutionPathManhattan.find(element => (element.row === indexR && element.col === indexC))
              && resolutionPath.find(element => (element.row === indexR && element.col === indexC)) ? 'dualPath' : ''}
            `"
          >
          </div>
        </td>
      </tr>
    </table>
  </q-page>
</template>

<script>
/* eslint-disable */
export default {
  name: 'PageIndex',
  data () {
    return {
      rows: 25,
      cols: 25,
      state: 1,
      agentPosition: {
        row: null,
        col: null
      },
      goalPosition: {
        row: null,
        col: null
      },
      currentAgentPosition: {
        row: null,
        col: null
      },
      obstaclesNumber: 0,
      obstacles: [],
      pathFound: false,
      resolutionPath: [],
      resolutionPathManhattan: [],

      invisibleObstaclesManhattan: [],
      invisibleObstaclesEuclidian: [],
      findingPath: false,
      searchedPath: false
    }
  },
  methods: {
    markInitialPosition (indexR, indexC) {
      this.agentPosition.row = indexR
      this.agentPosition.col = indexC
    },
    markGoalPosition (indexR, indexC) {
      this.goalPosition.row = indexR
      this.goalPosition.col = indexC
    },
    doStateAction (row, col) {
      if (this.state === 1) {
        this.markInitialPosition(row, col)
      } else if (this.state === 2) {
        this.markGoalPosition(row, col)
      }
    },
    confirmState () {
      if (this.state === 1) {
        if (this.agentPosition.row !== null && this.agentPosition.col !== null) {
          this.state = 2
        } else {
          this.$q.notify({
            message: 'Marque uma posicial inicial para o agente primeiro!',
            color: 'negative'
          })
        }
      } else if (this.state === 2) {
        if (this.goalPosition.row !== null && this.goalPosition.col !== null) {
          this.state = 3
        } else {
          this.$q.notify({
            message: 'Marque uma posicial objetivo para o agente primeiro!',
            color: 'negative'
          })
        }
      } else if (this.state === 3) {
        this.createObstacles()
      }
    },
    createObstacles () {
      this.$q.loading.show()
      this.obstacles = []
      if (this.obstaclesNumber > (this.rows * this.cols) / 2) {
        this.obstaclesNumber = Math.floor((this.rows * this.cols) / 2)
      }

      let count = 0
      while (count < this.obstaclesNumber) {
        const row = Math.floor(Math.random() * this.rows)
        const col = Math.floor(Math.random() * this.cols)

        if (this.checkPositionAvailability(row, col)) {
          this.obstacles.push({
            row,
            col
          })
          count++
        }
      }
      this.$q.loading.hide()
    },
    checkPositionAvailability (row, col) {
      if (this.agentPosition.row === row && this.agentPosition.col === col) {
        return false
      } else if (this.goalPosition.row === row && this.goalPosition.col === col) {
        return false
      } else if (this.obstacles.find((element) => {
        return element.row === row && element.col === col
      })) {
        return false
      } else {
        return true
      }
    },
    async findPath () {
      this.findingPath = true
      this.searchedPath = true
      this.resolutionPath = []
      this.invisibleObstaclesEuclidian = []
      // Heurística utilizada será a distância euclidiana
      this.currentAgentPosition = JSON.parse(JSON.stringify(this.agentPosition))
      this.resolutionPath.push(this.currentAgentPosition)
      this.pathFound = false

      while (!this.pathFound) {
        console.log('-------------')
        if ((this.currentAgentPosition.row === this.goalPosition.row) && (this.currentAgentPosition.col === this.goalPosition.col)) {
          this.pathFound = true
        } else {
          let allowedMovements = this.validateSteps(this.currentAgentPosition, 1)
    
          let minimumMovement = this.calculateEuclidianDistanceByMovements(this.currentAgentPosition, allowedMovements, this.resolutionPath.length)
    
          if (minimumMovement) {
            console.log('MELHOR MOVIMENTO => ', minimumMovement)
            this.currentAgentPosition = this.updatePosition(this.currentAgentPosition, minimumMovement)
    
            this.resolutionPath.push(this.currentAgentPosition)
            console.log('NOVA POSIÇÃO DO AGENTE => ', this.currentAgentPosition)
          } else {
            this.invisibleObstaclesEuclidian.push(this.currentAgentPosition)
            this.resolutionPath.pop()
            this.currentAgentPosition = this.resolutionPath[this.resolutionPath.length - 1]

            if (!this.resolutionPath.length) {
              this.pathFound = true
              this.$q.notify({
                message: 'Não foi possível encontrar um caminho para o objetivo!!',
                color: 'negative'
              })
            }
          }

           await this.sleep(150)
          // debugger
        }
      }

      this.findingPath = false
    },
    async findPathManhattan () {
      this.findingPath = true
      this.searchedPath = true
      this.resolutionPathManhattan = []
      this.invisibleObstaclesManhattan = []
      // Heurística utilizada será a distância de manhattan
      this.currentAgentPosition = JSON.parse(JSON.stringify(this.agentPosition))
      this.resolutionPathManhattan.push(this.currentAgentPosition)
      this.pathFound = false

      while (!this.pathFound) {
        console.log('-------------')
        if ((this.currentAgentPosition.row === this.goalPosition.row) && (this.currentAgentPosition.col === this.goalPosition.col)) {
          this.pathFound = true
        } else {
          let allowedMovements = this.validateSteps(this.currentAgentPosition, 2)
    
          let minimumMovement = this.calculateManhattanDistanceByMovements(this.currentAgentPosition, allowedMovements, this.resolutionPathManhattan.length)

          if (minimumMovement) {
            console.log('MELHOR MOVIMENTO => ', minimumMovement)
            this.currentAgentPosition = this.updatePosition(this.currentAgentPosition, minimumMovement)
    
            this.resolutionPathManhattan.push(this.currentAgentPosition)
            console.log('NOVA POSIÇÃO DO AGENTE => ', this.currentAgentPosition)
          } else {
            this.invisibleObstaclesManhattan.push(this.currentAgentPosition)
            this.resolutionPathManhattan.pop()
            this.currentAgentPosition = this.resolutionPathManhattan[this.resolutionPathManhattan.length - 1]

            if (!this.resolutionPathManhattan.length) {
              this.pathFound = true
              this.$q.notify({
                message: 'Não foi possível encontrar um caminho para o objetivo!!',
                color: 'negative'
              })
            }
          }

          await this.sleep(150)
          // debugger
        }
      }
      this.findingPath = false
    },
    manhattanDistance (agentPosition, goalPosition) {
      return Math.abs(agentPosition.row - goalPosition.row) + Math.abs(agentPosition.col - goalPosition.col)
    },
    euclidianDistance (agentPosition, goalPosition) {
      return Math.sqrt(Math.pow(Math.abs(agentPosition.row - goalPosition.row), 2) + Math.pow(Math.abs(agentPosition.col - goalPosition.col), 2))
    },
    validateSteps (agentPosition, euristic) {
      let movements = {
        up: false,
        left: false,
        right: false,
        down: false
      }
      if (agentPosition.row > 0 && this.checkMovementAvailability({row: agentPosition.row - 1, col: agentPosition.col}, euristic)) {
        movements.up = true
      }
      if (agentPosition.row < (this.rows - 1) && this.checkMovementAvailability({row: agentPosition.row + 1, col: agentPosition.col}, euristic)) {
        movements.down = true
      }
      if (agentPosition.col > 0 && this.checkMovementAvailability({row: agentPosition.row, col: agentPosition.col - 1}, euristic)) {
        movements.left = true
      }
      if (agentPosition.col < (this.cols - 1) && this.checkMovementAvailability({row: agentPosition.row, col: agentPosition.col + 1}, euristic)) {
        movements.right = true
      }

      return movements
    },
    calculateManhattanDistanceByMovements (agentPosition, allowedMovements, stepsOrigin) {
      const distances = {
        up: null,
        down: null,
        left: null,
        right: null
      }

      if (allowedMovements.up) {
        distances.up = this.manhattanDistance({ row: agentPosition.row - 1, col: agentPosition.col }, this.goalPosition) + stepsOrigin
      }
      if (allowedMovements.left) {
        distances.left = this.manhattanDistance({ row: agentPosition.row, col: agentPosition.col - 1 }, this.goalPosition) + stepsOrigin
      }
      if (allowedMovements.right) {
        distances.right = this.manhattanDistance({ row: agentPosition.row, col: agentPosition.col + 1 }, this.goalPosition) + stepsOrigin
      }
      if (allowedMovements.down) {
        distances.down = this.manhattanDistance({ row: agentPosition.row + 1, col: agentPosition.col }, this.goalPosition) + stepsOrigin
      }

      console.log('DISTANCES => ', distances)

      let arrayDistances = Object.values(distances)
      arrayDistances = arrayDistances.map((element) => {
        if (element === null) {
          return 999999999
        } else {
          return element
        }
      })
      arrayDistances = arrayDistances.sort((a, b) => {
        return a - b
      })
      
      const minimumDistance = arrayDistances[0]
      let minimumMovement = null

      Object.keys(distances).forEach((key) => {
        if (distances[key] === minimumDistance) {
          minimumMovement = key
          return
        }
      })

      return minimumMovement
    },
    calculateEuclidianDistanceByMovements (agentPosition, allowedMovements, stepsOrigin) {
      const distances = {
        up: null,
        down: null,
        left: null,
        right: null
      }

      if (allowedMovements.up) {
        distances.up = this.euclidianDistance({ row: agentPosition.row - 1, col: agentPosition.col }, this.goalPosition) + stepsOrigin
      }
      if (allowedMovements.left) {
        distances.left = this.euclidianDistance({ row: agentPosition.row, col: agentPosition.col - 1 }, this.goalPosition) + stepsOrigin
      }
      if (allowedMovements.right) {
        distances.right = this.euclidianDistance({ row: agentPosition.row, col: agentPosition.col + 1 }, this.goalPosition) + stepsOrigin
      }
      if (allowedMovements.down) {
        distances.down = this.euclidianDistance({ row: agentPosition.row + 1, col: agentPosition.col }, this.goalPosition) + stepsOrigin
      }

      console.log('DISTANCES => ', distances)

      let arrayDistances = Object.values(distances)
      arrayDistances = arrayDistances.map((element) => {
        if (element === null) {
          return 999999999
        } else {
          return element
        }
      })
      arrayDistances = arrayDistances.sort((a, b) => {
        return a - b
      })
      
      const minimumDistance = arrayDistances[0]
      let minimumMovement = null

      Object.keys(distances).forEach((key) => {
        if (distances[key] === minimumDistance) {
          minimumMovement = key
          return
        }
      })

      return minimumMovement
    },
    updatePosition(currentPosition, movement) {
      const movementsActions = {
        up: () => { return {row: currentPosition.row - 1, col: currentPosition.col }},
        down: () => { return {row: currentPosition.row + 1, col: currentPosition.col }},
        right: () => { return {row: currentPosition.row, col: currentPosition.col + 1 }},
        left: () => { return {row: currentPosition.row, col: currentPosition.col - 1 }},
      }

      return movementsActions[movement]()
    },
    checkMovementAvailability (position, euristic) {
      if (euristic === 1) {
        if (this.obstacles.find(element => (element.row === position.row && element.col === position.col))
          || this.resolutionPath.find(element => (element.row === position.row && element.col === position.col))
          || this.invisibleObstaclesEuclidian.find(element => (element.row === position.row && element.col === position.col))
          ) {
          return false
        }
        return true
      } else if (euristic === 2) {
        if (this.obstacles.find(element => (element.row === position.row && element.col === position.col))
          || this.resolutionPathManhattan.find(element => (element.row === position.row && element.col === position.col))
          || this.invisibleObstaclesManhattan.find(element => (element.row === position.row && element.col === position.col))
          ) {
          return false
        }
        return true
      }
    },
    reset () {
      this.state = 1
      this.obstaclesNumber = 0
      this.obstacles = []
      this.searchedPath = false
      this.agentPosition = {
        row: null,
        col: null
      }
      this.goalPosition = {
        row: null,
        col: null
      }
      this.resolutionPath = []
      this.resolutionPathManhattan = []

      this.invisibleObstaclesEuclidian = []
      this.invisibleObstaclesManhattan = []
    },
    sleep(ms) {
      return new Promise(resolve => setTimeout(resolve, ms));
    }
  }
}
</script>

<style lang="scss">
.cel {
  height: 25px;
  width: 25px;
  border: 1px solid black;
  cursor: pointer;
}

.initialCel {
  background-color: green !important;
}

.goalCel {
  background-color: blue !important;
}

.obstacleCel {
  background-color: black;
}

.resolutionPath {
  background-color: red;
}

.resolutionPathManhattan {
  background-color: purple;
}

.dualPath {
  background-color: cyan;
}
</style>
