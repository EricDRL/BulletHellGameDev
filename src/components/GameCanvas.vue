<template>
  <div>
    <canvas
      ref="canvas"
      :width="width"
      :height="height"
      style="background: gray"
    ></canvas>

    <div v-if="estado === 'menu'" class="menu" style="display: flex; flex-direction: column; align-items: center; padding-bottom: 20px;">
      <img src="../../public/LogoMenu.png" style="max-width: 500px" alt="LOGO" />
      <button style="min-width: 250px;" @click="iniciarJogo">Iniciar Jogo</button>
    </div>

    <!-- Vídeo da HISTÓRIA INICIAL -->
    <div
      v-if="estado === 'historinha'"
      class="historinha"
      @click="avancarVideo"
      style="position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: black; display: flex; justify-content: center; align-items: center; z-index: 10;"
    >
      <video
        ref="videoPlayer"
        :src="videosHistorinha[videoIndex]"
        autoplay
        muted
        playsinline
        @ended="avancarVideo"
        style="position: absolute; top: 0; left: 0; width: 100vw; height: 100vh; object-fit: cover; cursor: pointer;"
      ></video>
      <div
        style="position: absolute; bottom: 20px; color: white; font-size: 18px; width: 100%; text-align: center; user-select: none;"
      >
      
      </div>
    </div>

    <!-- Vídeo da MORTE (GAME OVER) -->
    <div
      v-if="estado === 'gameover' && mostrandoVideoMorte"
      @click="sairDoVideoMorte"
      style="position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: black; display: flex; justify-content: center; align-items: center; z-index: 20;"
    >
      <video
        ref="videoMortePlayer"
        :src="videoMorte"
        autoplay
        muted
        playsinline
        @ended="sairDoVideoMorte"
        style="position: absolute; top: 0; left: 0; width: 100vw; height: 100vh; object-fit: cover; cursor: pointer; z-index: 21;"
      ></video>
      <div
        style="position: absolute; bottom: 20px; color: white; font-size: 18px; width: 100%; text-align: center; user-select: none; z-index: 22;"
      >
        
      </div>
    </div>

    <div v-if="estado === 'gameover' && !mostrandoVideoMorte" class="gameover" style="text-align: center; margin-top: 20px;">
      <h1 style="color: white">Game Over</h1>
      <button @click="reiniciarJogo">Reiniciar Jogo</button>
    </div>

  </div>
</template>


<script>




