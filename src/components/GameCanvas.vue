<template>
  <div class="game-container">
    <video
      v-if="showPhase5Background"
      :src="videoFundoFase5"
      autoplay
      loop
      muted
      playsinline
      class="background-video">
    </video>

    <canvas ref="canvas" :width="width" :height="height"></canvas>
    <canvas ref="collisionCanvas" :width="width" :height="height" style="display: none;"></canvas>

    <transition name="fade">
      <div v-if="estado === 'menu'" class="main-menu">
        <div class="menu-content">
          <img src="../assets/LogoMenu.png" class="game-logo" alt="LOGO" />

          <div class="button-group">
            <button class="menu-button" @click="iniciarOuContinuarJogo">
              Iniciar Jogo
            </button>
            <button class="menu-button" @click="estado = 'opcoes'">
              Opções
            </button>
            <button class="menu-button" @click="estado = 'creditos'">
              Créditos
            </button>
            <button class="menu-button" @click="sairJogo">
              Sair
            </button>
          </div>
        </div>
      </div>
    </transition>

    <transition name="fade">
      <div v-if="estado === 'historinha'" class="historinha" @click="avancarVideoHistorinha">
        <video ref="videoPlayer" :src="videosHistorinha[videoIndex]" autoplay muted playsinline @ended="avancarVideoHistorinha"
          class="video-player"></video>
        <button v-if="videoIndex < videosHistorinha.length - 1" class="skip-intro-button" @click.stop="pularVideoHistorinha">
          Pular história
        </button>
      </div>
    </transition>

    <transition name="fade">
      <div v-if="estado === 'cutsceneFase'" class="historinha" @click="avancarVideoCutsceneFase">
        <video ref="videoPlayerFase" :src="videosDeTransicao[currentCutsceneFaseIndex]" autoplay muted playsinline @ended="avancarVideoCutsceneFase"
          class="video-player"></video>
        <button class="skip-intro-button" @click.stop="pularCutsceneFase">
          Pular História
        </button>
      </div>
    </transition>

    <transition name="fade">
      <div v-if="estado === 'cutsceneFase3'" class="historinha" @click="avancarVideoCutsceneFase3">
        <video ref="videoPlayerFase3" :src="videosFase3[currentCutsceneFase3Index]" autoplay muted playsinline @ended="avancarVideoCutsceneFase3"
          class="video-player"></video>
        <button class="skip-intro-button" @click.stop="pularCutsceneFase3">
          Pular História
        </button>
      </div>
    </transition>

    <transition name="fade">
      <div v-if="estado === 'countdown'" class="countdown-screen">
        <h1 class="countdown-text">{{ countdownValue }}</h1>
      </div>
    </transition>

    <transition name="fade">
      <div v-if="estado === 'morte'" class="gameover-screen" @click="renascer">
        <video ref="videoMortePlayer" :src="videoMorte" autoplay muted playsinline @ended="renascer"
          class="video-player"></video>
      </div>
    </transition>

    <transition name="slide-up">
      <div v-if="estado === 'opcoes'" class="options-screen">
        <div class="options-content">
          <h2 class="screen-title">Opções</h2>
          <div class="option-group">
            <h3>Volume da Música</h3>
            <div class="volume-control">
              <label for="musicVolume">Volume:</label>
              <input type="range" id="musicVolume" min="0" max="100" v-model="volumeMusica" @input="updateVolume('musica', $event.target.value)">
              <span>{{ volumeMusica }}%</span>
            </div>
          </div>

          <div class="option-group">
            <h3>Controles</h3>
            <ul class="controls-list">
              <li>Movimento: <strong>W, A, S, D</strong></li>
            </ul>
          </div>
          <button class="menu-button back-button" @click="estado = 'menu'">Voltar ao Menu</button>
        </div>
      </div>
    </transition>

    <transition name="slide-down">
      <div v-if="estado === 'creditos'" class="credits-screen">
        <div class="credits-content">
          <h2 class="screen-title">Créditos</h2>
          <p class="credits-text">
            <strong>Desenvolvimento do Jogo:</strong> [Seu Nome ou Nome da Equipe]<br>
            <strong>Conceito e Design:</strong> [Seu Nome ou Nome da Equipe]<br>
            <strong>Arte e Gráficos:</strong> [Artistas/Ferramentas Utilizadas]<br>
            <strong>Música e Efeitos Sonoros:</strong> [Músicos/Fontes de Áudio]<br>
            <strong>Agradecimentos Especiais:</strong><br>
            Aos nossos professores, à comunidade Vue.js e a todos que nos apoiaram neste projeto!
          </p>
          <button class="menu-button back-button" @click="estado = 'menu'">Voltar ao Menu</button>
        </div>
      </div>
    </transition>

    <transition name="fade">
      <div v-if="estado === 'jogando'" class="in-game-ui">
      <!-- HUD do jogador -->
    <div class="game-hud">
      <div class="hud-container">
        <div class="hud-item">
          <span class="hud-label">Pontos:</span>
          <span class="hud-value">{{ pontos }}</span>
        </div>
        <div class="hud-item">
          <span class="hud-label">Fase:</span>
          <span class="hud-value">{{ faseAtualDoJogo }}</span>
        </div>
        <div class="hud-item">
          <span class="hud-label">Vida:</span>
          <div class="health-bar-container">
            <div class="health-bar" :style="{ width: (vidas/3)*100 + '%' }"></div>
          </div>
        </div>
        <div class="hud-item">
          <span class="hud-label">Tempo:</span>
          <span class="hud-value">{{ tempo }}s</span>
        </div>
      </div>
    </div>
    <!-- FIM DO HUD -->
        <button class="hamburger-button" @click="toggleGameMenu">
          ☰
        </button>
        <transition name="slide-right">
          <div v-if="inGameMenuOpen" class="in-game-menu">
            <h3 class="in-game-menu-title">Menu</h3>
            <div class="option-group-compact">
              <h3>Volume da Música</h3>
              <div class="volume-control-compact">
                <label for="musicVolumeInGame">Volume:</label>
                <input type="range" id="musicVolumeInGame" min="0" max="100" v-model="volumeMusica" @input="updateVolume('musica', $event.target.value)">
                <span>{{ volumeMusica }}%</span>
              </div>
            </div>
            <div class="in-game-menu-buttons">
              <button class="menu-button in-game-button" @click="voltarAoMenuPrincipal">Voltar ao Menu Principal</button>
              <button v-if="nivel < 5" class="menu-button in-game-button" @click="pularFase">
                Pular Fase
              </button>
              <button class="menu-button in-game-button" @click="toggleGameMenu">Continuar Jogo</button>
            </div>
          </div>
        </transition>
      </div>
    </transition>
  </div>
