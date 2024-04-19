<script lang="ts" setup>
import { onMounted } from 'vue';

type Particle = {
  opacity: number;
  x: number;
  y: number;
  radius: number;
  particleColor: string;
  velocity: { x: number; y: number };
};

let canvas: HTMLCanvasElement;
let container: HTMLElement;
let ctx: CanvasRenderingContext2D;

let interactionParticle: Particle | undefined;

let particles: Array<Particle> = [];

let animationFrame: number;
let createIntervalId: any = undefined;

const options = {
  velocity: 0.5, // the higher the faster
  density: 15000, // the lower the denser
  netLineDistance: 200,
  netLineColor: '#929292',
  particleColors: ['#aaa'],
};

const init = (elementId: string) => {
  const elem = document.getElementById(elementId);
  if (elem) {
    container = elem;
    canvas = document.createElement('canvas');
    sizeCanvas();
    container.appendChild(canvas);
    const context = canvas.getContext('2d');
    if (context) {
      ctx = context;
    } else {
      return;
    }

    initParticleNetwork();
    bindUiActions();

    // debounce the resize event
    window.addEventListener(
      'resize',
      debounce(() => {
        cancelAnimationFrame(animationFrame);
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        particles.length = 0;
        sizeCanvas();
        initParticleNetwork();
      }, 500)
    );
  } else {
    console.error('element not found');
  }
};

const sizeCanvas = function () {
  if (container.offsetWidth && container.offsetHeight) {
    canvas.width = container.offsetWidth;
    canvas.height = container.offsetHeight;
  }
};

const initParticleNetwork = () => {
  // Create particle objects
  createParticles(true);

  // Update canvas
  animationFrame = requestAnimationFrame(updateParticleNetwork.bind(this));
};

const updateParticleNetwork = () => {
  if (canvas) {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.globalAlpha = 1;

    // Draw connections
    for (let i = 0; i < particles.length; i++) {
      for (let j = particles.length - 1; j > i; j--) {
        let distance,
          p1 = particles[i],
          p2 = particles[j];

        // check very simply if the two points are even a candidate for further measurements
        distance = Math.min(Math.abs(p1.x - p2.x), Math.abs(p1.y - p2.y));
        if (distance > options.netLineDistance) {
          continue;
        }

        // the two points seem close enough, now let's measure precisely
        distance = Math.sqrt(Math.pow(p1.x - p2.x, 2) + Math.pow(p1.y - p2.y, 2));
        if (distance > options.netLineDistance) {
          continue;
        }

        ctx.beginPath();
        ctx.strokeStyle = options.netLineColor;
        ctx.globalAlpha = ((options.netLineDistance - distance) / options.netLineDistance) * p1.opacity * p2.opacity;
        ctx.lineWidth = 0.7;
        ctx.moveTo(p1.x, p1.y);
        ctx.lineTo(p2.x, p2.y);
        ctx.stroke();
      }
    }

    // Draw particles
    for (let index = 0; index < particles.length; index++) {
      const particle = particles[index];
      updateParticle(particle);
      drawParticle(particle);
    }

    if (options.velocity !== 0) {
      animationFrame = requestAnimationFrame(updateParticleNetwork.bind(this));
    }
  } else {
    cancelAnimationFrame(animationFrame);
  }
};

const createParticles = (isInitial: boolean) => {
  var quantity = (canvas.width * canvas.height) / options.density;

  if (isInitial) {
    var counter = 0;
    clearInterval(createIntervalId);
    createIntervalId = setInterval(function () {
      if (counter < quantity - 1) {
        // Create particle object
        const particle = createParticle();
        if (particle) {
          particles.push(particle);
        }
      } else {
        clearInterval(createIntervalId);
      }
      counter++;
    }, 250);
  } else {
    // Create particle objects
    for (var i = 0; i < quantity; i++) {
      const particle = createParticle();
      if (particle) {
        particles.push(particle);
      }
    }
  }
};

// Particle Functions
const createParticle = (overwriteX?: number, overwriteY?: number) => {
  const pos = generateRandomPosition();
  return {
    x: overwriteX || pos.x,
    y: overwriteY || pos.y,
    opacity: 0,
    radius: getLimitedRandom(1.5, 2.5, true),
    particleColor: options.particleColors[Math.floor(Math.random() * options.particleColors.length)],
    velocity: {
      x: (Math.random() - 0.5) * options.velocity,
      y: (Math.random() - 0.5) * options.velocity,
    },
  } as Particle;
};

const updateParticle = function (particle: Particle) {
  if (particle.opacity < 1) {
    particle.opacity += 0.01;
  } else {
    particle.opacity = 1;
  }
  // Change dir if outside map
  if (particle.x > canvas.width + 100 || particle.x < -100) {
    particle.velocity.x = -particle.velocity.x;
  }
  if (particle.y > canvas.height + 100 || particle.y < -100) {
    particle.velocity.y = -particle.velocity.y;
  }

  // Update position
  particle.x += particle.velocity.x;
  particle.y += particle.velocity.y;
};

const drawParticle = (particle: Particle) => {
  ctx.beginPath();
  ctx.fillStyle = particle.particleColor;
  ctx.globalAlpha = particle.opacity;
  ctx.arc(particle.x, particle.y, particle.radius, 0, 2 * Math.PI);
  ctx.fill();
};

