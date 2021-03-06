<template>
  <svg :width="width + 2 * margin" :height="height + 2 * margin">
    <g class="photon" :style="computeStyle">
      <g>
        <radialGradient id="SVGID_1_" cx="32" cy="32" r="32" gradientUnits="userSpaceOnUse">
          <stop offset="0" style="stop-color:#5C00D3" />
          <stop offset="1" style="stop-color:#5C00D3;stop-opacity:0" />
        </radialGradient>
        <circle class="st0" cx="32" cy="32" r="32" />
      </g>
      <g v-if="displayGaussian" class="gaussian">
        <path class="gaussian" :d="computeGaussianPath.pathUp" />
        <path class="gaussian" :d="computeGaussianPath.pathDown" />
      </g>

      <g v-if="displayElectric" class="electric">
        <path
          v-for="(z, index) in zs"
          :key="`electricPath-${index}`"
          :d="computeElectricPath(z)"
          :stroke-width="eScale(gaussianComplex(bre, bim, z, sigma))"
          :stroke="eColor(gaussianComplex(bre, bim, z, sigma))"
        />
      </g>

      <g v-if="displayMagnetic" class="magnetic">
        <path
          v-for="(z, index) in zs"
          :key="`magneticPath-${index}`"
          :d="computeMagneticPath(z)"
          :stroke-width="mScale(gaussianComplex(are, aim, z, sigma))"
          :stroke="mColor(gaussianComplex(bre, bim, z, sigma))"
        />
      </g>
    </g>
  </svg>
</template>

<script lang="ts">
import { Component, Vue, Prop } from 'vue-property-decorator'
import { select } from 'd3-selection'
import { range } from 'd3-array'
import { scaleLinear, scaleSequential } from 'd3-scale'
import { interpolateViridis, interpolateInferno } from 'd3-scale-chromatic'
import Particle from '@/engine/Particle'

const d3 = {
  select,
  range,
  scaleLinear,
  scaleSequential,
  interpolateInferno,
  interpolateViridis,
}

@Component
export default class AppPhoton extends Vue {
  @Prop() readonly particle!: Particle
  @Prop({ default: 0 }) readonly animate!: number
  @Prop({ default: 64 }) readonly width!: number
  @Prop({ default: 64 }) readonly height!: number
  @Prop({ default: 20 }) readonly margin!: number
  @Prop({ default: 20 }) readonly k!: number
  @Prop({ default: 0.3 }) readonly sigma!: number
  @Prop({ default: 0.001 }) readonly range!: number
  @Prop({ default: false }) readonly displayMagnetic!: boolean
  @Prop({ default: true }) readonly displayElectric!: boolean
  @Prop({ default: true }) readonly displayGaussian!: boolean
  @Prop({ default: true }) readonly displayOpacity!: boolean
  @Prop({ default: true }) readonly displayDirection!: boolean
  @Prop({ default: true }) readonly normalize!: boolean
  @Prop({ default: 1.3 }) readonly squeezeFactor!: number

  /**
   * Getters from particle
   */
  get step(): number {
    return 1 / this.width
  }

  get normalization(): number {
    if (this.normalize) {
      const { are, aim, bre, bim } = this.particle
      return Math.sqrt(are ** 2 + aim ** 2 + bre ** 2 + bim ** 2)
    }
    return 1
  }

  get intensity(): number {
    return this.particle.probability
  }

  get direction(): number {
    return this.particle.direction
  }

  get are(): number {
    return this.particle.are / this.normalization
  }

  get aim(): number {
    return this.particle.aim / this.normalization
  }

  get bre(): number {
    return this.particle.bre / this.normalization
  }

  get bim(): number {
    return this.particle.bim / this.normalization
  }

  get opacity(): number {
    const scalingPow = 0.5
    return this.particle.probability ** scalingPow
  }

  get computeStyle(): {} {
    return {
      opacity: `${this.displayOpacity ? this.opacity : 1}`,
      'transform-origin': `${this.width / 2}px ${this.height / 2}px`,
      transform: `rotate(${this.displayDirection ? this.particle.direction : 0}deg)`,
    }
  }

  /**
   * Compute electric path from points
   */
  computeElectricPath(z: number): string {
    let path = ''
    const ox = this.xScale(z - this.step)
    const oy = this.yScale(this.gaussianComplex(this.are, this.aim, z - this.step, this.sigma))
    const tx = this.xScale(z)
    const ty = this.yScale(this.gaussianComplex(this.are, this.aim, z, this.sigma))
    path += `M ${ox} ${oy} `
    path += `L ${tx} ${ty} `
    return path
  }