</template>
<script>
import { carregarSprites } from '../utils/carregarSprites.js';
import { carregarFundos } from '../utils/carregarFundos.js';
import { drawImage } from '../utils/drawImage.js';
// import { zonasPorFase } from '../utils/zonasPorFase.js';
// REMOVIDO: Não precisamos mais deste arquivo
// import { atualizarZonasDeColisao } from '../utils/colisao.js';
// REMOVIDO: Não precisamos mais desta função
// NOVO: Importa as novas funções de colisão por pixel e a colisão de projétil com player
import { verificarColisaoDePixel, verificarColisaoDeProjetilComPlayer } from '../utils/colisao.js';

import { gerarInimigosPorFase } from '../utils/gerarInimigosPorFase.js';
import { gerarBoss } from '../utils/gerarBoss.js';
import { atirarProjeteis } from '../utils/atirarProjeteis.js';

export default {
  name: "GameCanvas",
  data() {
    return {
      // zonasDeColisao: [], // REMOVIDO: Não precisamos mais de zonas estáticas
      fundos: [],
      fundoAtual: null,
      width: window.innerWidth,
      height: window.innerHeight,
      imagens: {},
      projectiles: [],
      powerUps: [],
      inimigos: [],
      estado: "menu", // "menu", "historinha", "jogando", "morte", "opcoes", "creditos", "cutsceneFase", "cutsceneFase3", "countdown"
      animationId: null,
      projectileInterval: null,
      projectileSpawnInterval: null,
      tempoInterval: null,
      keysPressed: {},
      tempo: 0,
      pontos: 0,
      pontosSalvos: 0,
      nivel: 1,
      velocidadeProjeteis: 3,
      velocidadeProjeteisPorFase: {
        1: 3,
        2: 4,
        3: 5,
        4: 6,
        5: 7,
      },
      vidas: 3,
      slowAtivo: false,
      slowTimeoutId: null,
      trocaFaseDelay: false,
      boss: null,
      bossDirecao: 1,
      bossDirecaoX: 1, // Direção horizontal do Boss
      bossState: 'attacking', // "attacking" ou "sleeping"
      bossTimers: { // Agrupa os timers do boss
        attack: null,
        rest: null,
      },
      player: {
        x: window.innerWidth / 2,
        y: window.innerHeight / 2,
        size: 120,
        hitboxRadius: 40
      },
      projectileSize: 48,
      videoIndex: 0,
      historinhaJaVista: false,
      currentCutsceneFaseIndex: 0,
      nextFaseAfterCutscene: 0,
      videosDeTransicao: [
        new URL('../assets/videos/video11.mp4', import.meta.url).href,
        new URL('../assets/videos/video12.mp4', import.meta.url).href,
        new URL('../assets/videos/video13.mp4', import.meta.url).href,
        new URL('../assets/videos/video14.mp4', import.meta.url).href,
        new URL('../assets/videos/video19.mp4', import.meta.url).href,
        new URL('../assets/videos/video20.mp4', import.meta.url).href,
        new URL('../assets/videos/video21.mp4', import.meta.url).href,
        new URL('../assets/videos/video22.mp4', import.meta.url).href,
        new URL('../assets/videos/video23.mp4', import.meta.url).href,
        new URL('../assets/videos/video24.mp4', import.meta.url).href,
        new URL('../assets/videos/video25.mp4', import.meta.url).href,
        new URL('../assets/videos/video26.mp4', import.meta.url).href,
        new URL('../assets/videos/video27.mp4', import.meta.url).href,
        new URL('../assets/videos/video28.mp4', import.meta.url).href,
        new URL('../assets/videos/video29.mp4', import.meta.url).href,
      ],
      cutsceneFaseMap: {
        2: { startIndex: 0, endIndex: 3 },
        4: { startIndex: 4, endIndex: 7 },
        5: { startIndex: 8, endIndex: 14 },
      },
      cutscenesFaseJaVistas: {
        2: false,
        3: false,
        4: false,
        5: false,
      },
      currentCutsceneFase3Index: 0,
      videosFase3: [
        new URL('../assets/videos/video15.mp4', import.meta.url).href,
        new URL('../assets/videos/video16.mp4', import.meta.url).href,
        new URL('../assets/videos/video17.mp4', import.meta.url).href,
        new URL('../assets/videos/video18.mp4', import.meta.url).href,
      ],
      cutsceneFase3JaVista: false,
      videoMorte: new URL('../assets/videos/videomorte.mp4', import.meta.url).href,
      videoFundoFase5: new URL('../assets/videos/fundo_fase5.mp4', import.meta.url).href, // Adicione o caminho real do seu vídeo
      volumeMusica: 50,
      inGameMenuOpen: false,
      bgMusic: null,
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
      countdownValue: 3,
      countdownIntervalId: null,
      faseAtualDoJogo: 1,

      // NOVO: Adicionado para o canvas de colisão
      collisionCanvas: null,
      collisionCtx: null,

      // NOVO: Imagens de colisão
      colisaoImages: {
        colisao1: null, // Para a Fase 2
        colisao2: null, // Para a Fase 5
        colisao3: null, // Para a Fase 1
        colisao4: null, // Para a Fase 4
      },
      // Mapeamento de fase para o nome da imagem de colisão
      faseColisaoMap: {
        1: 'colisao3',
        2: 'colisao1',
        4: 'colisao4',
        5: 'colisao2',
      },
    };
  },

  computed: {
    showPhase5Background() {
      // Esta propriedade computada controla a visibilidade do vídeo de fundo
      return this.faseAtualDoJogo === 5 && (this.estado === 'jogando' || this.estado === 'countdown');
    }
  },

  mounted() {
    // NOVO: Função para carregar as imagens de colisão
    const loadCollisionImages = () => {
      const promises = [];
      for (const key in this.colisaoImages) {
        if (Object.hasOwnProperty.call(this.colisaoImages, key)) {
          const img = new Image();
          img.src = new URL(`../assets/background/${key}.png`, import.meta.url).href;
          promises.push(new Promise((resolve, reject) => {
            img.onload = () => {
              this.colisaoImages[key] = img;
              resolve();
            };
            img.onerror = (e) => {
              console.error(`Erro ao carregar imagem de colisão ${key}.png:`, e);
              reject(e);
            };
          }));
        }
      }
      return Promise.all(promises);
    };
    // MODIFICADO: Inclui o carregamento das imagens de colisão no Promise.all
    Promise.all([carregarSprites(), carregarFundos(), loadCollisionImages()])
      .then(([sprites, fundos]) => {
        this.imagens = sprites;
        this.fundos = fundos;
        this.fundoAtual = fundos[0];
        console.log("Todas as imagens (sprites, fundos e colisões) carregadas com sucesso!");

        // NOVO: Inicializa o canvas de colisão
        this.collisionCanvas = this.$refs.collisionCanvas;
        this.collisionCtx = this.collisionCanvas.getContext('2d', { willReadFrequently: true }); // willReadFrequently para melhor performance de getImageData
        // Redimensiona o canvas de colisão para corresponder ao canvas principal
        this.collisionCanvas.width = this.width;
        this.collisionCanvas.height = this.height;

        // Adiciona a imagem do boss dormindo
        this.imagens.boss_sleeping = new Image();
        this.imagens.boss_sleeping.src = new URL('../assets/sprites/cartman2.png', import.meta.url).href;
        this.imagens.boss_sleeping.onload = () => {
            console.log("Imagem do boss_sleeping carregada!");
        };
        this.imagens.boss_sleeping.onerror = (e) => {
            console.error("Erro ao carregar a imagem do boss_sleeping:", e);
        };


        this.bgMusic = new Audio(new URL('../assets/audio/background_music.mp4', import.meta.url).href);
        this.bgMusic.loop = true;
        this.bgMusic.volume = this.volumeMusica / 100;
        this.bgMusic.play().catch(e => console.warn("Erro ao iniciar música de fundo:", e));
        const startMusicOnInteraction = () => {
          if (this.bgMusic && this.bgMusic.paused) {
            this.bgMusic.play().catch(e => console.warn("Erro ao tentar tocar música após interação:", e));
          }
          window.removeEventListener('click', startMusicOnInteraction);
          window.removeEventListener('keydown', startMusicOnInteraction);
        };
        window.addEventListener('click', startMusicOnInteraction);
        window.addEventListener('keydown', startMusicOnInteraction);
      })
      .catch(error => {
        console.error("Erro ao carregar recursos do jogo:", error);
        alert("Não foi possível carregar os recursos do jogo. Por favor, recarregue a página.");
      });
  },

  methods: {
    // NOVO: Método para configurar o canvas de colisão com a imagem correta
    setupCollisionCanvas(fase) {
      const collisionImageName = this.faseColisaoMap[fase];
      const collisionImage = this.colisaoImages[collisionImageName];

      if (this.collisionCtx && collisionImage) {
        this.collisionCtx.clearRect(0, 0, this.width, this.height);
        this.collisionCtx.drawImage(collisionImage, 0, 0, this.width, this.height);
        // console.log(`Canvas de colisão configurado para fase ${fase} com imagem ${collisionImageName}.`);
      } else if (this.collisionCtx) {
        // Limpa o canvas de colisão se não houver imagem específica para a fase (ex: fase 3)
        this.collisionCtx.clearRect(0, 0, this.width, this.height);
        // console.log(`Canvas de colisão limpo para fase ${fase}.`);
      }
    },

    iniciarOuContinuarJogo() {
      if (!this.historinhaJaVista) {
        this.estado = "historinha";
        this.videoIndex = 0;
        this.$nextTick(() => {
          if (this.$refs.videoPlayer) this.$refs.videoPlayer.play();
        });
      } else {
        this.iniciarGameplay(this.faseAtualDoJogo);
      }
    },

    avancarVideoHistorinha() {
      if (this.videoIndex < this.videosHistorinha.length - 1) {
        this.videoIndex++;
      } else {
        this.historinhaJaVista = true;
        this.iniciarGameplay(this.faseAtualDoJogo);
      }
    },

    pularVideoHistorinha() {
      this.historinhaJaVista = true;
      this.iniciarGameplay(this.faseAtualDoJogo);
    },

    mostrarCutsceneFase(proximaFase) {
        const cutsceneInfo = this.cutsceneFaseMap[proximaFase];
        if (cutsceneInfo && !this.cutscenesFaseJaVistas[proximaFase]) {
            this.estado = "cutsceneFase";
            this.currentCutsceneFaseIndex = cutsceneInfo.startIndex;
            this.nextFaseAfterCutscene = proximaFase;
            this.limparTimers();
            this.$nextTick(() => {
                const videoPlayerFase = this.$refs.videoPlayerFase;
                if (videoPlayerFase) {
                    videoPlayerFase.load();
                    videoPlayerFase.play().catch(e => console.warn("Erro ao tocar vídeo da cutscene de fase:", e));
                }
            });
        } else {
            this.iniciarContagemRegressiva(proximaFase);
        }
    },

    avancarVideoCutsceneFase() {
        const cutsceneInfo = this.cutsceneFaseMap[this.nextFaseAfterCutscene];
        if (cutsceneInfo && this.currentCutsceneFaseIndex < cutsceneInfo.endIndex) {
            this.currentCutsceneFaseIndex++;
            this.$nextTick(() => {
                const videoPlayerFase = this.$refs.videoPlayerFase;
                if (videoPlayerFase) {
                    videoPlayerFase.load();
                    videoPlayerFase.play().catch(e => console.warn("Erro ao tocar o próximo vídeo da cutscene de fase:", e));
                }
            });
        } else {
            this.cutscenesFaseJaVistas[this.nextFaseAfterCutscene] = true;
            this.iniciarContagemRegressiva(this.nextFaseAfterCutscene);
        }
    },

    pularCutsceneFase() {
      const videoPlayerFase = this.$refs.videoPlayerFase;
      if (videoPlayerFase) {
        videoPlayerFase.pause();
        videoPlayerFase.currentTime = 0;
      }
      this.cutscenesFaseJaVistas[this.nextFaseAfterCutscene] = true;
      this.iniciarContagemRegressiva(this.nextFaseAfterCutscene);
    },

    iniciarCutsceneFase3() {
      this.estado = "cutsceneFase3";
      this.currentCutsceneFase3Index = 0;
      this.limparTimers();
      this.$nextTick(() => {
        const videoPlayerFase3 = this.$refs.videoPlayerFase3;
        if (videoPlayerFase3) {
          videoPlayerFase3.play().catch(e => console.warn("Erro ao tocar vídeo da cutscene da Fase 3:", e));
        }
      });
    },

    avancarVideoCutsceneFase3() {
      if (this.currentCutsceneFase3Index < this.videosFase3.length - 1) {
        this.currentCutsceneFase3Index++;
        this.$nextTick(() => {
          const videoPlayerFase3 = this.$refs.videoPlayerFase3;
          if (videoPlayerFase3) {
            videoPlayerFase3.load();
            videoPlayerFase3.play().catch(e => console.warn("Erro ao tocar o próximo vídeo da cutscene da Fase 3:", e));
          }
        });
      } else {
        this.cutsceneFase3JaVista = true;
        this.iniciarContagemRegressiva(3);
      }
    },

    pularCutsceneFase3() {
      const videoPlayerFase3 = this.$refs.videoPlayerFase3;
      if (videoPlayerFase3) {
        videoPlayerFase3.pause();
        videoPlayerFase3.currentTime = 0;
      }
      this.cutsceneFase3JaVista = true;
      this.iniciarContagemRegressiva(3);
    },

    iniciarContagemRegressiva(proximaFaseAlvo) {
      this.fundoAtual = this.fundos[proximaFaseAlvo - 1] || this.fundos[0];
      this.estado = 'countdown';

      // Força o desenho do novo fundo no canvas
      const canvas = this.$refs.canvas;
      if (canvas && this.fundoAtual) {
          const ctx = canvas.getContext('2d');
          ctx.clearRect(0, 0, this.width, this.height); // Limpa a tela
          // Desenha o fundo da próxima fase (a menos que seja a fase 5, que usa vídeo)
          if (proximaFaseAlvo !== 5) {
            ctx.drawImage(this.fundoAtual, 0, 0, this.width, this.height);
          }
      }

      this.countdownValue = 3;
      if (this.countdownIntervalId) {
          clearInterval(this.countdownIntervalId);
      }

      this.countdownIntervalId = setInterval(() => {
        if (this.countdownValue > 1) {
          this.countdownValue--;
        } else if (this.countdownValue === 1) {
          this.countdownValue = 'JÁ!';
          setTimeout(() => {
            clearInterval(this.countdownIntervalId);
            this.countdownIntervalId = null;
            this.avancarParaProximaFase(proximaFaseAlvo);
          }, 800);
        } else {
          clearInterval(this.countdownIntervalId);
          this.countdownIntervalId = null;
        }
      }, 1000);
    },

    avancarParaProximaFase(novaFase) {
      this.pontosSalvos = this.pontos;
      this.nivel = novaFase;
      this.faseAtualDoJogo = novaFase;
      this.projectiles = []; // Clear projectiles from previous phase [cite: 88]
      // this.zonasDeColisao = atualizarZonasDeColisao(this.nivel, zonasPorFase()); // REMOVIDO: Não usa mais zonas estáticas
      this.setupCollisionCanvas(this.nivel);
      // NOVO: Desenha a imagem de colisão da nova fase
      this.pontos += 25;
      this.velocidadeProjeteis = this.velocidadeProjeteisPorFase[this.nivel];
      // Updated player position for phase 5 [cite: 23]
      if (this.nivel === 5) {
        this.player.x = 220;
        this.player.y = this.height / 2;
      }
      this.setupInimigos();
      this.estado = "jogando";
      this.iniciarTimer();
      this.iniciarLoop();
    },

    iniciarGameplay(faseParaIniciar) {
      this.estado = "jogando";
      this.nivel = faseParaIniciar;
      this.faseAtualDoJogo = faseParaIniciar;
      this.velocidadeProjeteis = this.velocidadeProjeteisPorFase[faseParaIniciar];
      // Update player position for phase 5 when starting gameplay [cite: 23]
      if (this.faseAtualDoJogo === 5) {
        this.player.x = 220;
        this.player.y = this.height / 2;
      } else {
        this.player.x = this.width / 2;
        this.player.y = this.height / 2;
      }
      // this.zonasDeColisao = atualizarZonasDeColisao(this.nivel, zonasPorFase());
      // REMOVIDO: Não usa mais zonas estáticas
      this.setupCollisionCanvas(this.nivel);
      // NOVO: Desenha a imagem de colisão da fase atual
      this.setupInimigos();
      this.$nextTick(() => {
        this.setupControles();
        this.iniciarTimer();
        this.iniciarLoop();
      });
    },

    renascer() {
      if (this.estado !== 'morte') return;
      setTimeout(() => {
        this.estado = 'jogando';
        const video = this.$refs.videoMortePlayer;
        if (video) {
          video.pause();
          video.currentTime = 0;
        }
        this.limparTimers();

        this.vidas = 3;
        if (this.faseAtualDoJogo === 1) {
          this.pontos = 0;
          this.pontosSalvos = 0;
          this.nivel = 1;
        } else {
          this.pontos = this.pontosSalvos;
        }
        this.tempo = 0;

        this.velocidadeProjeteis = this.velocidadeProjeteisPorFase[this.faseAtualDoJogo];
        this.fundoAtual = this.fundos[this.faseAtualDoJogo - 1] || this.fundos[0];

        // Updated player position for phase 5 on respawn [cite: 23]
        if (this.faseAtualDoJogo === 5) {
          this.player.x = 220;
          this.player.y = this.height / 2;
        } else {
          this.player.x = this.width / 2;
          this.player.y = this.height / 2;
        }
        this.projectiles = [];
        this.powerUps = [];
        this.slowAtivo = false;

        this.setupInimigos();
        this.setupCollisionCanvas(this.faseAtualDoJogo); // NOVO: Redesenha o canvas de colisão ao renascer
        this.iniciarTimer();
        this.iniciarLoop();
      }, 500);
    },

    resetarEstadoDoJogo() {
      const historinhaJaVistaTemp = this.historinhaJaVista;
      const imagensTemp = this.imagens;
      const fundosTemp = this.fundos;
      const volumeMusicaTemp = this.volumeMusica;
      const bgMusicTemp = this.bgMusic;
      const cutsceneFase3JaVistaTemp = this.cutsceneFase3JaVista;
      // NOVO: Salva as imagens de colisão temporariamente
      const colisaoImagesTemp = this.colisaoImages;
      this.limparTimers();
      Object.assign(this.$data, this.$options.data.call(this));
      this.historinhaJaVista = historinhaJaVistaTemp;
      this.imagens = imagensTemp;
      this.fundos = fundosTemp;
      this.volumeMusica = volumeMusicaTemp;
      this.bgMusic = bgMusicTemp;
      this.cutsceneFase3JaVista = cutsceneFase3JaVistaTemp;
      // NOVO: Restaura as imagens de colisão
      this.colisaoImages = colisaoImagesTemp;
      this.nivel = 1;
      this.faseAtualDoJogo = 1;
      this.velocidadeProjeteis = this.velocidadeProjeteisPorFase[1];
      this.pontos = 0;
      this.pontosSalvos = 0;
      this.cutscenesFaseJaVistas = { 2: false, 3: false, 4: false, 5: false };
      this.cutsceneFase3JaVista = false;
      if (this.bgMusic) {
        this.bgMusic.volume = this.volumeMusica / 100;
        if (this.bgMusic.paused) {
          this.bgMusic.play().catch(e => console.warn("Erro ao tentar tocar música após reset:", e));
        }
      }
      this.fundoAtual = this.fundos[this.nivel - 1] || this.fundos[0];
      this.setupCollisionCanvas(this.faseAtualDoJogo);
      // NOVO: Configura o canvas de colisão no reset
    },

    limparTimers() {
      if (this.tempoInterval) {
        clearInterval(this.tempoInterval);
        this.tempoInterval = null;
      }
      if (this.projectileInterval) {
        clearInterval(this.projectileInterval);
        this.projectileInterval = null;
      }
      if (this.projectileSpawnInterval) {
        clearInterval(this.projectileSpawnInterval);
        this.projectileSpawnInterval = null;
      }
      if (this.slowTimeoutId) {
        clearTimeout(this.slowTimeoutId);
        this.slowTimeoutId = null;
      }
       if (this.bossTimers.attack) {
        clearTimeout(this.bossTimers.attack);
        this.bossTimers.attack = null;
      }
      if (this.bossTimers.rest) {
        clearTimeout(this.bossTimers.rest);
        this.bossTimers.rest = null;
      }
      cancelAnimationFrame(this.animationId);
      this.animationId = null;
      if (this.countdownIntervalId) {
          clearInterval(this.countdownIntervalId);
          this.countdownIntervalId = null;
      }
    },

    setupControles() {
      if (this._handleKeyDown) {
        window.removeEventListener("keydown", this._handleKeyDown);
      }
      if (this._handleKeyUp) {
        window.removeEventListener("keyup", this._handleKeyUp);
      }

      this._handleKeyDown = (e) => { this.keysPressed[e.key] = true; };
      this._handleKeyUp = (e) => { this.keysPressed[e.key] = false; };

      window.addEventListener("keydown", this._handleKeyDown);
      window.addEventListener("keyup", this._handleKeyUp);
    },

    setupInimigos() {
      const size = 80;
      const padding = 50;
      if (this.faseAtualDoJogo === 5) {
        this.inimigos = [];
        this.boss = gerarBoss(this.width, this.height, 400);
        this.iniciarCicloBoss(); // Inicia o ciclo de ataque/descanso
      } else {
        this.boss = null;
        this.inimigos = gerarInimigosPorFase(this.faseAtualDoJogo, this.width, this.height, size, padding);
      }
    },

    iniciarTimer() {
      clearInterval(this.tempoInterval);
      this.tempoInterval = setInterval(() => {
        if (this.estado !== "jogando" || this.inGameMenuOpen) return;

        this.tempo++;
        if (this.tempo % 25 === 0 && this.nivel < 5 && !this.trocaFaseDelay) {
            this.trocaFaseDelay = true;

            setTimeout(() => {
                if (this.nivel === 1) {
                    this.mostrarCutsceneFase(this.nivel + 1);
                } else if (this.nivel === 2 && !this.cutsceneFase3JaVista) {
                    this.iniciarCutsceneFase3();
                } else if (this.nivel === 3) {
                  this.mostrarCutsceneFase(this.nivel + 1);
                } else if (this.nivel === 4) {
                    this.mostrarCutsceneFase(this.nivel + 1);
                }
                this.trocaFaseDelay = false;
            }, 3000);
        }
        this.pontos += 10;
      }, 1000);
    },

    iniciarLoop() {
      const canvas = this.$refs.canvas;
      if (!canvas) {
        console.error("Canvas não encontrado ao iniciar o loop!");
        return;
      }
      const ctx = canvas.getContext("2d");
      clearInterval(this.projectileInterval);
      this.projectileInterval = setInterval(() => {
        if (this.estado !== "jogando" || this.inGameMenuOpen) return;

        const spawnChance = Math.random();
        if (spawnChance < 0.05) {
          const powerUp = {
            x: Math.random() * this.width,
            y: -50,
            r: 20,
            type: Math.random() < 0.5 ? "life" : "slow",
            rotation: 0,
          };
          this.powerUps.push(powerUp);
        }
      }, 300);
      clearInterval(this.projectileSpawnInterval);
      
      // Removed 2 second delay before enemies start firing [cite: 139]
      this.projectileSpawnInterval = setInterval(() => {
        if (this.estado !== "jogando" || this.inGameMenuOpen) return;

        if (this.faseAtualDoJogo < 5) {
          this.inimigos.forEach(inimigo => {
              atirarProjeteis({
                atirador: inimigo,
                player: this.player,
                slowAtivo: this.slowAtivo,
                velocidadeProjeteis: this.velocidadeProjeteis,
                imagens: this.imagens,
                projectiles: this.projectiles
              }, { velocidadeMultiplicador: 1, width: this.projectileSize, height: this.projectileSize });
            });
          } else if (this.boss && this.bossState === 'attacking') {
            atirarProjeteis({
              atirador: this.boss,
              player: this.player,
              slowAtivo: this.slowAtivo,
              velocidadeProjeteis: this.velocidadeProjeteis,
              imagens: this.imagens,
              projectiles: this.projectiles
            }, { velocidadeMultiplicador: 2, width: this.projectileSize, height: this.projectileSize });
          }
        }, 700);

      cancelAnimationFrame(this.animationId);
      this.animate(ctx);
    },

    animate(ctx) {
      if (this.estado !== "jogando" || this.inGameMenuOpen) {
        cancelAnimationFrame(this.animationId);
        return;
      }

      ctx.clearRect(0, 0, this.width, this.height);
      // Desenha o fundo da imagem apenas se NÃO for a fase 5
      if (this.faseAtualDoJogo !== 5) {
          if (this.fundoAtual || this.imagens.fundoPadrao) {
            ctx.drawImage(this.fundoAtual || this.imagens.fundoPadrao, 0, 0, this.width, this.height);
          }
      }

      const speed = 5;
      const playerRadius = this.player.hitboxRadius;
      let newPlayerX = this.player.x;
      let newPlayerY = this.player.y;
      // Calcular novas posições baseadas nas teclas pressionadas
      if (this.keysPressed['w'] || this.keysPressed['W']) {
        newPlayerY -= speed;
      }
      if (this.keysPressed['s'] || this.keysPressed['S']) {
        newPlayerY += speed;
      }
      if (this.keysPressed['a'] || this.keysPressed['A']) {
        newPlayerX -= speed;
      }
      if (this.keysPressed['d'] || this.keysPressed['D']) {
        newPlayerX += speed;
      }

      // NOVO: Verificar colisão de pixel para a nova posição do jogador
      const isCollidingWithWalls = verificarColisaoDePixel(
        this.collisionCtx,
        newPlayerX,
        newPlayerY,
        playerRadius,
        this.width,
        this.height
      );
      // Atualizar a posição do jogador SOMENTE se não houver colisão
      if (!isCollidingWithWalls) {
        this.player.x = newPlayerX;
        this.player.y = newPlayerY;
      }

      // Manter jogador dentro dos limites da tela (sempre bom ter, mesmo com colisão de pixel)
      const metadePlayer = this.player.size / 2;
      this.player.x = Math.max(metadePlayer, Math.min(this.width - metadePlayer, this.player.x));
      this.player.y = Math.max(metadePlayer, Math.min(this.height - metadePlayer, this.player.y));

      if (this.imagens.player) {
        drawImage(ctx, this.imagens.player, this.player.x, this.player.y, this.player.size, this.player.size);
      }

      if (this.faseAtualDoJogo < 5) {
        this.processarInimigos(ctx);
      } else {
        this.processarBoss(ctx);
      }

      this.processarProjetiles(ctx);
      this.processarPowerUps(ctx);
      this.animationId = requestAnimationFrame(() => this.animate(ctx));
    },

    processarBoss(ctx) {
      if (this.bossState === 'attacking') { // Só move o boss se ele estiver atacando
        this.moverBoss();
      }
      if (!this.boss) return;
      // Usa a imagem normal ou a de dormir com base no estado
      const bossImage = this.bossState === 'sleeping' ?
        this.imagens.boss_sleeping : this.imagens.boss;

      if (bossImage) {
        drawImage(ctx, bossImage, this.boss.x, this.boss.y, this.boss.size, this.boss.size, 0);
      }
    },

    iniciarCicloBoss() {
      if (!this.boss) return;
      this.bossState = 'attacking';
      // Limpa timer anterior para evitar múltiplos ciclos rodando
      if(this.bossTimers.attack) clearTimeout(this.bossTimers.attack);
      if(this.bossTimers.rest) clearTimeout(this.bossTimers.rest);

      // Após 15s, o boss vai descansar
      this.bossTimers.attack = setTimeout(() => {
        this.bossState = 'sleeping';
        
        // Após 5s de descanso, ele volta a atacar
        this.bossTimers.rest = setTimeout(() => {
          this.iniciarCicloBoss(); // Reinicia o ciclo
        }, 5000); // 5 segundos de descanso

      }, 15000);
    },

    processarInimigos(ctx) {
      this.inimigos.forEach((inimigo, index) => {
        const imgSize = inimigo.size;
        const spriteIndex = (index % 5) + 1;
        const img = this.imagens['inimigo' + spriteIndex];

        if (img) {
          drawImage(ctx, img, inimigo.x, inimigo.y, imgSize, imgSize, 0);
        }
      });
    },

    processarProjetiles(ctx) {
      this.projectiles = this.projectiles.filter((p) => {
        p.x += p.xVel;
        p.y += p.yVel;

        const margin = 50;
        if (p.x < -margin || p.y < -margin || p.x > this.width + margin || p.y > this.height + margin) {
          return false;
        }

        // MODIFICADO: Usa a função de colisão de projétil com o player
        if (verificarColisaoDeProjetilComPlayer(p.x, p.y, p.r, this.player.x, this.player.y, this.player.hitboxRadius)) {
          this.vidas--;
          if (this.vidas <= 0) {
            this.estado = "morte";
            this.limparTimers();
          }
          return false;
        }

        // NOVO: Colisão de projéteis com as paredes (opcional, adicionei para exemplo)
        // Se desejar que projéteis também colidam com as paredes, descomente o bloco abaixo.
        // if (verificarColisaoDePixel(this.collisionCtx, p.x, p.y, p.r, this.width, this.height)) {
        //   return false;
        // Remove o projétil se colidir com a parede
        // }
        
        p.rotation = (p.rotation || 0) + 0.05;
        if (p.img) {
          drawImage(ctx, p.img, p.x, p.y, p.width, p.height, p.rotation);
        }

        return true;
      });
    },

    processarPowerUps(ctx) {
      this.powerUps = this.powerUps.filter((pu) => {
        pu.y += 2;

        if (verificarColisaoDeProjetilComPlayer(pu.x, pu.y, pu.r, this.player.x, this.player.y, this.player.hitboxRadius)) {
          this.coletarPowerUp(pu);
          return false;
        }

        const img = pu.type === "life" ? this.imagens.vida : this.imagens.slow;
        const size = 60;
        pu.rotation = (pu.rotation || 0) + 0.03;
        if (img) {
          drawImage(ctx, img, pu.x, pu.y, size, size, pu.rotation);
        }

        return pu.y <= this.height + 50;
      });
    },

    coletarPowerUp(powerUp) {
      if (powerUp.type === "life") {
        this.vidas++;
        this.pontos += 50;
      } else if (powerUp.type === "slow") {
        this.slowAtivo = true;
        this.pontos += 20;

        if (this.slowTimeoutId) {
          clearTimeout(this.slowTimeoutId);
        }

        this.slowTimeoutId = setTimeout(() => {
          this.slowAtivo = false;
          this.slowTimeoutId = null;
        }, 5000);
      }
    },

    moverBoss() {
      if (!this.boss) return;
      // Atualiza as posições X e Y
      this.boss.x += this.bossDirecaoX * (this.boss.velocidade * 0.75);
      // Movimento horizontal um pouco mais lento
      this.boss.y += this.bossDirecao * this.boss.velocidade;
      const halfSize = this.boss.size / 2;

      // Verifica colisão com as bordas verticais
      if (this.boss.y + halfSize > this.height - 50 || this.boss.y - halfSize < 50) {
        this.bossDirecao *= -1;
      }

      // Verifica colisão com as bordas horizontais
      if (this.boss.x + halfSize > this.width || this.boss.x - halfSize < 0) {
        this.bossDirecaoX *= -1;
      }
    },

    sairJogo() {
      console.log("Saindo do jogo...");
      alert("O jogo seria fechado aqui! Mas como estamos no navegador, fica por isso mesmo. :)");
    },

    updateVolume(type, value) {
      if (type === 'musica') {
        this.volumeMusica = parseInt(value);
        if (this.bgMusic) {
          this.bgMusic.volume = this.volumeMusica / 100;
        }
      }
    },

    toggleGameMenu() {
      this.inGameMenuOpen = !this.inGameMenuOpen;
      if (this.inGameMenuOpen) {
        this.limparTimers();
        if (this.bgMusic) {
          this.bgMusic.pause();
        }
      } else {
        this.iniciarLoop();
        this.iniciarTimer();
        if (this.bgMusic) {
          this.bgMusic.play().catch(e => console.warn("Erro ao tentar tocar música ao fechar menu in-game:", e));
        }
      }
    },

    pularFase() {
      if (this.nivel >= 5) return;
      this.inGameMenuOpen = false;
      // Pausa a música para não tocar sobre o vídeo da cutscene
      if (this.bgMusic) {
        this.bgMusic.pause();
      }

      // Chama a função que mostra a cutscene e avança para o próximo nível
      this.mostrarCutsceneFase(this.nivel + 1);
    },

    voltarAoMenuPrincipal() {
      this.inGameMenuOpen = false;
      setTimeout(() => {
        this.estado = 'menu';
        this.limparTimers();
        this.resetarEstadoDoJogo();
        if (this.bgMusic) {
          this.bgMusic.currentTime = 0;
          this.bgMusic.play().catch(e => console.warn("Erro ao tocar música ao voltar ao menu principal:", e));
        }
      }, 500);
    },
  },

  beforeUnmount() {
    if (this._handleKeyDown) {
      window.removeEventListener("keydown", this._handleKeyDown);
    }
    if (this._handleKeyUp) {
      window.removeEventListener("keyup", this._handleKeyUp);
    }
    this.limparTimers();
    if (this.bgMusic) {
      this.bgMusic.pause();
      this.bgMusic = null;
    }
  }
};
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');
.game-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  font-family: 'Press Start 2P', cursive, Arial, sans-serif;
}

