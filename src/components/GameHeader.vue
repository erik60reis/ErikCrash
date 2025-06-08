<template>
  <div class="header">
    <div class="logo">🚀 Erik Crash</div>
    <div class="balance-info">
      <div class="balance-label">Seu Saldo</div>
      <div class="balance-value">R$ {{ formatMoney(balance) }}</div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'GameHeader',
  props: {
    balance: {
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
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30px;
}

.logo {
  font-size: 2em;
  font-weight: 700;
  background: linear-gradient(45deg, #00ff94, #00b8ff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.balance-info {
  text-align: right;
}

.balance-label {
  font-size: 0.9em;
  color: #888;
  margin-bottom: 5px;
}

.balance-value {
  font-size: 1.5em;
  font-weight: 600;
  color: #00ff94;
}
</style> 