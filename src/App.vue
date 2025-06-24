<template>
  <div class="app">
    <div class="game-container">
      <game-header :balance="balance" />
      
      <game-area
        :is-playing="isPlaying"
        :has-crashed="hasCrashed"
        :current-multiplier="currentMultiplier"
        :chart-data="chartData"
        :chart-options="chartOptions"
        :rocket-path="rocketPath"
      />

      <game-controls
        :bet-amount="betAmount"
        :balance="balance"
        :is-playing="isPlaying"
        :loading="loading"
        :current-multiplier="currentMultiplier"
        @decrease-bet="decreaseBet"
        @increase-bet="increaseBet"
        @game-action="handleGameAction"
      />
    </div>
  </div>
</template>

<script>
import { ref, reactive } from 'vue'
import GameHeader from './components/GameHeader.vue'
import GameArea from './components/GameArea.vue'
import GameControls from './components/GameControls.vue'
import { Chart as ChartJS, CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend } from 'chart.js'

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend)

export default {
  name: 'App',
  components: {
    GameHeader,
    GameArea,
    GameControls
  },
  setup() {
    const balance = ref(20)
    const betAmount = ref(5)
    const isPlaying = ref(false)
    const currentMultiplier = ref(1)
    const hasCrashed = ref(false)
    const loading = ref(false)
    const gameInterval = ref(null)
    const balanceHistory = ref([20])
    const rocketPath = ref([])
    const gamesPlayed = ref(0)
    
    const gameState = ref({
      totalGames: 0,
      recentResults: [], // Últimos 10 resultados
      streakCount: 0, // Sequência atual (positiva = wins, negativa = losses)
      lastMultipliers: [], // Últimos multiplicadores alcançados
      playerMomentum: 0, // -1 a 1, indica se está ganhando ou perdendo
      sessionPeak: 20, // Maior saldo da sessão
    
    })
    
    const chartData = reactive({
      labels: [],
      datasets: [{
        label: 'Multiplicador',
        data: [],
        borderColor: '#00ff94',
        backgroundColor: 'rgba(0, 255, 148, 0.1)',
        fill: true,
        tension: 0,
        borderWidth: 2,
        pointRadius: 0,
        stepped: 'before',
        segment: {
          borderColor: ctx => {
            const value = ctx.p1.parsed.y;
            if (value >= 5) return '#ff4444';
            if (value >= 3) return '#ffaa00';
            return '#00ff94';
          },
        }
      }]
    })

    const chartOptions = reactive({
      responsive: true,
      maintainAspectRatio: false,
      scales: {
        y: {
          beginAtZero: true,
          grid: {
            color: 'rgba(255, 255, 255, 0.1)'
          },
          ticks: {
            color: '#fff',
            callback: function(value) {
              return value.toFixed(1) + 'x';
            }
          },
          min: 1,
          max: 10,
        },
        x: {
          grid: {
            display: false
          },
          ticks: {
            display: false
          }
        }
      },
      plugins: {
        legend: {
          display: false
        },
        tooltip: {
          enabled: false
        }
      },
      animation: {
        duration: 0
      },
      elements: {
        line: {
          borderJoinStyle: 'round',
          borderCapStyle: 'round'
        }
      },
      interaction: {
        intersect: false,
        mode: 'index'
      }
    })

    const handleGameAction = () => {
      if (!isPlaying.value) {
        if (balance.value >= betAmount.value) {
          gameState.value.totalGames++
          
          // Estados do jogo
          isPlaying.value = true
          loading.value = false
          currentMultiplier.value = 1
          balance.value -= betAmount.value
          
          // Limpa dados anteriores do chart
          chartData.labels.length = 0
          chartData.datasets[0].data.length = 0
          rocketPath.value.length = 0
          
          updateChart()
          
          requestAnimationFrame(() => {
            startGame()
          })
        }
      } else {
        // Jogador retirou - sempre ganha!
        const winAmount = betAmount.value * currentMultiplier.value
        balance.value += winAmount
        
        // Atualiza estado positivo
        updateGameState(true, currentMultiplier.value)
        
        endGame(false)

        gamesPlayed.value++
      }
    }

    const startGame = () => {
      const crashPoint = generateCrashPoint()
      const gameSpeed = 50
      let multiplierIncrease = 0.01
      let lastUpdate = performance.now()

      gameInterval.value = setInterval(() => {
        const now = performance.now()
        const deltaTime = now - lastUpdate
        lastUpdate = now

        currentMultiplier.value += multiplierIncrease * (deltaTime / gameSpeed)
        updateChart()

        if (currentMultiplier.value >= crashPoint) {
          // Jogador perdeu
          updateGameState(false, crashPoint)
          endGame(true)
          gamesPlayed.value++
        }
      }, gameSpeed)
    }

    const generateCrashPoint = () => {
      let result =  Math.random() * 3;

      const random = Math.random() * 100; // Generate random percentage (0-100)
  
      // Define probability thresholds
      // 40% chance for 2x (0-40%)
      if (random < 40) {
        result = 1.5 + Math.random() * 0.5; // Crash between 1x and 2x
      } else if (random < 60) {
        // 20% chance for 4x (40-60%)
        result = 2 + Math.random() * 2; // Crash between 2x and 4x
      } else if (random < 70) {
        // 10% chance for 8x (60-70%)
        result = 4 + Math.random() * 4; // Crash between 4x and 8x
      }

      if (Math.random() <= 0.2) {
        result = 1.01;
      }

      if (gamesPlayed.value <= 0) {
        if (result < 2.5) {
          result = 2.5;
        }
      }

      if (result >= calculateMaxMultiplier()) {
        result = calculateMaxMultiplier() - 0.25;
      }

      return result;
    }

    //calculate mutliplier that will lead the current game the balance to R$ 80
    const calculateMaxMultiplier = () => {
      const maxMultiplier = (80 - balance.value) / betAmount.value
      return maxMultiplier
    }

    const updateGameState = (won, multiplier) => {
      // Atualiza histórico
      gameState.value.recentResults.push(won)
      if (gameState.value.recentResults.length > 10) {
        gameState.value.recentResults.shift()
      }
      
      gameState.value.lastMultipliers.push(multiplier)
      if (gameState.value.lastMultipliers.length > 5) {
        gameState.value.lastMultipliers.shift()
      }
      
      // Atualiza streak
      if (won) {
        gameState.value.streakCount = gameState.value.streakCount > 0 ? 
          gameState.value.streakCount + 1 : 1
      } else {
        gameState.value.streakCount = gameState.value.streakCount < 0 ? 
          gameState.value.streakCount - 1 : -1
      }
      
      // Calcula momentum
      const recentWins = gameState.value.recentResults.slice(-5).filter(r => r).length
      gameState.value.playerMomentum = (recentWins - 2.5) / 2.5 // -1 a 1
      
      // Estados especiais - mais restritivos
      gameState.value.coolingDown = gameState.value.streakCount <= -2
      
      // Atualiza pico da sessão
      if (balance.value > gameState.value.sessionPeak) {
        gameState.value.sessionPeak = balance.value
      }
    }

    const endGame = (crashed = false) => {
      if (gameInterval.value) {
        clearInterval(gameInterval.value)
        gameInterval.value = null
      }

      hasCrashed.value = crashed
      isPlaying.value = false

      // Adiciona saldo ao histórico
      balanceHistory.value.push(balance.value)
      if (balanceHistory.value.length > 10) {
        balanceHistory.value.shift()
      }

      // Remove qualquer boost artificial - deixe o saldo drenar naturalmente
      // Apenas um boost mínimo quando chega a 0 para manter o jogo funcionando
      if (balance.value <= 0) {
        balance.value = 1 // Mínimo para continuar
      }

      loading.value = crashed

      if (crashed) {
        setTimeout(() => {
          chartData.labels.length = 0
          chartData.datasets[0].data.length = 0
          currentMultiplier.value = 1
          hasCrashed.value = false
          loading.value = false
          rocketPath.value.length = 0
        }, 2000)
      } else {
        chartData.labels.length = 0
        chartData.datasets[0].data.length = 0
        currentMultiplier.value = 1
        rocketPath.value.length = 0
      }
    }

    const updateChart = () => {
      const maxPoints = 30
      
      const lastPoint = chartData.datasets[0].data[chartData.datasets[0].data.length - 1]
      if (lastPoint !== currentMultiplier.value) {
        chartData.labels.push('')
        chartData.datasets[0].data.push(currentMultiplier.value)
        
        if (chartData.labels.length > maxPoints) {
          chartData.labels.shift()
          chartData.datasets[0].data.shift()
        }
      }
    }

    const increaseBet = () => {
      if (betAmount.value < balance.value) {
        betAmount.value = Math.min(balance.value, betAmount.value + 1)
      }
    }

    const decreaseBet = () => {
      if (betAmount.value > 1) { // Permite apostar até R$1
        betAmount.value--
      }
    }

    return {
      balance,
      betAmount,
      isPlaying,
      currentMultiplier,
      hasCrashed,
      loading,
      chartData,
      chartOptions,
      rocketPath,
      handleGameAction,
      increaseBet,
      decreaseBet
    }
  }
}
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700&display=swap');

.app {
  max-width: 900px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Poppins', sans-serif;
  min-height: 100vh;
  background: #0f0f1a;
}

.game-container {
  background: linear-gradient(145deg, #1a1a2e, #16162b);
  border-radius: 20px;
  padding: 30px;
  color: white;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
}

@media (max-width: 768px) {
  .app {
    padding: 10px;
  }
  
  .game-container {
    padding: 15px;
  }
}
</style>