.background-video {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  object-fit: cover;
  z-index: -1;
/* Posiciona o vídeo atrás do canvas */
}

canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: transparent;
/* Garante que o canvas seja transparente */
  z-index: 0;
}

.main-menu {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background: linear-gradient(135deg, rgba(15, 0, 0, 0.95), rgba(40, 0, 0, 0.95), rgba(15, 0, 0, 0.95));
  background-image: url('data:image/svg+base64,PHN2ZyB2aWV3Qm94PSIwIDAgOCA4IiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciPjxwYXRoIGQ9MDBoOHYwSDB6TTMgMGgxVjFoLTF6TTIgMmgxVjNoLTF6TTUgMmgxVjNoLTF6TTIgNWgxVjZoLTF6TTUgNWgxVjZoLTF6TTMgN2gxVjhoLTF6IiBmaWxsPSIjMmYwMDAwIiBmaWxsLW9wYWNpdHk9IjAuMSIvPjwvc3ZnPg==');
  background-size: 8px 8px;
  z-index: 10;
}

.menu-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 50px;
  background: linear-gradient(145deg, #1a0000, #3a0000);
  border-radius: 20px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.7), 0 0 50px rgba(255, 0, 0, 0.2);
  animation: fadeInScale 0.7s ease-out forwards;
  border: 2px solid #5a0000;
  width: 90%;
