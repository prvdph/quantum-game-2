<template>
  <div class="game">
    <!-- DRAG AND DROP CELL -->
    <div class="hoverCell"></div>

    <!-- OVERLAY -->
    <app-overlay
      :game-state="overlayGameState"
      :victory-already-shown="victoryAlreadyShown"
      class="overlay"
      @click.native="frameIndex = 0"
    >
      <p class="backButton">GO BACK</p>
      <router-link :to="nextLevelOrOvelay">
        <app-button :overlay="true" :inline="false">NEXT LEVEL</app-button>
      </router-link>
    </app-overlay>

    <!-- GENERAL LAYOUT -->
    <game-layout>
      <!-- HEADER-MIDDLE -->
      <h1 v-if="error" slot="header-middle" class="error">{{ error }}</h1>
      <h1 v-else slot="header-middle" class="title">
        <router-link :to="previousLevel">
          <img src="@/assets/graphics/icons/previousLevel.svg" alt="Previous Level" width="24" />
        </router-link>
        {{ level.id + ' - ' + level.name }}
        <router-link :to="nextLevelOrOvelay">
          <img src="@/assets/graphics/icons/nextLevel.svg" alt="Next Level" width="24" />
        </router-link>
      </h1>

      <!-- MAIN-LEFT -->
      <section slot="main-left">
        <game-toolbox
          :toolbox="level.toolbox"
          @updateCell="updateCell"
          @hover="updateInfoPayload"
        />
        <game-infobox :info-payload="infoPayload" />
      </section>

      <!-- MAIN-MIDDLE -->
      <section slot="main-middle">
        <game-board
          :particles="particles"
          :path-particles="pathParticles"
          :fate="fateCoord"
          :display-fate="displayFate"
          :hints="hints"
          :grid="level.grid"
          :absorptions="filteredAbsorptions"
          :frame-index="frameIndex"
          @updateSimulation="updateSimulation"
          @updateCell="updateCell"
          @play="play"
          @hover="updateInfoPayload"
        />
        <game-controls
          :frame-index="frameIndex"
          :total-frames="simulation.frames.length"
          :display-status="displayStatus"
          @rewind="rewind"
          @step-back="stepBack"
          @play="play"
          @step-forward="stepForward"
          @fast-forward="fastForward"
          @new-fate="computeNewFate"
          @reload="reload"
          @downloadLevel="downloadLevel"
          @loadedLevel="loadLevel($event)"
          @hover="updateInfoPayload"
          @saveLevel="handleSave"
        />
      </section>

      <!-- MAIN-RIGHT -->
      <section slot="main-right">
        <game-goals
          :game-state="level.gameState"
          :percentage="level.gameState.totalAbsorptionPercentage"
        />
        <div class="ket-viewer-game">
          <ket-viewer class="ket" :vector="activeFrame.photons.vector" />
        </div>
      </section>
    </game-layout>
  </div>
</template>

<script lang="ts">
import _ from 'lodash'
import { Vue, Component, Watch } from 'vue-property-decorator'
import { State, Mutation, namespace } from 'vuex-class'
import { IHint, GameStateEnum } from '@/engine/interfaces'
import { IInfoPayload } from '@/mixins/gameInterfaces'
import { Cell, Grid, Level, Particle, Coord } from '@/engine/classes'
import Toolbox from '@/engine/Toolbox'
import MultiverseGraph from '@/engine/MultiverseGraph'
import QuantumFrame from '@/engine/QuantumFrame'
import QuantumSimulation from '@/engine/QuantumSimulation'
import Absorption from '@/engine/Absorption'
import levels from '@/assets/data/levels'
import { KetViewer } from 'bra-ket-vue'
import GameGoals from '@/components/GamePage/GameGoals.vue'
import GameInfobox from '@/components/GamePage/GameInfobox.vue'
import GameToolbox from '@/components/GamePage/GameToolbox.vue'
import GameControls from '@/components/GamePage/GameControls.vue'
import GamePhotons from '@/components/GamePage/GamePhotons.vue'
import GameLayout from '@/components/GamePage/GameLayout.vue'
import GameBoard from '@/components/Board/index.vue'
import GameGraph from '@/components/GamePage/GameGraph.vue'
import AppButton from '@/components/AppButton.vue'
import AppOverlay from '@/components/AppOverlay.vue'
import { getRockTalkIdByLevelId } from '@/components/RockTalkPage/loadRockTalks'

