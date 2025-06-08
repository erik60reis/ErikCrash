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
    
    // Sistema mais agressivo para levar o saldo a zero
    const gameState = ref({
      totalGames: 0,
      recentResults: [], // Últimos 10 resultados
      streakCount: 0, // Sequência atual (positiva = wins, negativa = losses)
      lastMultipliers: [], // Últimos multiplicadores alcançados
      playerMomentum: 0, // -1 a 1, indica se está ganhando ou perdendo
      luckFactor: 0.5, // Fator de sorte que oscila
      sessionPeak: 20, // Maior saldo da sessão
      recoveryMode: false, // Modo recuperação quando saldo muito baixo
      hotStreak: false, // Se está numa sequência de sorte
      coolingDown: false, // Se está em período de "azar"
      aggressionLevel: 0.7, // Nível de agressividade do sistema (0-1)
      sabotageActivated: false, // Ativado após atingir R$60
      sabotage2Activated: false, // Ativado quando volta para R$35 ou menos
      sabotage2Force: 0.3, // Intensidade inicial do segundo sabotagem (0-1)
      cheatNextRound: false
    })

    // Inicializar dados do chart corretamente
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

        // NUNCA permitir ultrapassar R$80 após sabotagem
        if (gameState.value.sabotageActivated && balance.value >= 80) {
          balance.value = 79
        }
        
        // Atualiza estado positivo
        updateGameState(true, currentMultiplier.value)
        
        endGame(false)
      }
    }

    const startGame = () => {
      const crashPoint = calculateAggressiveCrashPoint()
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
        }
      }, gameSpeed)
    }

    const calculateAggressiveCrashPoint = () => {
      console.log('calculateAggressiveCrashPoint - cheatNextRound:', gameState.value.cheatNextRound)
      // Se cheatNextRound está ativado, forçar crash baixo (1.0x-1.03x)
      if (gameState.value.cheatNextRound) {
        gameState.value.cheatNextRound = false
        return 1 + Math.random() * 0.03
      }

      // Fase 0: sem sabotagem, jogo "fácil"
      if (!gameState.value.sabotageActivated) {
        return generateFriendlyCrashPoint()
      }

      // Fase 2: segundo sabotagem (mais suave mas crescente)
      if (gameState.value.sabotage2Activated) {
        return generateStage2CrashPoint(gameState.value.sabotage2Force)
      }

      // Sistema mais agressivo para drenar o saldo
      const balanceRatio = balance.value / 20 // Ratio do saldo inicial
      const momentum = gameState.value.playerMomentum
      
      // Base muito agressiva - 80% de chance de crash baixo
      let crashProbability = 0.8
      
      // Se o saldo está alto, seja ainda mais agressivo
      if (balance.value > 15) {
        crashProbability = 0.9
      } else if (balance.value > 10) {
        crashProbability = 0.85
      }
      
      // Se o jogador está ganhando muito, corte a sorte
      if (gameState.value.hotStreak) {
        crashProbability = 0.95
      }
      
      // Apenas dê uma folga quando o saldo está muito baixo
      if (balance.value <= 5) {
        crashProbability = 0.6 // Um pouco de alívio
      }
      
      // Se está quase sem dinheiro, permita uma pequena recuperação ocasional
      if (balance.value <= 2) {
        crashProbability = 0.5 // 50/50 chance
      }
      
      // Aumenta agressividade ao longo do tempo
      const sessionProgress = gameState.value.totalGames / 20
      crashProbability += sessionProgress * 0.1
      
      return generateAggressiveCrashPoint(Math.min(crashProbability, 0.95))
    }

    const generateAggressiveCrashPoint = (crashProbability) => {
      const random = Math.random()
      
      if (random < crashProbability * 0.5) {
        // Crash instantâneo ou muito baixo (1.0x - 1.2x) - muito frustrante
        return 1 + (Math.random() * 0.2)
      } else if (random < crashProbability * 0.8) {
        // Crash baixo (1.2x - 1.8x) - perda pequena mas constante
        return 1.2 + (Math.random() * 0.6)
      } else if (random < crashProbability * 0.95) {
        // Crash médio-baixo (1.8x - 2.5x) - falsa esperança
        return 1.8 + (Math.random() * 0.7)
      } else {
        // Crash alto raro (2.5x - 4x) - para dar esperança ocasional
        return 2.5 + (Math.random() * 1.5)
      }
    }

    const generateStage2CrashPoint = (force) => {
      // force varia aprox. 0.3 -> 0.95 enquanto o jogador continua <=35
      const crashProbability = Math.min(force, 0.95)
      const random = Math.random()

      if (random < crashProbability * 0.4) {
        return 1 + Math.random() * 0.5 // 1.0x -1.5x
      } else if (random < crashProbability * 0.8) {
        return 1.5 + Math.random() * 0.7 // 1.5x -2.2x
      } else if (random < crashProbability * 0.95) {
        return 2.2 + Math.random() * 0.8 // 2.2x -3x
      } else {
        return 3 + Math.random() * 2 // 3x -5x
      }
    }

    const generateFriendlyCrashPoint = () => {
      const random = Math.random()
      // Mais chances de multiplicadores altos para favorecer o jogador
      if (random < 0.2) {
        return 1 + (Math.random() * 0.3) // 1.0x - 1.3x (pequena perda)
      } else if (random < 0.5) {
        return 1.3 + (Math.random() * 1.2) // 1.3x - 2.5x
      } else if (random < 0.85) {
        return 2.5 + (Math.random() * 2) // 2.5x - 4.5x (boas vitórias)
      } else {
        return 4.5 + (Math.random() * 3) // 4.5x - 7.5x (raras grandes vitórias)
      }
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
      gameState.value.hotStreak = gameState.value.streakCount >= 2 // Reduzido de 3 para 2
      gameState.value.coolingDown = gameState.value.streakCount <= -2
      gameState.value.recoveryMode = balance.value < 5 // Aumentado de 8 para 5
      
      // Se retirou muito rápido, ativa azar no próximo round
      if (won && multiplier < 1.2 && balance.value >= 35) {
        console.log('updateGameState - setting cheatNextRound for multiplier', multiplier)
        gameState.value.cheatNextRound = true
      }
      
      // Ativa primeira sabotagem se atingir R$60 pela primeira vez
      if (!gameState.value.sabotageActivated && balance.value + betAmount.value >= 70) {
        gameState.value.sabotageActivated = true
      }

      // Transição para segunda sabotagem quando saldo cai para 35 ou menos
      if (
        gameState.value.sabotageActivated &&
        !gameState.value.sabotage2Activated &&
        balance.value <= 35
      ) {
        gameState.value.sabotage2Activated = true
        gameState.value.sabotage2Force = 0.3
      }

      // Se já está na segunda sabotagem e saldo continua <=35, aumente força
      if (gameState.value.sabotage2Activated && balance.value <= 35) {
        gameState.value.sabotage2Force = Math.min(
          0.95,
          gameState.value.sabotage2Force + 0.05
        )
      }
      
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