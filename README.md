# Crash Game

Uma simulação do jogo "Crash" (jogo do foguetinho) implementada em Vue.js.

## Características

- Saldo inicial de R$ 20
- Controle de aposta com botões de incremento/decremento
- Animação do foguete subindo
- Gráfico em tempo real
- Sistema de multiplicador
- Mecânica de crash realista

## Como executar

1. Instale as dependências:
```bash
npm install
```

2. Execute o servidor de desenvolvimento:
```bash
npm run dev
```

3. Abra o navegador em `http://localhost:3000`

## Como jogar

1. Use os botões + e - para ajustar o valor da sua aposta
2. Clique em "Apostar" para iniciar uma rodada
3. O multiplicador começará a subir
4. Clique em "Retirar" antes do crash para garantir seus ganhos
5. Se o foguete crashar antes de você retirar, você perde a aposta

## Tecnologias utilizadas

- Vue.js 3
- Chart.js
- Vite 