const userModule = namespace('userModule')

@Component({
  components: {
    GameLayout,
    GamePhotons,
    KetViewer,
    GameGoals,
    GameInfobox,
    GameToolbox,
    GameGraph,
    GameControls,
    GameBoard,
    AppButton,
    AppOverlay,
  },
})
export default class Game extends Vue {
  level = Level.createDummy()
  @State('activeCell') activeCell!: Cell // this need to me removed ASASP - the same fate as... fate
  @State('gameState') gameState!: GameStateEnum
  @State('simulationState') simulationState!: boolean
  @userModule.Action('SAVE_LEVEL') actionSaveLevel!: Function
  @userModule.Action('UPDATE_LEVEL') actionUpdateLevel!: Function
  @userModule.Action('GET_LEVEL_DATA') actionGetLevelStoreData!: Function
  @userModule.Action('CLEAR_LEVEL_DATA') actionClearLevelStoreData!: Function
  @userModule.Getter('fetchedLevelBoardState') moduleGetterFetchedBoardState!: string
  @Mutation('SET_CURRENT_LEVEL_ID') mutationSetCurrentLevelID!: (id: string) => void
  @Mutation('SET_GAME_STATE') mutationSetGameState!: (gameState: GameStateEnum) => void
  @Mutation('SET_SIMULATION_STATE') mutationSetSimulationState!: (simulationState: boolean) => void
  @Mutation('SET_HOVERED_CELL') mutationSetHoveredCell!: (cell: Cell) => void
  frameIndex = 0
  simulation: QuantumSimulation = new QuantumSimulation(Grid.emptyGrid())
  multiverseGraph: MultiverseGraph = new MultiverseGraph(this.simulation)
  error = ''
  playInterval = 0
  absorptionThreshold = 0.0001
  infoPayload: IInfoPayload = {
    kind: 'ui',
    particles: [],
    text: 'Hover on a element for more information.',
  }

  fateCoord = Coord.importCoord({ x: -1, y: -1 })
  fateStep = 999
  displayFate = false
  victoryAlreadyShown = false
  showVictory = false
  unsubscribe?: Function

  get displayStatus(): string {
    if (this.simulationState) {
      return 'Quantum simulation (live)' // (each time a random outcome)
    } else if (this.frameIndex > 0) {
      return 'Quantum simulation (step-by-step)'
    } else {
      return 'Classical laser beam' // (still with polarization & interference)
    }
  }

  // LIFECYCLE
  created(): void {
    this.loadLevel()
    this.mutationSetCurrentLevelID(this.levelId)
    window.addEventListener('keyup', this.handleArrowPress)
  }

  beforeDestroy(): void {
    window.removeEventListener('keyup', this.handleArrowPress)
    if (this.unsubscribe) {
      this.unsubscribe()
    }
  }

  /**
   * Parse url to extract level number
   * if missing then fallback to '0' for infinity level / sandbox
   */
  get levelId(): string {
    return this.$route.params.id || '0'
  }

  /**
   * Watch the current route and update level accordingly
   */
  @Watch('levelId')
  changeLevel(): void {
    this.mutationSetCurrentLevelID(this.levelId)
    this.loadLevel()
  }

  /**
   * Depending on whether the level at hand is a custom
   * or a default one, appropriate level loading logic is applied.
   * @returns an answear to whether the level is a custom one
   * @remarks In the future we need a more flexible way
   * to determine this.
   */
  get isCustomLevel(): boolean {
    return this.levelId.length > 4
  }

