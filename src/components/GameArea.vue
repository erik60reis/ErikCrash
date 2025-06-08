<template>
  <div class="game-area">
    <div class="stars"></div>
    <div class="rocket-path">
      <div
        v-for="(point, index) in rocketPath"
        :key="index"
        class="path-line"
        :class="{ 'fading': index < rocketPath.length - 10 }"
        :style="{
          width: Math.sqrt(Math.pow(point.x, 2) + Math.pow(point.y, 2)) + 'px',
          transform: `translate(50px, ${450 - point.y}px) rotate(${Math.atan2(-point.y, point.x) * (180 / Math.PI)}deg)`
        }"
      ></div>
    </div>
    <div class="rocket-container" :class="{ crashed: hasCrashed }">
      <div class="rocket" :style="rocketStyle">
        <div class="rocket-body">
          🚀
          <div class="flame" v-if="isPlaying && !hasCrashed"></div>
        </div>
        <div class="trail" v-if="isPlaying && !hasCrashed"></div>
      </div>
      <div class="crash-value" v-if="hasCrashed">
        {{ currentMultiplier.toFixed(2) }}x
      </div>
      <div class="multiplier" v-if="isPlaying && !hasCrashed">
        {{ currentMultiplier.toFixed(2) }}x
      </div>
    </div>
    
    <div class="chart-container">
      <line-chart :chart-data="chartData" :options="chartOptions" />
    </div>
  </div>
</template>

<script>
import { Line as LineChart } from 'vue-chartjs'
import { computed } from 'vue'

export default {
  name: 'GameArea',
  components: {
    LineChart
  },
  props: {
    isPlaying: {
      type: Boolean,
      required: true
    },
    hasCrashed: {
      type: Boolean,
      required: true
    },
    currentMultiplier: {
      type: Number,
      required: true
    },
    chartData: {
      type: Object,
      required: true
    },
    chartOptions: {
      type: Object,
      required: true
    },
    rocketPath: {
      type: Array,
      required: true
    }
  },
  setup(props) {
    const rocketStyle = computed(() => {
      if (!props.isPlaying) {
        return {
          transform: 'translate(0px, 0px) rotate(-45deg)',
          transition: 'transform 0.2s ease-out'
        }
      }
      
      const progress = props.currentMultiplier - 1
      const x = progress * 60
      const y = progress * 80
      const rotation = Math.sin(progress * 0.5) * 10 - 45

      return {
        transform: `translate(${x}px, -${y}px) rotate(${rotation}deg)`,
        transition: 'transform 0.1s linear'
      }
    })

    return {
      rocketStyle
    }
  }
}
</script>

<style scoped>
.game-area {
  position: relative;
  height: 450px;
  background: #13132380;
  border-radius: 15px;
  margin-bottom: 30px;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.stars {
  position: absolute;
  width: 100%;
  height: 100%;
  background-image: 
    radial-gradient(2px 2px at 20px 30px, #ffffff, rgba(0,0,0,0)),
    radial-gradient(2px 2px at 40px 70px, #ffffff, rgba(0,0,0,0)),
    radial-gradient(2px 2px at 50px 160px, #ffffff, rgba(0,0,0,0)),
    radial-gradient(2px 2px at 90px 40px, #ffffff, rgba(0,0,0,0)),
    radial-gradient(2px 2px at 130px 80px, #ffffff, rgba(0,0,0,0));
  background-repeat: repeat;
  animation: stars 15s linear infinite;
}

@keyframes stars {
  from { background-position: 0 0; }
  to { background-position: 0 1000px; }
}

.rocket-path {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.path-line {
  position: absolute;
  background: linear-gradient(
    to right,
    transparent,
    rgba(0, 255, 148, 0.1),
    rgba(0, 255, 148, 0.3),
    rgba(0, 255, 148, 0.1)
  );
  height: 3px;
  transform-origin: left center;
  opacity: 0.7;
  transition: all 0.3s;
  box-shadow: 
    0 0 5px rgba(0, 255, 148, 0.3),
    0 0 10px rgba(0, 255, 148, 0.2);
  border-radius: 1.5px;
}

.path-line.fading {
  opacity: 0.15;
  height: 2px;
}

.rocket-container {
  position: absolute;
  bottom: 20px;
  left: 50px;
  z-index: 2;
}

.rocket {
  position: relative;
  font-size: 2.5em;
  transition: transform 0.1s linear;
}

.rocket-body {
  position: relative;
}

.flame {
  position: absolute;
  bottom: -10px;
  left: 50%;
  transform: translateX(-50%);
  width: 10px;
  height: 20px;
  background: linear-gradient(to bottom, #ff4d4d, #ff9900);
  border-radius: 50%;
  animation: flame 0.2s infinite alternate;
}

@keyframes flame {
  from { height: 20px; opacity: 0.8; }
  to { height: 25px; opacity: 1; }
}

.trail {
  position: absolute;
  bottom: 0;
  left: 50%;
  width: 4px;
  height: 20px;
  background: linear-gradient(to bottom, rgba(255,255,255,0.8), transparent);
  animation: trail 0.2s infinite;
}

@keyframes trail {
  from { height: 20px; opacity: 0.8; }
  to { height: 30px; opacity: 0; }
}

.multiplier {
  position: absolute;
  top: -30px;
  left: 50%;
  transform: translateX(-50%);
  color: #00ff94;
  font-weight: bold;
  font-size: 1.2em;
  text-shadow: 0 0 10px rgba(0, 255, 148, 0.5);
}

.crash-value {
  position: absolute;
  top: -30px;
  left: 50%;
  transform: translateX(-50%);
  color: #ff4444;
  font-weight: bold;
  font-size: 1.5em;
  text-shadow: 0 0 10px rgba(255, 68, 68, 0.5);
}

.chart-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
  padding: 20px;
}

.crashed .rocket {
  color: #ff4444;
  transform: rotate(45deg);
  transition: all 0.3s ease-in;
}
</style> 