/* Adicionado para ocupar mais espaço horizontal */
  max-width: 800px;
/* Aumentado o limite de largura */
}

.game-logo {
  width: 90%;
  max-width: 650px;
/* Aumentado o tamanho máximo do logo */
  margin-bottom: 40px;
  filter: drop-shadow(0 0 15px rgba(255, 50, 50, 0.7)) brightness(1.1);
  animation: logoPulse 2s infinite alternate ease-in-out;
}

@keyframes logoPulse {
  0% { transform: scale(1);
    filter: drop-shadow(0 0 15px rgba(255, 50, 50, 0.7));
  }
  100% { transform: scale(1.03);
    filter: drop-shadow(0 0 25px rgba(255, 0, 0, 0.9)); }
}

.button-group {
  display: flex;
  flex-direction: column;
  gap: 20px;
  width: 100%;
  max-width: 450px;
/* Aumentado o tamanho máximo dos botões */
}

.menu-button {
  padding: 18px 30px;
  font-size: 1.3em;
  font-weight: bold;
  color: #fff;
  background-color: #550000;
  border: 3px solid #ff5555;
  border-radius: 10px;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(.25,.8,.25,1);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.5);
  text-transform: uppercase;
  letter-spacing: 2px;
  text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.5);
  position: relative;
  overflow: hidden;
}