  /**
   * Level Loading Logic
   * @params none, the function evaluates which level should be loaded
   * @returns nothing, function substitures this.level
   * @remarks the 'saved' variable determines whether the level should
   * be loaded out of app data, or fetched from db
   */
  loadLevel(): void {
    this.victoryAlreadyShown = false
    this.error = ''

    if (this.isCustomLevel) {
      // 1) dispatch action for vuex to get the level, the "shared" parameter
      //    handles which db we target
      this.actionGetLevelStoreData({ id: this.levelId })
        .then(() => {
          // 2) subscribe to mutation triggered asynchronously
          this.unsubscribe = this.$store.subscribe((mutation, rootState) => {
            if (mutation.type === 'userModule/SET_FETCHED_LEVEL') {
              // 3) in case of a successful fetch,
              //    get substitute this.level with store data.
              if (rootState.userModule.fetchedLevel) {
                const fetchedLevelBoardObj = JSON.parse(this.moduleGetterFetchedBoardState)
                this.level = Level.importLevel(fetchedLevelBoardObj)
                this.setFirstToolAsHovered()
                this.updateSimulation()
                this.actionClearLevelStoreData()
              }
            }
          })
        })
        .catch((error: Error) => {
          this.error = error.message
        })
    } else {
      this.level = Level.importLevel(levels[parseInt(this.levelId)])
      this.setFirstToolAsHovered()
      this.updateSimulation()
    }
  }

  // Set hovered cell as first element of toolbox
  setFirstToolAsHovered(): void {
    if (this.level.toolbox.uniqueCellList.length > 0) {
      this.mutationSetHoveredCell(this.level.toolbox.uniqueCellList[0])
    }
  }

  updateInfoPayload(infoPayload: IInfoPayload): void {
    this.infoPayload = infoPayload
  }

  /**
   * Level loading and initialization
   * @returns boolean
   */
  updateSimulation(): void {
    // Compute simulation frames
    this.simulation = new QuantumSimulation(this.level.grid)
    this.simulation.initializeFromLaser()
    this.simulation.computeFrames(40)
    // Set absorption events to compute gameState
    this.level.gameState.absorptions = this.filteredAbsorptions
    // Reset simulation variables
    this.frameIndex = 0
    this.displayFate = false
    this.setEnergizedCells()
    if (this.levelId !== '0' && this.levelId.length < 5) {
      this.mutationSetGameState(this.level.gameState.gameState)
    }
    this.mutationSetSimulationState(false)
    console.debug(this.level.gameState.toString())
  }

  /**
   * Launch overlay if it's the last frame and the player has a game state set
   */
  get overlayGameState(): string {
    if (this.frameIndex === this.fateStep) {
      return this.gameState
    }
    return 'InProgress'
  }

  /**
   * Set the energized cells from the simulation
   */
  setEnergizedCells(): void {
    this.level.grid.resetEnergized()
    const coords = this.filteredAbsorptions.map((absorption) => absorption.coord)
    this.level.grid.setEnergized(coords)
  }

  /**
   * Set the energized cells from the simulation
   */
  setEnergizedCellAtTheEnd(): void {
    this.displayFate = true
    this.level.grid.setEnergized([this.fateCoord])
  }

  /**
   * Output cells linked to detection events
   * @returns Cell and percentage
   */
  get filteredAbsorptions(): Absorption[] {
    // Filter out of grid cells
    return this.simulation.absorptions.filter((absorption: Absorption) => {
      return absorption.cell.coord.x !== -1 && absorption.probability > this.absorptionThreshold
    })
  }

  /**
   * Change active frame with provided Id
   */
  handleChangeActiveFrame(activeId: number): void {
    this.frameIndex = activeId
  }

  /**
   * Process the goals from level with the results of the quantum simulation
   *  @returns goals
   */
  get framePercentage(): number {
    return this.activeFrame.probability * 100
  }

  /**
   * compute paths for quantum laser paths
   * @returns individual paths
   */
  get pathParticles(): Particle[] {
    return _.uniq(this.simulation.allParticles)
  }

