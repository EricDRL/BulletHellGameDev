<template>
  <div>
    <!-- Menu -->
    <div v-if="estado === 'menu'" class="menu">
      <h1 style="color: white">Bullet Hell</h1>
      <button @click="iniciarJogo">Iniciar Jogo</button>
    </div>

    <!-- Canvas -->
    <canvas
      v-if="estado === 'jogando' || estado === 'gameover'"
      ref="canvas"
      :width="width"
      :height="height"
      style="background: black; cursor: none; position: absolute; top: 0; left: 0;"
    ></canvas>

    <!-- Game Over -->
    <div v-if="estado === 'gameover'" class="menu">
      <h1 style="color: white">Game Over</h1>
      <p style="color: white">Pontos: {{ pontos }}</p>
      <button @click="reiniciarJogo">Reiniciar</button>
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
      estado: "menu",
      animationId: null,
      projectileInterval: null,
      tempoInterval: null,
      keysPressed: {},
      tempo: 0,
      pontos: 0,
      nivel: 1,
      velocidadeProjeteis: 3
    };
  },
  methods: {
    iniciarJogo() {
      this.projectiles = [];
      this.estado = "jogando";
      this.tempo = 0;
      this.pontos = 0;
      this.nivel = 1;
      this.velocidadeProjeteis = 3;
      this.$nextTick(() => {
        this.setupControles();
        this.iniciarTimer();
        this.iniciarLoop();
      });
    },
    reiniciarJogo() {
      this.playerX = this.width / 2;
      this.playerY = this.height / 2;
      this.projectiles = [];
      this.estado = "jogando";
      this.tempo = 0;
      this.pontos = 0;
      this.nivel = 1;
      this.velocidadeProjeteis = 3;
      this.$nextTick(() => {
        this.setupControles();
        this.iniciarTimer();
        this.iniciarLoop();
      });
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
        if (this.tempo % 5 === 0 && this.nivel < 5) {
          this.nivel++;
          this.pontos += 100;
          this.velocidadeProjeteis += 1;
        }
      }, 1000);
    },
    iniciarLoop() {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");

      // Criação de projéteis
      const createProjectiles = () => {
        if (this.estado !== "jogando") return;
        const angle = Math.random() * 2 * Math.PI;
        const distance = 200;
        const startX = this.playerX + distance * Math.cos(angle);
        const startY = this.playerY + distance * Math.sin(angle);

        const dx = this.playerX - startX;
        const dy = this.playerY - startY;
        const length = Math.hypot(dx, dy);

        this.projectiles.push({
          x: startX,
          y: startY,
          r: 6,
          xVel: (dx / length) * this.velocidadeProjeteis,
          yVel: (dy / length) * this.velocidadeProjeteis
        });
      };

      this.projectileInterval = setInterval(createProjectiles, 300);

      const animate = () => {
        if (this.estado !== "jogando") {
          cancelAnimationFrame(this.animationId);
          clearInterval(this.projectileInterval);
          clearInterval(this.tempoInterval);
          return;
        }

        // Movimentação
        const speed = 5;
        if (this.keysPressed["ArrowUp"] || this.keysPressed["w"]) this.playerY -= speed;
        if (this.keysPressed["ArrowDown"] || this.keysPressed["s"]) this.playerY += speed;
        if (this.keysPressed["ArrowLeft"] || this.keysPressed["a"]) this.playerX -= speed;
        if (this.keysPressed["ArrowRight"] || this.keysPressed["d"]) this.playerX += speed;

        // Limites
        this.playerX = Math.max(10, Math.min(this.width - 10, this.playerX));
        this.playerY = Math.max(10, Math.min(this.height - 10, this.playerY));

        ctx.clearRect(0, 0, this.width, this.height);

        // Desenha jogador
        ctx.beginPath();
        ctx.arc(this.playerX, this.playerY, 10, 0, Math.PI * 2);
        ctx.fillStyle = "cyan";
        ctx.fill();
        ctx.closePath();

        // HUD
        ctx.fillStyle = "white";
        ctx.font = "20px Arial";
        ctx.fillText(`Tempo: ${this.tempo}s`, 20, 30);
        ctx.fillText(`Pontos: ${this.pontos}`, 20, 60);
        ctx.fillText(`Fase: ${this.nivel}`, 20, 90);

        // Atualiza projéteis
        this.projectiles = this.projectiles.filter((p) => {
          p.x += p.xVel;
          p.y += p.yVel;

          const dx = p.x - this.playerX;
          const dy = p.y - this.playerY;
          const distance = Math.hypot(dx, dy);

          if (distance < p.r + 10) {
            this.estado = "gameover";
            return false;
          }

          // Define cor baseada na fase
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

        this.animationId = requestAnimationFrame(animate);
      };

      animate();
    }
  }
};
</script>

<style>
body {
  margin: 0;
  overflow: hidden;
}
.menu {
  position: absolute;
  top: 40%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  z-index: 10;
}
button {
  padding: 10px 20px;
  font-size: 20px;
  margin-top: 20px;
  cursor: pointer;
}
</style>