.menu-button::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255,255,255,0.2), transparent);
  transition: all 0.7s cubic-bezier(.25,.8,.25,1);
}

.menu-button:hover::before {
  left: 100%;
}

.menu-button:hover {
  background-color: #ff3333;
  transform: translateY(-5px) scale(1.02);
  box-shadow: 0 10px 25px rgba(255, 51, 51, 0.6), 0 0 30px rgba(255, 0, 0, 0.4);
  border-color: #fff;
}

.menu-button:active {
  transform: translateY(0) scale(0.98);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.4);
  background-color: #aa0000;
  border-color: #ff0000;
}

.historinha, .gameover-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: #000;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 20;
}

.video-player {
  width: 100%;
  height: 100%;
  object-fit: contain;
  background-color: black;
}

.skip-intro-button {
  position: absolute;
  top: 20px;
  left: 20px;
  padding: 10px 20px;
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.4);
  border-radius: 5px;
  cursor: pointer;
  font-size: 0.9em;
  transition: background-color 0.3s ease, border-color 0.3s ease;
  z-index: 25;
}

.skip-intro-button:hover {
  background-color: rgba(255, 50, 50, 0.7);
  border-color: #ffcccc;
}

.options-screen, .credits-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: linear-gradient(135deg, rgba(15, 0, 0, 0.95), rgba(40, 0, 0, 0.95), rgba(15, 0, 0, 0.95));
  background-image: url('data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgOCA4IiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciPjxwYXRoIGQ9MkYwMDAwIiBmaWxsLW9wYWNpdHk9IjAuMSIvPjwvc3ZnPg==');
  background-size: 8px 8px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  color: white;
  z-index: 10;
  padding: 20px;
  box-sizing: border-box;
}

