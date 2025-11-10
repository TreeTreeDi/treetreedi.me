<template>
  <svg viewBox="0 0 100 100" fill="none" xmlns="http://www.w3.org/2000/svg">
    <title>Symbol</title>
    <g class="symbol-path">
      <path d="M 50 12 L 62 32 L 84 32 L 69 47 L 76 68 L 50 56 L 24 68 L 31 47 L 16 32 L 38 32 Z" />

      <circle cx="50" cy="50" r="20" />

      <path d="M 50 36 Q 60 38 60 43 Q 60 47 50 50 Q 40 53 40 57 Q 40 62 50 64" />

      <path d="M 50 12 L 50 24" />
      <path d="M 84 32 L 74 38" />
      <path d="M 76 68 L 66 60" />
      <path d="M 24 68 L 34 60" />
      <path d="M 16 32 L 26 38" />

      <circle cx="50" cy="50" r="4" />
    </g>
  </svg>
</template>

<style scoped>
@media (prefers-reduced-motion) {
  .symbol-path {
    animation: none !important;
    stroke-dasharray: unset !important;
    opacity: 1 !important;
  }
}
@media print {
  .symbol-path {
    animation: none !important;
    stroke-dasharray: unset !important;
    opacity: 1 !important;
  }
}

@keyframes draw-symbol {
  0% {
    stroke-dasharray: 0 450;
    opacity: 0;
  }
  3% {
    opacity: 1;
  }
  55% {
    stroke-dasharray: 450 0;
  }
  90% {
    stroke-dasharray: 450 0;
    opacity: 1;
  }
  100% {
    stroke-dasharray: 0 450;
    opacity: 0;
  }
}

.symbol-path {
  stroke-dasharray: 450 0;
  animation: draw-symbol 5s cubic-bezier(0.65, 0, 0.35, 1) infinite;
  transform-origin: center;
  stroke: #303030;
  stroke-width: 2.8;
  stroke-linecap: round;
  stroke-linejoin: round;
  fill: none;
}

.dark .symbol-path {
  stroke: #fdfdfd;
}
</style>