  /**
   * Compute electric path from points
   */
  computeMagneticPath(z: number): string {
    let path = ''
    const ox = this.xScale(z - this.step)
    const oy = this.yScale(this.gaussianComplex(this.bre, this.bim, z - this.step, this.sigma))
    const tx = this.xScale(z)
    const ty = this.yScale(this.gaussianComplex(this.bre, this.bim, z, this.sigma))
    path += `M ${ox} ${oy} `
    path += `L ${tx} ${ty}`
    return path
  }

  /**
   * Compute electric path from points
   * @return bots svg paths as strings
   */
  get computeGaussianPath(): { pathUp: string; pathDown: string } {
    let pathUp = ''
    let pathDown = ''
    this.zs.forEach((z, index) => {
      const x = this.xScale(z)
      const y = this.yScale(this.gaussian(z))
      if (index === 0) {
        pathUp += `M ${x} ${y} `
        pathDown += `M ${x} ${this.availableHeight - y} `
      } else {
        pathUp += `L ${x} ${y} `
        pathDown += `L ${x} ${this.availableHeight - y} `
      }
    })
    return { pathUp, pathDown }
  }

  get availableHeight(): number {
    return this.height - this.margin
  }

  /**
   * Get horizontal scaling
   */
  /* eslint-disable-next-line */
  get xScale() {
    return d3
      .scaleLinear()
      .domain([-1, 1])
      .range([this.margin, this.width - this.margin])
  }

  /**
   * Get vertical scaling
   * FIXME: squeezeFactor
   */
  /* eslint-disable-next-line */
  get yScale() {
    return d3
      .scaleLinear()
      .domain([-this.squeezeFactor, this.squeezeFactor])
      .range([0, this.availableHeight])
  }

  /**
   * Get magnetic scaling
   */
  /* eslint-disable-next-line */
  get mScale() {
    return d3
      .scaleLinear()
      .domain([-1, 1])
      .range([0.5, 6])
  }

  /**
   * Get electric scaling
   */
  /* eslint-disable-next-line */
  get eScale() {
    return d3
      .scaleLinear()
      .domain([-1, 1])
      .range([1, 4])
  }

  /**
   * Get SVG coordinates of particle animation
   * @returns string
   */
  get toCoord(): string {
    const tileSize = 64
    const x = this.particle.relativeTarget.x * tileSize
    const y = this.particle.relativeTarget.y * tileSize
    return `${x} ${y}`
  }

  /**
   * Numbers of points to render, should scale with the width
   * @returns a range of steps
   */
  get zs(): number[] {
    return d3.range(-1, 1, this.step)
  }

  /**
   * Magnetic field color scheme.
   */
  mColor = d3.scaleSequential(d3.interpolateViridis).domain([-1, 1])

  /**
   * Electric field color scheme.
   * See also: https://github.com/d3/d3-scale-chromatic/
   *
   */
  eColor = d3
    .scaleLinear<string>()
    .domain([-1, 0, 1])
    .range(['#5c00d3', '#ff0055', '#ffde3e']) // PURPLE RED YELLOW

  /**
   * Compute graph properties from complex values
   */
  computeComplex(re: number, im: number, z: number, k = 20): number {
    return re * Math.cos(k * z) + im * Math.sin(k * z)
  }

  /**
   * Gaussian scaling
   */
  gaussian(z: number, sigma = 0.3): number {
    return Math.exp((-z * z) / (2 * sigma * sigma))
  }

  /**
   * Gaussian scaling of the graph
   */
  gaussianComplex(re: number, im: number, z: number, sigma = 0.3): number {
    return this.computeComplex(re, im, z) * Math.exp((-z * z) / (2 * sigma * sigma))
  }
}
</script>

<style lang="scss" scoped>
.photon {
  padding: 0px;
  position: relative;
  .electric {
    fill: transparent;
  }
  .magnetic {
    fill: transparent;
  }
  .gaussian {
    stroke-width: 3px;
    fill: rgba(92, 0, 211, 1);
    stroke: rgba(92, 0, 211, 0.5);
    stroke-linecap: round;
    stroke-linejoin: round;
  }
  .st0 {
    fill: url(#SVGID_1_);
  }
}
</style>
