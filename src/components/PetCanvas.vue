<script setup lang="ts">
import { ref, computed } from "vue";
import { useConfigStore } from "../stores/config";

const props = defineProps<{ mood: "idle" | "talking" | "happy"; connected: boolean }>();
const cfg = useConfigStore();
const emit = defineEmits<{
  (e: "leftClick"): void;
  (e: "rightClick"): void;
  (e: "doubleClick"): void;
  (e: "dragMove", dx: number, dy: number): void;
  (e: "dragStart"): void;
  (e: "dragEnd"): void;
}>();

const DOUBLE_CLICK_MS = 260;
const DRAG_THRESHOLD_PX = 4;

let dragging = false;
let downX = 0;
let downY = 0;
let lastX = 0;
let lastY = 0;
let movedPastThreshold = false;

let pendingSingleClick: number | null = null;

const dragDirection = ref<"none" | "left" | "right">("none");
let dragDebounceTimer: number | null = null;
let proposedDirection: "none" | "left" | "right" = "none";

function setDragDirection(dir: "none" | "left" | "right") {
  if (dir === "none") {
    if (dragDebounceTimer !== null) {
      clearTimeout(dragDebounceTimer);
      dragDebounceTimer = null;
    }
    proposedDirection = "none";
    dragDirection.value = "none";
    return;
  }
  
  if (dragDirection.value === dir) {
    if (dragDebounceTimer !== null) {
      clearTimeout(dragDebounceTimer);
      dragDebounceTimer = null;
    }
    proposedDirection = dir;
    return;
  }

  if (proposedDirection !== dir) {
    proposedDirection = dir;
    if (dragDebounceTimer !== null) clearTimeout(dragDebounceTimer);
    dragDebounceTimer = window.setTimeout(() => {
      dragDirection.value = proposedDirection;
      dragDebounceTimer = null;
    }, 80); // Debounce to prevent jittering
  }
}

const spriteState = computed(() => {
  if (dragDirection.value === "left") return { row: 2, frames: 8 };
  if (dragDirection.value === "right") return { row: 1, frames: 8 };
  if (props.mood === "happy") return { row: 3, frames: 4 };
  if (props.mood === "talking") return { row: 4, frames: 5 };
  return { row: 0, frames: 6 }; // idle
});

const animationKey = computed(() => `${spriteState.value.row}-${spriteState.value.frames}`);

function clearPendingSingle() {
  if (pendingSingleClick !== null) {
    clearTimeout(pendingSingleClick);
    pendingSingleClick = null;
  }
}

function onPointerDown(e: PointerEvent) {
  if (e.button !== 0) return;
  (e.currentTarget as Element).setPointerCapture(e.pointerId);
  dragging = false;
  movedPastThreshold = false;
  downX = lastX = e.clientX;
  downY = lastY = e.clientY;
}

function onPointerMove(e: PointerEvent) {
  if (!(e.buttons & 1)) return; // left button not held
  if (!dragging && !movedPastThreshold) {
    if (Math.hypot(e.clientX - downX, e.clientY - downY) > DRAG_THRESHOLD_PX) {
      movedPastThreshold = true;
      dragging = true;
      emit("dragStart");
    }
  }
  if (dragging) {
    const dx = e.clientX - lastX;
    const dy = e.clientY - lastY;
    lastX = e.clientX;
    lastY = e.clientY;
    
    if (dx > 0) setDragDirection("right");
    else if (dx < 0) setDragDirection("left");

    if (dx !== 0 || dy !== 0) emit("dragMove", dx, dy);
  }
}

function onPointerUp(e: PointerEvent) {
  if (e.button !== 0) return;
  try {
    (e.currentTarget as Element).releasePointerCapture(e.pointerId);
  } catch {
    // ignore
  }
  if (dragging) {
    dragging = false;
    movedPastThreshold = false;
    setDragDirection("none");
    emit("dragEnd");
    return;
  }
  if (pendingSingleClick !== null) {
    clearPendingSingle();
    emit("doubleClick");
    return;
  }
  pendingSingleClick = window.setTimeout(() => {
    pendingSingleClick = null;
    emit("leftClick");
  }, DOUBLE_CLICK_MS);
}

function onPointerCancel() {
  if (dragging) {
    dragging = false;
    movedPastThreshold = false;
    setDragDirection("none");
    emit("dragEnd");
  }
}

