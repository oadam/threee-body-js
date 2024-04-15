<script setup lang="ts">
import { Ref, ref, computed, onMounted, onUnmounted } from 'vue'

const SPEED_PER_DRAGGED_PX = 1 // pixel per second per pixel
const DT = 0.01
const KG_PER_PIXEL_CUBE = 1 / 100
const G = 50000;

interface Pos {
  x: number
  y: number
}
interface Planet {
  mass: number
  pos: Pos
  prevPos: Pos
  totalDv: Pos
}
const planets: Ref<Planet[]> = ref([])
const reset = () => planets.value = [{
  mass: 200,
  pos: {x: 0, y: 0},
  prevPos: {x: 0, y: 0},
  totalDv: {x: 0, y: 0},
}];
reset();
const mouseDownStart: Ref<null | Pos> = ref(null)
const mousePos: Ref<null | Pos> = ref(null)
const onMouseDown = (event: Event) =>
  (mouseDownStart.value = { x: event.clientX, y: event.clientY })
const onMouseMove = (event: Event) =>
  (mousePos.value = { x: event.clientX, y: event.clientY })
const onMouseUp = (event: Event) => {
  if (!mouseDownStart.value) {
    console.log('no start')
    return
  }
  const start = mouseDownStart.value
  const end = { x: event.clientX, y: event.clientY }
  planets.value.push({
    mass: 1,
    pos: start,
    prevPos: {
      x: start.x - (end.x - start.x) * DT * SPEED_PER_DRAGGED_PX,
      y: start.y - (end.y - start.y) * DT * SPEED_PER_DRAGGED_PX
    }
  })
  mouseDownStart.value = null;
  mousePos.value = null;
}
const SIZE = { x: 1000, y: 800 }
const COLORS = ['yellow', 'red', 'green', 'blue', 'pink']
const displayedPlanets = computed(() =>
  planets.value.map((p, i) => ({
    pos: p.pos,
    // mass = 4/3*Math.Pi*Math.pow(radius, 3) * KG_PER_PIXEL_CUBE
    radius: Math.pow(p.mass * 3 / 4 / Math.PI / KG_PER_PIXEL_CUBE, 1/3),
    color: COLORS[i % COLORS.length]
  }))
)
const speedArrow = computed(() => {
  if (!mouseDownStart.value || !mousePos.value) {return null;}
  return {start: mouseDownStart.value, end: mousePos.value};
})
const totalMass = computed(() => planets.value.reduce((acc, p) => acc + p.mass, 0));

let lastPhysics: number|null = null;
const onFrame = (timemillis: number) => {
  if (lastPhysics === null) {lastPhysics = timemillis; requestAnimationFrame(onFrame);return;}
  while (lastPhysics + DT * 1000 < timemillis) {
    lastPhysics += DT * 1000;
    for (const p of planets.value) {
      p.totalDv = {x: 0, y: 0};
    }
    for (let i = 0; i < planets.value.length; i++) {
      for (let j = i + 1; j < planets.value.length; j++) {
        const a = planets.value[i];
        const b = planets.value[j];
        const ab = {x: b.pos.x - a.pos.x, y: b.pos.y - a.pos.y};
        const distanceSquared = Math.pow(ab.x, 2) + Math.pow(ab.y, 2);
        if (distanceSquared === 0) {
          continue;
        }
        const distance = Math.sqrt(distanceSquared);
        // energy after impulse is 1/2 * ma * (va + impulse/ma)^2 + (vb -impulse/mb)^2
        // variation is approx 
        const impulse = G * DT * a.mass * b.mass / distanceSquared;
        const abDir = {x: ab.x / distance, y: ab.y / distance};
        a.totalDv.x += abDir.x * impulse / a.mass;
        a.totalDv.y += abDir.y * impulse / a.mass;
        b.totalDv.x -= abDir.x * impulse / b.mass;
        b.totalDv.y -= abDir.y * impulse / b.mass;
      }
    }
    const centerOfGrav = { x: 0, y: 0};
    for (const p of planets.value) {
      const newPrevPos = {...p.pos};
      p.pos.x = 2 * p.pos.x - p.prevPos.x + p.totalDv.x * DT;
      p.pos.y = 2 * p.pos.y - p.prevPos.y + p.totalDv.y * DT;
      p.prevPos = newPrevPos;
      centerOfGrav.x += p.mass / totalMass.value * p.pos.x;
      centerOfGrav.y += p.mass / totalMass.value * p.pos.y;
    }
    for (const p of planets.value) {
      p.pos.x -= centerOfGrav.x - SIZE.x/2;
      p.prevPos.x -= centerOfGrav.x - SIZE.x/2;
      p.pos.y -= centerOfGrav.y - SIZE.y/2;
      p.prevPos.y -= centerOfGrav.y - SIZE.y/2;
    }
  }
  requestAnimationFrame(onFrame);
};
onMounted(() => {
  requestAnimationFrame(onFrame);
});
</script>

<template>
  <svg
    @mousedown="onMouseDown"
    @mouseup="onMouseUp"
    @mousemove="onMouseMove"
    :width="SIZE.x"
    :height="SIZE.y"
    style="background-color: black"
    :viewBox="'0 0 ' + SIZE.x + ' ' + SIZE.y"
    xmlns="http://www.w3.org/2000/svg"
  >
    <circle
      v-for="p in displayedPlanets"
      :cx="p.pos.x"
      :cy="p.pos.y"
      :r="p.radius"
      :fill="p.color"
    />
    <line v-if="speedArrow" :x1="speedArrow.start.x" :y1="speedArrow.start.y" :x2="speedArrow.end.x" :y2="speedArrow.end.y" stroke="red"/>
  </svg>
  <button @click="reset">RESET</button>
</template>

<style scoped>
button {
  position: absolute;
  top: 10px;
  left: 10px;
}
</style>