.options-content, .credits-content {
    background: rgba(30, 0, 0, 0.85);
    padding: 40px;
    border-radius: 15px;
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.6), 0 0 30px rgba(255, 0, 0, 0.1);
    border: 2px solid #6a0000;
    width: 90%;
    max-width: 600px;
    text-align: center;
}

.screen-title {
  font-size: 3.5em;
  margin-bottom: 40px;
  color: #ff6666;
  text-shadow: 0 0 15px rgba(255, 100, 100, 0.8);
  letter-spacing: 3px;
  text-transform: uppercase;
}

.option-group {
  background: rgba(40, 0, 0, 0.7);
  padding: 25px 35px;
  border-radius: 10px;
  margin-bottom: 30px;
  width: 100%;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.5);
  border: 1px solid #6a0000;
}

.option-group h3 {
  font-size: 1.6em;
  margin-top: 0;
  margin-bottom: 20px;
  color: #ffcc00;
  text-align: center;
  text-shadow: 0 0 5px rgba(255, 204, 0, 0.5);
}

.volume-control {
  display: flex;
  align-items: center;
  gap: 15px;
  justify-content: center;
}

.volume-control label {
  font-size: 1.1em;
  color: #ccc;
  min-width: 70px;
  text-align: right;
}

.volume-control input[type="range"] {
  width: 60%;
  -webkit-appearance: none;
  height: 10px;
  background: #333;
  border-radius: 5px;
  outline: none;
  transition: background 0.2s ease;
}