function onContextMenu(e: MouseEvent) {
  e.preventDefault();
  clearPendingSingle();
  emit("rightClick");
}
</script>

<template>
  <div
    class="pet"
    :class="{
      talking: cfg.petTheme === 'Default' && mood === 'talking',
      happy: cfg.petTheme === 'Default' && mood === 'happy'
    }"
    @pointerdown="onPointerDown"
    @pointermove="onPointerMove"
    @pointerup="onPointerUp"
    @pointercancel="onPointerCancel"
    @contextmenu="onContextMenu"
  >
    <template v-if="cfg.petTheme === 'Default'">
      <span class="camel">🐫</span>
    </template>
    <template v-else>
      <div 
        class="sprite" 
        :key="animationKey"
        :style="{
          '--current-row': spriteState.row,
          '--current-frame-count': spriteState.frames,
          'animation-duration': (spriteState.frames * 0.18) + 's',
          'animation-timing-function': 'steps(' + spriteState.frames + ')'
        }"
      ></div>
    </template>
    <span class="dot" :class="{ on: connected }" />
  </div>
</template>

<style scoped>
.pet {
  width: 140px;
  height: 140px;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  cursor: grab;
  background: transparent;
  -webkit-user-select: none;
  user-select: none;
  touch-action: none;
  pointer-events: auto;
  animation: idle-float 4.8s ease-in-out infinite;
}
.pet:active {
  cursor: grabbing;
}

.sprite {
  width: 192px;
  height: 208px;
  background-image: url('../assets/sigrika-spritesheet.webp');
  background-size: 1536px 1872px;
  background-repeat: no-repeat;
  background-position-y: calc(var(--current-row) * -208px);
  animation-name: play-sprite;
  animation-iteration-count: infinite;
  transform: scale(0.65);
  pointer-events: none;
  flex-shrink: 0;
}

.dot {
  position: absolute;
  bottom: 6px;
  right: 14px;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #c0392b;
  box-shadow: 0 0 4px rgba(0, 0, 0, 0.4);
  opacity: 0.75;
  pointer-events: none;
}
.dot.on {
  background: #27ae60;
}

@keyframes idle-float {
  0%, 100% { transform: translateY(0) rotate(-1.5deg); }
  25%      { transform: translateY(-3px) rotate(0.5deg); }
  50%      { transform: translateY(0) rotate(1.5deg); }
  75%      { transform: translateY(-2px) rotate(-0.5deg); }
}

@keyframes play-sprite {
  from { background-position-x: 0; }
  to { background-position-x: calc(-1 * var(--current-frame-count) * 192px); }
}

.pet.talking {
  animation: walking 0.7s ease-in-out infinite;
}
.pet.talking .camel {
  animation: step-squash 0.7s ease-in-out infinite;
}
.pet.happy {
  animation: happy 0.45s ease-in-out 4;
}

.camel {
  font-size: 96px;
  line-height: 1;
  pointer-events: none;
  filter: drop-shadow(0 6px 10px rgba(0, 0, 0, 0.35))
          drop-shadow(0 0 2px rgba(0, 0, 0, 0.25));
  transform-origin: 50% 80%;
  animation: breathe 3.2s ease-in-out infinite;
}

@keyframes breathe {
  0%, 100% { transform: scale(1); }
  50%      { transform: scale(1.05); }
}

@keyframes walking {
  0%   { transform: translateX(0)    translateY(0)    rotate(-2deg); }
  25%  { transform: translateX(-3px) translateY(-5px) rotate(-5deg); }
  50%  { transform: translateX(0)    translateY(0)    rotate(-1deg); }
  75%  { transform: translateX(3px)  translateY(-5px) rotate(2deg); }
  100% { transform: translateX(0)    translateY(0)    rotate(-2deg); }
}

@keyframes step-squash {
  0%, 50%, 100% { transform: scale(1, 1); }
  25%           { transform: scale(1.04, 0.96); }
  75%           { transform: scale(0.97, 1.03); }
}

@keyframes happy {
  0%, 100% { transform: rotate(0deg) scale(1); }
  25%      { transform: rotate(-10deg) scale(1.06); }
  75%      { transform: rotate(10deg) scale(1.06); }
}
</style>

