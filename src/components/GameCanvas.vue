<template>
  <div>
    <canvas
      ref="canvas"
      :width="width"
      :height="height"
      style="background: black; cursor: none;"
    ></canvas>
    <div v-if="estado === 'menu'" class="menu">
      <h1 style="color: white;">Jogo Bullet Hell</h1>
      <button @click="iniciarJogo">Iniciar Jogo</button>
    </div>
    <div v-if="estado === 'gameover'" class="gameover">
      <h1 style="color: white;">Game Over</h1>
      <button @click="reiniciarJogo">Reiniciar Jogo</button>
    </div>
  </div>
</template>

<script>
export default {
  name: "GameCanvas",
  data() {
    return {
      width: window.innerWidth,
      height: window.innerHeight,
      playerX: window.innerWidth / 2,
      playerY: window.innerHeight / 2,
      projectiles: [],
      powerUps: [],
      estado: "menu",
      animationId: null,
      projectileInterval: null,
      tempoInterval: null,
      keysPressed: {},
      tempo: 0,
      pontos: 0,
      nivel: 1,
      velocidadeProjeteis: 3,
      vidas: 3,
    };
  },
  methods: {
    iniciarJogo() {
      this.playerX = this.width / 2;
      this.playerY = this.height / 2;
      this.projectiles = [];
      this.powerUps = [];
      this.estado = "jogando";
      this.tempo = 0;
      this.pontos = 0;
      this.nivel = 1;
      this.velocidadeProjeteis = 3;
      this.vidas = 3;
      this.$nextTick(() => {
        this.setupControles();
        this.iniciarTimer();
        this.iniciarLoop();
      });
    },
    reiniciarJogo() {
      this.iniciarJogo();
    },
    setupControles() {
      window.addEventListener("keydown", (e) => {
        this.keysPressed[e.key] = true;
      });
      window.addEventListener("keyup", (e) => {
        this.keysPressed[e.key] = false;
      });
    },
    iniciarTimer() {
      clearInterval(this.tempoInterval);
      this.tempoInterval = setInterval(() => {
        this.tempo++;
        if (this.tempo % 15 === 0 && this.nivel < 5) {
          this.nivel++;
          this.pontos += 25; // Pontos ao passar de fase
          this.velocidadeProjeteis += 1;
        }
        this.pontos += 10; // Pontos a cada 5 segundos
      }, 1000);
    },
    iniciarLoop() {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");

      const createProjectiles = () => {
        if (this.estado !== "jogando") return;

        const quantidade = 2 + Math.floor(Math.random() * 3); // 2 a 4 projéteis
        for (let i = 0; i < quantidade; i++) {
          const angle = Math.random() * 2 * Math.PI;
          const spawnDistance = 300 + Math.random() * 200;
          const startX = this.playerX + spawnDistance * Math.cos(angle);
          const startY = this.playerY + spawnDistance * Math.sin(angle);

          const dx = this.playerX - startX;
          const dy = this.playerY - startY;
          const length = Math.hypot(dx, dy);

          this.projectiles.push({
            x: startX,
            y: startY,
            r: 6,
            xVel: (dx / length) * this.velocidadeProjeteis,
            yVel: (dy / length) * this.velocidadeProjeteis,
          });
        }
      };

      const spawnPowerUp = () => {
        if (this.estado !== "jogando") return;

        const spawnChance = Math.random();
        if (spawnChance < 0.05) { // 5% chance de spawnar um power-up a cada ciclo
          const powerUp = {
            x: Math.random() * this.width,
            y: -50,
            r: 10,
            type: Math.random() < 0.5 ? "life" : "speed", // Tipo do power-up
          };
          this.powerUps.push(powerUp);
        }
      };

      this.projectileInterval = setInterval(() => {
        createProjectiles();
        spawnPowerUp();
      }, 300);

      const animate = () => {
        if (this.estado !== "jogando") {
          cancelAnimationFrame(this.animationId);
          clearInterval(this.projectileInterval);
          clearInterval(this.tempoInterval);
          return;
        }

        const speed = 5;
        if (this.keysPressed["ArrowUp"] || this.keysPressed["w"]) this.playerY -= speed;
        if (this.keysPressed["ArrowDown"] || this.keysPressed["s"]) this.playerY += speed;
        if (this.keysPressed["ArrowLeft"] || this.keysPressed["a"]) this.playerX -= speed;
        if (this.keysPressed["ArrowRight"] || this.keysPressed["d"]) this.playerX += speed;

        this.playerX = Math.max(10, Math.min(this.width - 10, this.playerX));
        this.playerY = Math.max(10, Math.min(this.height - 10, this.playerY));

        ctx.clearRect(0, 0, this.width, this.height);

        // HUD
        ctx.fillStyle = "white";
        ctx.font = "20px Arial";
        ctx.fillText(`Tempo: ${this.tempo}s`, 20, 30);
        ctx.fillText(`Pontos: ${this.pontos}`, 20, 60);
        ctx.fillText(`Fase: ${this.nivel}`, 20, 90);
        ctx.fillText(`Vidas: ${this.vidas}`, 20, 120);

        // Player
        ctx.beginPath();
        ctx.arc(this.playerX, this.playerY, 10, 0, Math.PI * 2);
        ctx.fillStyle = "cyan";
        ctx.fill();
        ctx.closePath();

        // Projéteis
        this.projectiles = this.projectiles.filter((p) => {
          p.x += p.xVel;
          p.y += p.yVel;

          const dx = p.x - this.playerX;
          const dy = p.y - this.playerY;
          const distance = Math.hypot(dx, dy);

          if (distance < p.r + 10) {
            this.vidas--;
            if (this.vidas <= 0) {
              this.estado = "gameover";
            }
            return false;
          }

          let cor = "red";
          if (this.nivel === 2) cor = "lime";
          else if (this.nivel === 3) cor = "blue";
          else if (this.nivel === 4) cor = "yellow";

          ctx.beginPath();
          ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2);
          ctx.fillStyle = cor;
          ctx.fill();
          ctx.closePath();

          return (
            p.x >= -50 &&
            p.y >= -50 &&
            p.x <= this.width + 50 &&
            p.y <= this.height + 50
          );
        });

        // Power-ups
        this.powerUps = this.powerUps.filter((pu) => {
          pu.y += 2; // Velocidade do power-up

          // Verificando colisão com o player
          const dx = pu.x - this.playerX;
          const dy = pu.y - this.playerY;
          const distance = Math.hypot(dx, dy);

          if (distance < pu.r + 10) {
            if (pu.type === "life") {
              this.vidas++; // Adiciona vida
            } else if (pu.type === "speed") {
              this.velocidadeProjeteis += 1; // Aumenta a velocidade dos projéteis
            }
            return false; // Remove o power-up após pegar
          }

          ctx.beginPath();
          ctx.arc(pu.x, pu.y, pu.r, 0, Math.PI * 2);
          ctx.fillStyle = "yellow"; // Cor dos power-ups
          ctx.fill();
          ctx.closePath();

          return pu.y <= this.height + 50; // Só mantém os power-ups dentro da tela
        });

        this.animationId = requestAnimationFrame(animate);
      };

      animate();
    },
  },
};
</script>

<style scoped>
canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: black;
  cursor: none;
  z-index: 0;
}

.menu, .gameover {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
  text-align: center;
  z-index: 1;
}

button {
  padding: 10px 20px;
  font-size: 16px;
  margin-top: 10px;
  cursor: pointer;
}
</style>