.volume-control input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #ff3333;
  cursor: pointer;
  border: 3px solid #fff;
  box-shadow: 0 0 8px rgba(255, 51, 51, 0.7);
}

.volume-control span {
  font-weight: bold;
  color: #ffcc00;
  font-size: 1.2em;
  min-width: 50px;
  text-align: left;
}

.controls-list {
  list-style: none;
  padding: 0;
  text-align: center;
}

.controls-list li {
  font-size: 1.2em;
  margin-bottom: 12px;
  color: #eee;
  text-shadow: 1px 1px 2px rgba(0,0,0,0.5);
}

.credits-text {
    font-size: 1.1em;
    line-height: 1.8;
    color: #ddd;
    margin-bottom: 40px;
    text-align: left;
    white-space: pre-line;
}

.credits-text strong {
    color: #ffcc00;
}

.back-button {
  margin-top: 30px;
  width: auto;
  padding: 15px 40px;
}

.in-game-ui {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 5;
}

.hamburger-button {
  position: absolute;
  top: 25px;
  right: 25px;
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  border: 3px solid #ff5555;
  border-radius: 50%;
  width: 55px;
  height: 55px;
  font-size: 2em;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  z-index: 6;
  pointer-events: auto;
  transition: background-color 0.3s ease, border-color 0.3s ease, transform 0.2s ease;
  box-shadow: 0 0 10px rgba(255, 50, 50, 0.5);
}