  /**
   * Get the current simulation frame
   */
  get activeFrame(): QuantumFrame {
    return this.simulation.frames[this.frameIndex]
  }

  /**
   * Current simulation frame particles
   */
  get particles(): Particle[] {
    return this.activeFrame.particles
  }

  /**
   * Compute another fate for the simulation
   */
  computeNewFate(): void {
    const fate = this.simulation.sampleRandomRealization()
    this.displayFate = false
    this.fateCoord = fate.coord
    this.fateStep = fate.step
  }

  /**
   * Show previous frame and check it exists
   *  @returns frameIndex
   */
  rewind(): void {
    this.frameIndex = 0
  }

  /**
   * Show previous frame and check it exists
   *  @returns frameIndex
   */
  stepBack(): number {
    const newframeIndex = this.frameIndex - 1
    if (newframeIndex < 0) {
      console.error("Can't access frames before simulation...")
      return 0
    }
    this.frameIndex = newframeIndex
    return this.frameIndex
  }

  /**
   *  Reset frameIndex, flip it up for every frame,
   *  then clear the interval
   * @returns void
   */
  play(): void {
    if (this.simulationState) {
      clearInterval(this.playInterval)
      this.mutationSetSimulationState(false)
      return
    }
    this.level.grid.resetEnergized()
    this.computeNewFate()
    this.frameIndex = 0
    this.playInterval = setInterval(() => {
      if (this.frameIndex < this.fateStep) {
        this.frameIndex += 1
      } else {
        this.mutationSetSimulationState(false)
        this.setEnergizedCellAtTheEnd()
        clearInterval(this.playInterval)
        if (!this.victoryAlreadyShown && this.gameState === GameStateEnum.Victory) {
          setTimeout(() => {
            // dirty hack
            this.victoryAlreadyShown = true
          }, 200)
        } else {
          setTimeout(() => {
            this.updateSimulation()
          }, 1000)
        }
      }
    }, 200)
    this.mutationSetSimulationState(true)
  }

  /**
   * Show next frame and check it exists
   *  @returns frameIndex
   */
  stepForward(): number {
    this.level.grid.resetEnergized()
    const newframeIndex = this.frameIndex + 1
    if (newframeIndex > this.simulation.frames.length - 1) {
      console.error("Can't access frames that are not computed yet...")
      return this.simulation.frames.length - 1
    }
    this.frameIndex = newframeIndex
    return this.frameIndex
  }

  /**
   * Reload the current page
   */
  fastForward(): void {
    this.frameIndex = this.simulation.frames.length - 1
  }

  /**
   * Reload the current page
   */
  reload(): void {
    window.location.reload(false)
  }

  /**
   * Export the level in JSON format and uploads it
   * @returns level in JSON format
   */
  downloadLevel(): void {
    const json = JSON.stringify(this.level.exportLevelForDownload(), null, 2)
    const blob = new Blob([json], { type: 'octet/stream' })
    const link = document.createElement('a')
    link.href = window.URL.createObjectURL(blob)
    link.download = 'level.json'
    link.click()
  }

  /**
   * Left and right keys to see frames
   * TODO: Implement dev mode
   */
  handleArrowPress(e: { keyCode: number }): void {
    // console.log(e.keyCode);
    switch (e.keyCode) {
      // case 81:
      //   this.level.grid.moveAll(180);
      //   break;
      // case 68:
      //   this.level.grid.moveAll(0);
      //   break;
      // case 90:
      //   this.level.grid.moveAll(90);
      //   break;
      // case 83:
      //   this.level.grid.moveAll(270);
      //   break;
      // case 65:
      //   this.level.grid.rotateAll();
      //   break;
      // case 69:
      //   this.level.grid.reflectAll();
      //   break;
      case 32:
        this.play()
        break
      case 37:
        this.stepBack()
        break
      case 39:
        this.stepForward()
        break
      default:
        break
    }
  }

  removeFromCurrentTools(cell: Cell): void {
    this.level.toolbox.removeTool(cell)
  }