const createInteractionParticle = () => {
  // Add interaction particle

  interactionParticle = createParticle();
  interactionParticle.velocity = {
    x: 0,
    y: 0,
  };
  particles.push(interactionParticle);
  return interactionParticle; // TODO: remove return if not needed
};

const removeInteractionParticle = () => {
  if (interactionParticle) {
    var index = particles.indexOf(interactionParticle);
    if (index > -1) {
      interactionParticle = undefined;
      particles.splice(index, 1);
    }
  }
};

// ui functions
const bindUiActions = () => {
  // Mouse / touch event handling
  let spawnQuantity = 3;
  let mouseIsDown = false;
  let touchIsMoving = false;

  canvas.addEventListener('mousemove', (e) => {
    if (!interactionParticle) {
      createInteractionParticle();
    }
    if (interactionParticle) {
      interactionParticle.x = e.offsetX;
      interactionParticle.y = e.offsetY;
    }
  });
  canvas.addEventListener('touchmove', (e) => {
    touchIsMoving = true;
    if (!interactionParticle) {
      createInteractionParticle();
    }
    if (interactionParticle) {
      interactionParticle.x = e.changedTouches[0].clientX;
      interactionParticle.y = e.changedTouches[0].clientY;
    }
  });

  canvas.addEventListener('mousedown', (_e: MouseEvent) => {
    mouseIsDown = true;
    var counter = 0;
    var quantity = spawnQuantity;
    var intervalId = setInterval(function () {
      if (mouseIsDown) {
        if (counter === 1) {
          quantity = 1;
        }
        for (var i = 0; i < quantity; i++) {
          if (interactionParticle) {
            const particle = createParticle(interactionParticle.x, interactionParticle.y);
            particles.push(particle);
          }
        }
      } else {
        clearInterval(intervalId);
      }
      counter++;
    }, 50);
  });
  canvas.addEventListener('touchstart', (e) => {
    setTimeout(function () {
      if (!touchIsMoving) {
        for (var i = 0; i < spawnQuantity; i++) {
          const particle = createParticle(e.changedTouches[0].clientX, e.changedTouches[0].clientY);
          particles.push(particle);
        }
      }
    }, 200);
  });

  canvas.addEventListener('mouseup', (_e: MouseEvent) => {
    mouseIsDown = false;
  });
  canvas.addEventListener('mouseout', (_e: MouseEvent) => {
    removeInteractionParticle();
  });
  canvas.addEventListener('touchend', (e) => {
    touchIsMoving = false;
    removeInteractionParticle();
  });
};

// helper functions

const generateRandomPosition = () => {
  return {
    x: Math.random() * canvas.width,
    y: Math.random() * canvas.height,
  };
};

const getLimitedRandom = (min: number, max: number, roundToInteger: boolean = false) => {
  var number = Math.random() * (max - min) + min;
  if (roundToInteger) {
    number = Math.round(number);
  }
  return number;
};

const debounce = (mainFunction: () => void, delay: number) => {
  let timer: NodeJS.Timeout;
  // Return an anonymous function that takes in any number of arguments
  return function () {
    // Clear the previous timer to prevent the execution of 'mainFunction'
    clearTimeout(timer);

    // Set a new timer that will execute 'mainFunction' after the specified delay
    timer = setTimeout(() => {
      mainFunction();
    }, delay);
  };
};

onMounted(() => {
  init('particle-network-animation');
});
</script>
<template>
  <div id="particle-network-animation"></div>
</template>
<style lang="scss" scoped>
#particle-network-animation {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: calc(100vh - var(--top-bar-height));
  z-index: 1;

  &::before {
    z-index: -2;
    content: '';
    position: absolute;
    background: transparent;
    top: 0;
    right: 0;
    bottom: 0;
    left: 0;
    opacity: 0.2;
  }
}
.glow {
  z-index: -1;
  position: fixed;
  top: 50%;
  left: 50%;
  background-image: radial-gradient(circle closest-side, rgba(255, 255, 255, 0.025), transparent);
}
$duration: 25s;
.glow-1 {
  width: 150vw;
  height: 150vh;
  margin-top: -75vh;
  margin-left: -75vw;
  animation: glow-1-move $duration linear infinite both;
}
@keyframes glow-1-move {
  from {
    transform: translate(-100%, 100%);
  }
  to {
    transform: translate(100%, -100%);
  }
}
.glow-2 {
  width: 100vw;
  height: 100vh;
  margin-top: -50vh;
  margin-left: -50vw;
  animation: glow-2-move $duration linear calc($duration / 3) infinite both;
}
@keyframes glow-2-move {
  from {
    transform: translate(-100%, 0%);
  }
  to {
    transform: translate(100%, 100%);
  }
}
.glow-3 {
  width: 120vw;
  height: 120vh;
  margin-top: -60vh;
  margin-left: -60vw;
  animation: glow-3-move $duration linear calc($duration / 3 * 2) infinite both;
}
@keyframes glow-3-move {
  from {
    transform: translate(100%, 100%);
  }
  to {
    transform: translate(0%, -100%);
  }
}
</style>