.hamburger-button:hover {
  background-color: rgba(255, 51, 51, 0.8);
  border-color: #fff;
  transform: scale(1.05);
}
.hamburger-button:active {
    transform: scale(0.95);
}

.in-game-menu {
  position: absolute;
  top: 0;
  right: 0;
  width: 350px;
  height: 100vh;
  background: linear-gradient(90deg, rgba(15, 0, 0, 0.98), rgba(40, 0, 0, 0.98));
  box-shadow: -8px 0 20px rgba(0, 0, 0, 0.7);
  padding: 50px 30px;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  align-items: center;
  z-index: 10;
  pointer-events: auto;
}

.in-game-menu-title {
  font-size: 2.5em;
  color: #ffcc00;
  margin-bottom: 50px;
  text-shadow: 0 0 8px rgba(255, 204, 0, 0.6);
  letter-spacing: 2px;
}

.in-game-menu-buttons {
  display: flex;
  flex-direction: column;
  gap: 15px;
  width: 100%;
  margin-top: auto;
/* Empurra os botões para o final */
  padding-bottom: 30px;
}

.in-game-button {
    font-size: 1.1em;
    padding: 15px 20px;
}

.option-group-compact {
  width: 100%;
  margin-bottom: 40px;
  text-align: center;
  background: rgba(40, 0, 0, 0.7);
  padding: 20px;
  border-radius: 10px;
  border: 1px solid #6a0000;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.4);
}

.option-group-compact h3 {
  font-size: 1.4em;
  color: #ff3333;
  margin-bottom: 15px;
}

.volume-control-compact {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
}

.volume-control-compact label,
.volume-control-compact span {
  font-size: 1em;
  color: #ccc;
}

.volume-control-compact input[type="range"] {
  width: 180px;
  height: 8px;
  background: #333;
  border-radius: 5px;
  outline: none;
  transition: background 0.2s ease;
  -webkit-appearance: none;
}

.volume-control-compact input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: #ff3333;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 0 7px rgba(255, 50, 50, 0.6);
}

.volume-control-compact span {
  font-weight: bold;
  color: #ff6666;
  min-width: 35px;
  text-align: left;
}

/* Animações de transição */
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.5s ease;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}

.slide-up-enter-active, .slide-up-leave-active {
  transition: transform 0.6s cubic-bezier(0.165, 0.84, 0.44, 1);
}
.slide-up-enter-from, .slide-up-leave-to {
  transform: translateY(100vh);
}

.slide-down-enter-active, .slide-down-leave-active {
  transition: transform 0.6s cubic-bezier(0.165, 0.84, 0.44, 1);
}
.slide-down-enter-from, .slide-down-leave-to {
  transform: translateY(-100vh);
}

.slide-right-enter-active, .slide-right-leave-active {
  transition: transform 0.5s cubic-bezier(0.23, 1, 0.32, 1);
}
.slide-right-enter-from, .slide-right-leave-to {
  transform: translateX(100%);
}

.countdown-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 15;
}

.countdown-text {
  font-size: 12em;
  color: #ffcc00;
  text-shadow: 0 0 25px rgba(255, 204, 0, 0.8), 0 0 10px #000;
  animation: countdown-pulse 1s infinite;
}

@keyframes countdown-pulse {
  0% { transform: scale(1);
}
  50% { transform: scale(1.1); }
  100% { transform: scale(1);
}
}

@keyframes fadeInScale {
  from {
    opacity: 0;
    transform: scale(0.9);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

/* ESTILOS DO HUD */
.game-hud {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 5;
  pointer-events: none;
}

.hud-container {
  background-color: rgba(0, 0, 0, 0.7);
  border: 2px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  padding: 12px 15px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  font-family: 'Press Start 2P', cursive;
}

.hud-item {
  display: flex;
  align-items: center;
  margin-bottom: 8px;
  font-size: 0.8em;
  color: white;
  text-shadow: 1px 1px 2px black;
}

.hud-item:last-child {
  margin-bottom: 0;
}

.hud-label {
  min-width: 70px;
  color: #ffcc00;
}

.hud-value {
  margin-left: 10px;
}

.health-bar-container {
  flex-grow: 1;
  height: 12px;
  background-color: rgba(70, 70, 70, 0.7);
  border-radius: 4px;
  overflow: hidden;
  margin-left: 10px;
  border: 1px solid rgba(0, 0, 0, 0.5);
}

.health-bar {
  height: 100%;
  background-color: #ff3333;
  background-image: linear-gradient(to right, #ff0000, #ff6666);
  border-radius: 3px;
  transition: width 0.3s ease;
}

@media (max-width: 768px) {
  .hud-item {
    font-size: 0.7em;
  }
  
  .hud-label {
    min-width: 60px;
  }
  
  .health-bar-container {
    height: 10px;
  }
}
/* FIM DOS ESTILOS DO HUD */

</style>