  addToCurrentTools(cell: Cell): void {
    this.level.toolbox.addTool(cell, this.activeCell)
  }

  setCurrentTools(cells: Cell[]): void {
    this.level.toolbox = new Toolbox(cells)
  }

  resetCurrentTools(): void {
    this.level.toolbox.reset()
  }

  /**
   * the main level grid state updating method
   * checks the updated cell details, compares them
   * with the active cell and proceeds accordingly
   * @param cell
   * @returns void
   */
  updateCell(cell: Cell): void {
    const sourceCell = this.activeCell
    const targetCell = cell
    if (
      // handle moving from toolbox to grid
      this.activeCell.isFromToolbox &&
      cell.isFromGrid &&
      cell.isVoid
    ) {
      this.removeFromCurrentTools(this.activeCell)
    } else if (
      // handle moving from grid to toolbox
      this.activeCell.isFromGrid &&
      cell.isFromToolbox &&
      !this.activeCell.isVoid &&
      !cell.isVoid
    ) {
      this.addToCurrentTools(cell)
      this.level.grid.set(this.activeCell.reset())
    }
    // FIXME: unify moving logic
    this.level.grid.move(sourceCell, targetCell)
    // const mutatedCells = this.level.grid.move(sourceCell, targetCell)
    // mutatedCells.forEach((mutatedCell: Cell) => {
    //   this.level.grid.set(mutatedCell)
    // })
    this.saveLevelToStore()
    this.updateSimulation()
  }

  /**
   * Utilized by updateCell to keep level in localStorage
   */
  saveLevelToStore(): void {
    const currentStateJSONString = JSON.stringify(this.level.exportLevel())
    localStorage.setItem(this.currentLevelName, currentStateJSONString)
  }

  /**
   * Actual 'save' button handling
   */
  handleSave(): void {
    const currentStateJSONString = JSON.stringify(this.level.exportLevel())
    const newState = { ...this.$store.state, boardState: currentStateJSONString }
    if (!this.isCustomLevel) {
      this.actionSaveLevel(newState)
    } else {
      this.actionUpdateLevel(newState)
    }
  }

  clearLS(): void {
    localStorage.removeItem(this.currentLevelName)
  }

  // GETTERS
  get currentLevelName(): string {
    return `level${this.levelId}`
  }

  /**
   * The next/previous level url getters
   * They get blocked in case it is a custom level
   * with its hashed ID.
   */
  get previousLevel(): string {
    if (this.isCustomLevel) {
      return `/level/${this.levelId}`
    }
    return `/level/${parseInt(this.levelId) - 1}`
  }

  get nextLevel(): string {
    if (this.isCustomLevel) {
      return `/level/${this.levelId}`
    }
    return `/level/${parseInt(this.levelId) + 1}`
  }

  /**
   * Should an overlay be shown when progressing
   * to the next level?
   * @returns an router link :to attribute string
   */
  get nextLevelOrOvelay(): string {
    const possibleOverlay = getRockTalkIdByLevelId(parseInt(this.levelId))
    return possibleOverlay ? `/rocks/${possibleOverlay}` : this.nextLevel
  }

  get hints(): IHint[] {
    return this.level.hints.map((hint) => hint.exportHint())
  }
}
</script>

<style lang="scss" scoped>
// Drag & drop
.hoverCell {
  pointer-events: none;
  position: absolute;
  z-index: 10;
}
// Overlay
.overlay {
  .backButton {
    font-size: 0.8rem;
    opacity: 0.8;
    cursor: pointer;
  }
}

// Title
h1.title {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  font-size: 1.5rem;
  margin-bottom: 30;
  margin-top: 0;
  text-transform: uppercase;
  @media screen and (max-width: 1000px) {
    a img {
      width: 6vw !important;
    }
  }
}

.game {
  width: 100%;
  min-height: 100vh;
  &.goals {
    height: 600px;
  }
}
.ket-viewer-game {
  margin-top: 10px;
  padding: 10px;
  background-color: rgba(0, 0, 0, 0.1);
}
</style>
