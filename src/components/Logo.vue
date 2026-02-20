<template>
  <svg
    xmlns="http://www.w3.org/2000/svg"
    viewBox="0 0 24 24"
    width="28"
    height="28"
    fill="none"
    stroke="currentColor"
    stroke-width="1.5"
    stroke-linecap="round"
    stroke-linejoin="round"
    aria-label="TreeTreeDi Logo"
    role="img"
  >
    <title>TreeTreeDi Logo</title>
    <path
      class="anim-adv-cursive"
      pathLength="100"
      d="M 5 19 C 4 12, 4 6, 5 4 C 8 3, 9 5, 8 9 C 8 13, 4 15, 2 17 C 6 15, 8 16, 11 19 C 12 14, 15 9, 14 5 C 19 4, 21 6, 20 12 C 19 17, 15 21, 12 21 C 15 18, 19 18, 22 21"
    />
    <circle
      class="anim-stamp"
      cx="21"
      cy="20"
      r="2"
    />
  </svg>
</template>

<style scoped>
.anim-stamp {
  fill: #ef4444;
  stroke: none;
  transform-origin: 21px 20px;
  animation: stampPop 5s cubic-bezier(0.34, 1.56, 0.64, 1) infinite;
}

.anim-adv-cursive {
  stroke-dasharray: 100;
  animation: drawAdvancedCursive 5s ease-in-out infinite;
}

@media (prefers-reduced-motion) {
  .anim-adv-cursive,
  .anim-stamp {
    animation: none !important;
    stroke-dasharray: unset !important;
    transform: none !important;
    opacity: 1 !important;
  }
}

@media print {
  .anim-adv-cursive,
  .anim-stamp {
    animation: none !important;
    stroke-dasharray: unset !important;
    transform: none !important;
    opacity: 1 !important;
  }
}

@keyframes drawAdvancedCursive {
  0% {
    stroke-dashoffset: 100;
  }
  40%,
  55% {
    stroke-dashoffset: 0;
  }
  100% {
    stroke-dashoffset: -100;
  }
}

@keyframes stampPop {
  0%,
  35% {
    transform: scale(0) rotate(-45deg);
    opacity: 0;
  }
  45%,
  65% {
    transform: scale(1) rotate(0deg);
    opacity: 1;
  }
  85%,
  100% {
    transform: scale(0) rotate(45deg);
    opacity: 0;
  }
}