export default {
  name: "GameCanvas",
 data() {
  return {
    historinhaJaVista: false, 
    fundos: [],
    fundoAtual: null,
    width: window.innerWidth,
    height: window.innerHeight,
    playerX: window.innerWidth / 2,
    playerY: window.innerHeight / 2,
    projectiles: [],
    powerUps: [],
    inimigos: [],
    animationId: null,
    projectileInterval: null,
    tempoInterval: null,
    keysPressed: {},
    tempo: 0,
    pontos: 0,
    nivel: 1,
    velocidadeProjeteis: 3,
    vidas: 3,
    slowAtivo: false,
    slowTimeoutId: null,
    trocaFaseDelay: false,
    boss: null,
    bossDirecao: 1,

    // Estados e vídeos
    estado: "menu",  // só uma vez!
    videoIndex: 0,
    videoMorteIndex: 0,
    mostrandoVideoMorte: false,

    videosHistorinha: [
      new URL('../assets/videos/video1.mp4', import.meta.url).href,
      new URL('../assets/videos/video2.mp4', import.meta.url).href,
      new URL('../assets/videos/video3.mp4', import.meta.url).href,
      new URL('../assets/videos/video4.mp4', import.meta.url).href,
      new URL('../assets/videos/video5.mp4', import.meta.url).href,
      new URL('../assets/videos/video6.mp4', import.meta.url).href,
      new URL('../assets/videos/video7.mp4', import.meta.url).href,
      new URL('../assets/videos/video8.mp4', import.meta.url).href,
      new URL('../assets/videos/video9.mp4', import.meta.url).href,
      new URL('../assets/videos/video10.mp4', import.meta.url).href,
    ],

    // URL do vídeo de morte
    videoMorte: new URL('../assets/videos/videomorte.mp4', import.meta.url).href,


  };
},


    

 mounted() {
  const carregarFundo = (num) =>
    new Promise((resolve) => {
      const img = new Image();
      img.src = new URL(`../assets/background/bg${num}.png`, import.meta.url).href;

      img.onload = () => resolve(img);
    });

  Promise.all([3, 1, 2, 4, 5].map(carregarFundo)).then((imgs) => {
    this.fundos = imgs;
    this.fundoAtual = imgs[0];
  });

  // Setup controles só uma vez
  this.setupControles();
},


  methods: {
 iniciarJogo() {
  if (!this.historinhaJaVista) {
    this.estado = "historinha";
    this.videoIndex = 0;
    this.historinhaJaVista = true; // marca que já viu
  } else {
    // já viu a historinha, vai direto para o jogo
    this.estado = "jogando";
    this.inicializarJogo();
  }
},


  
  avancarVideo() {
  if (this.videoIndex < this.videosHistorinha.length - 1) {
    // Avança para o próximo vídeo, continua na tela de vídeo
    this.videoIndex++;
  } else {
    // Acabou os vídeos, começa o jogo
    this.estado = "jogando";
    this.playerX = this.width / 2;
    this.playerY = this.height / 2;
    this.projectiles = [];
    this.powerUps = [];
    this.tempo = 0;
    this.pontos = 0;
    this.nivel = 1;
    this.fundoAtual = this.fundos[0];
    this.velocidadeProjeteis = 3;
    this.vidas = 3;
    this.slowAtivo = false;
    this.trocaFaseDelay = false;
    this.boss = null;
    this.bossDirecao = 1;
    this.setupInimigos();
    this.$nextTick(() => {
      this.setupControles();
      this.iniciarTimer();
      this.iniciarLoop();
    });
  }
},

  inicializarJogo() {
  cancelAnimationFrame(this.animationId);
  clearInterval(this.projectileInterval);
  clearInterval(this.tempoInterval);
console.log("Velocidade dos projéteis:", this.velocidadeProjeteis);

  this.playerX = this.width / 2;
  this.playerY = this.height / 2;
  this.projectiles = [];
  this.powerUps = [];
  this.tempo = 0;
  this.pontos = 0;
  this.nivel = 1;
  this.fundoAtual = this.fundos[0];
  this.velocidadeProjeteis = 3;
  this.vidas = 3;
  this.slowAtivo = false;
  this.trocaFaseDelay = false;
  this.boss = null;
  this.bossDirecao = 1;
  this.setupInimigos();
  this.$nextTick(() => {
    this.setupControles();
    this.iniciarTimer();
    this.iniciarLoop();
  });
},



    mostrarVideoMorte() {
  console.log("Mostrando vídeo de morte:", this.videoMorte);
  this.estado = "gameover";
  this.mostrandoVideoMorte = true;
  // Opcional: parar animação e timers do jogo
  cancelAnimationFrame(this.animationId);
  clearInterval(this.projectileInterval);
  clearInterval(this.tempoInterval);
},


 sairDoVideoMorte() {
  this.mostrandoVideoMorte = false;
  this.estado = "jogando";
  this.velocidadeProjeteis = 3;  // reset explicito da velocidade
  this.vidas = 3;                // reset vidas tambem
  this.inicializarJogo();
},


 reiniciarJogo() {
  this.mostrandoVideoMorte = false;
  this.estado = "jogando";
  this.velocidadeProjeteis = 3;  // reset explicito da velocidade
  this.vidas = 3;                // reset vidas tambem
  this.inicializarJogo();
},

  setupControles() {
    window.addEventListener("keydown", (e) => {
      this.keysPressed[e.key] = true;
    });
    window.addEventListener("keyup", (e) => {
      this.keysPressed[e.key] = false;
    });
  },
    setupInimigos() {
      this.inimigos = [];
      const size = 40;
      const padding = 50;

      if (this.nivel < 5) {
        // inimigos normais só nas fases 1-4
        // Topo - 2 inimigos
        this.inimigos.push({
          x: this.width * 0.25,
          y: padding,
          size,
          lastShot: 0,
          shootInterval: Math.max(400, 1500 - this.nivel * 200),
        });
        this.inimigos.push({
          x: this.width * 0.75,
          y: padding,
          size,
          lastShot: 0,
          shootInterval: Math.max(400, 1500 - this.nivel * 200),
        });

        // Base - 2 inimigos
        this.inimigos.push({
          x: this.width * 0.25,
          y: this.height - padding,
          size,
          lastShot: 0,
          shootInterval: Math.max(400, 1500 - this.nivel * 200),
        });
        this.inimigos.push({
          x: this.width * 0.75,
          y: this.height - padding,
          size,
          lastShot: 0,
          shootInterval: Math.max(400, 1500 - this.nivel * 200),
        });

        // Esquerda - 1 inimigo
        this.inimigos.push({
          x: padding,
          y: this.height / 2,
          size,
          lastShot: 0,
          shootInterval: Math.max(400, 1500 - this.nivel * 200),
        });

        // Direita - 1 inimigo
        this.inimigos.push({
          x: this.width - padding,
          y: this.height / 2,
          size,
          lastShot: 0,
          shootInterval: Math.max(400, 1500 - this.nivel * 200),
        });
      } else {
        // fase 5 - cria o boss
        const bossSize = 80;
        this.boss = {
          x: this.width / 2,
          y: this.height / 2,
          size: bossSize,
          lastShot: 0,
          shootInterval: 400, // mais rápido
          vida: 30, // vida do boss
          velocidade: 2,
        };
      }
    },
    iniciarTimer() {
      clearInterval(this.tempoInterval);
      this.tempoInterval = setInterval(() => {
        if (this.estado !== "jogando") return;
console.log("Velocidade dos projéteis:", this.velocidadeProjeteis);

        this.tempo++;
        if (this.tempo % 15 === 0 && this.nivel < 5 && !this.trocaFaseDelay) {
          this.trocaFaseDelay = true;
          setTimeout(() => {
            this.nivel++;
            this.fundoAtual = this.fundos[this.nivel - 1];
            this.pontos += 25; // Pontos ao passar de fase
            this.velocidadeProjeteis += 1;

            if (this.nivel === 5) {
              this.projectiles = []; // Limpa projéteis antigos ao entrar na fase 5
            }

            this.setupInimigos();
            this.trocaFaseDelay = false;
          }, 3000); // Delay de 3 segundos na troca de fase
        }
        this.pontos += 10; // Pontos a cada segundo
      }, 1000);
    },

    iniciarLoop() {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");

     const animate = () => {
 if (this.estado === "menu" || (this.estado === "gameover" && !this.mostrandoVideoMorte)) {
  // Para o loop e timers somente no menu ou gameover sem vídeo ativo
  cancelAnimationFrame(this.animationId);
  clearInterval(this.projectileInterval);
  clearInterval(this.tempoInterval);
  return;
}

// Se estiver mostrando o vídeo de morte, NÃO cancela o loop para garantir que o canvas continue desenhando fundo ou tela preta, se quiser.
if (this.mostrandoVideoMorte) {
  // Opcionalmente, desenhe uma tela preta ou alguma coisa, ou deixe o vídeo DOM aparecer sobre o canvas
  ctx.fillStyle = "black";
  ctx.fillRect(0, 0, this.width, this.height);

  this.animationId = requestAnimationFrame(animate);
  return;
}



  // Desenha o fundo
  if (this.fundoAtual) {
    ctx.drawImage(this.fundoAtual, 0, 0, this.width, this.height);
  } else {
    ctx.fillStyle = "black";
    ctx.fillRect(0, 0, this.width, this.height);
  }

        const speed = 5;
        if (this.keysPressed["ArrowUp"] || this.keysPressed["w"])
          this.playerY -= speed;
        if (this.keysPressed["ArrowDown"] || this.keysPressed["s"])
          this.playerY += speed;
        if (this.keysPressed["ArrowLeft"] || this.keysPressed["a"])
          this.playerX -= speed;
        if (this.keysPressed["ArrowRight"] || this.keysPressed["d"])
          this.playerX += speed;

        this.playerX = Math.max(10, Math.min(this.width - 10, this.playerX));
        this.playerY = Math.max(10, Math.min(this.height - 10, this.playerY));

        

        // HUD (sobreposto ao jogo)
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

        if (this.nivel < 5) {
          // Inimigos normais
          this.inimigos.forEach((inimigo) => {
            ctx.fillStyle = "magenta";
            ctx.fillRect(
              inimigo.x - inimigo.size / 2,
              inimigo.y - inimigo.size / 2,
              inimigo.size,
              inimigo.size
            );

            // Atirar projeteis com intervalo
            const now = Date.now();
            if (now - inimigo.lastShot > inimigo.shootInterval) {
              inimigo.lastShot = now;
              // Cria projétil em direção ao player
              const dx = this.playerX - inimigo.x;
              const dy = this.playerY - inimigo.y;
              const length = Math.hypot(dx, dy);

              const speedProj = this.slowAtivo
                ? this.velocidadeProjeteis * 0.5
                : this.velocidadeProjeteis;

              this.projectiles.push({
                x: inimigo.x,
                y: inimigo.y,
                r: 6,
                xVel: (dx / length) * speedProj,
                yVel: (dy / length) * speedProj,
              });
            }
          });
        } else {
          // Fase 5 - boss
          // Movimentação vertical do boss
          if (this.boss) {
            this.boss.y += this.bossDirecao * this.boss.velocidade;
            if (
              this.boss.y + this.boss.size / 2 > this.height - 50 ||
              this.boss.y - this.boss.size / 2 < 50
            ) {
              this.bossDirecao *= -1;
            }

            ctx.fillStyle = "purple";
            ctx.fillRect(
              this.boss.x - this.boss.size / 2,
              this.boss.y - this.boss.size / 2,
              this.boss.size,
              this.boss.size
            );

            // Boss atira mais rápido e projéteis mais rápidos
            const now = Date.now();
            if (now - this.boss.lastShot > this.boss.shootInterval) {
              this.boss.lastShot = now;
              const dx = this.playerX - this.boss.x;
              const dy = this.playerY - this.boss.y;
              const length = Math.hypot(dx, dy);

              const bossSpeed = this.slowAtivo
                ? this.velocidadeProjeteis * 0.75
                : this.velocidadeProjeteis * 2;

              this.projectiles.push({
                x: this.boss.x,
                y: this.boss.y,
                r: 10,
                xVel: (dx / length) * bossSpeed,
                yVel: (dy / length) * bossSpeed,
              });
            }
          }
        }

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
  this.mostrarVideoMorte(); // chama o método para mostrar o vídeo e mudar o estado
  return false; // para remover o projétil
}


            return false;
          }

          let cor = "red";
          if (this.nivel === 2) cor = "lime";
          else if (this.nivel === 3) cor = "blue";
          else if (this.nivel === 4) cor = "yellow";
          else if (this.nivel === 5) cor = "purple";

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
            } else if (pu.type === "slow") {
              if (this.slowTimeoutId) clearTimeout(this.slowTimeoutId);
              this.slowAtivo = true;
              this.slowTimeoutId = setTimeout(() => {
                this.slowAtivo = false;
              }, 3000); // 3 segundos de slow
            }
            return false; // Remove o power-up após pegar
          }

          ctx.beginPath();
          ctx.arc(pu.x, pu.y, pu.r, 0, Math.PI * 2);
          ctx.fillStyle = pu.type === "life" ? "green" : "yellow"; // Verde para vida, amarelo para slow
          ctx.fill();
          ctx.closePath();

          return pu.y <= this.height + 50; // Só mantém os power-ups dentro da tela
        });

        this.animationId = requestAnimationFrame(animate);
      };

      // Criar power-ups aleatórios (life e slow)
      this.projectileInterval = setInterval(() => {
        if (this.estado !== "jogando") return;

        // spawn power-up
        const spawnChance = Math.random();
        if (spawnChance < 0.05) {
          const powerUp = {
            x: Math.random() * this.width,
            y: -50,
            r: 10,
            type: Math.random() < 0.5 ? "life" : "slow",
          };
          this.powerUps.push(powerUp);
        }
      }, 300);

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
  z-index: 0;
}

.menu,
.gameover {
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
