<template>
  <div class="controls">
    <div class="bet-container">
      <div class="bet-label">Sua Aposta</div>
      <div class="bet-controls">
        <button class="bet-button" @click="$emit('decrease-bet')">-</button>
        <span class="bet-amount">R$ {{ formatMoney(betAmount) }}</span>
        <button class="bet-button" @click="$emit('increase-bet')">+</button>
      </div>
    </div>
    
    <button 
      class="play-button" 
      :class="{ 'cashout-button': isPlaying }"
      @click="$emit('game-action')"
      :disabled="(loading && !isPlaying) || (!isPlaying && balance < betAmount)"
    >
      <span class="button-text">{{ isPlaying ? `Retirar (${currentMultiplier.toFixed(2)}x)` : 'Apostar' }}</span>
      <span class="button-shine"></span>
    </button>
  </div>
</template>

<script>
export default {
  name: 'GameControls',
  props: {
    betAmount: {
      type: Number,
      required: true
    },
    balance: {
      type: Number,
      required: true
    },
    isPlaying: {
      type: Boolean,
      required: true
    },
    loading: {
      type: Boolean,
      required: true
    },
    currentMultiplier: {
      type: Number,
      required: true
    }
  },
  methods: {
    formatMoney(value) {
      return value.toFixed(2)
    }
  }
}
</script>

<style scoped>
.controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 20px;
}

.bet-container {
  flex: 1;
}

.bet-label {
  font-size: 0.9em;
  color: #888;
  margin-bottom: 10px;
}

.bet-controls {
  display: flex;
  align-items: center;
  gap: 15px;
  background: rgba(255, 255, 255, 0.05);
  padding: 10px;
  border-radius: 10px;
}

.bet-button {
  background: rgba(255, 255, 255, 0.1);
  border: none;
  color: white;
  width: 40px;
  height: 40px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1.2em;
  transition: all 0.2s;
}

.bet-button:hover {
  background: rgba(255, 255, 255, 0.2);
}

.bet-amount {
  min-width: 100px;
  text-align: center;
  font-size: 1.2em;
  font-weight: 500;
}

.play-button {
  position: relative;
  background: linear-gradient(45deg, #00ff94, #00b8ff);
  border: none;
  color: white;
  padding: 15px 40px;
  border-radius: 10px;
  font-size: 1.2em;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;
  overflow: hidden;
}

.play-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(0, 255, 148, 0.3);
}

.play-button:disabled {
  background: #333;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.cashout-button {
  background: linear-gradient(45deg, #00b8ff, #0066ff);
}

.button-shine {
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(
    120deg,
    transparent,
    rgba(255, 255, 255, 0.3),
    transparent
  );
  animation: shine 3s infinite;
}

@keyframes shine {
  0% { left: -100%; }
  20% { left: 100%; }
  100% { left: 100%; }
}

@media (max-width: 768px) {
  .controls {
    flex-direction: column;
  }
  
  .bet-container {
    width: 100%;
  }
  
  .play-button {
    width: 100%;
  }
}
</style> 