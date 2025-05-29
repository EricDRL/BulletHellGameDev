<template>
  <div class="game-container">
    <canvas ref="canvas" :width="width" :height="height"></canvas>

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
import { zonasPorFase } from '../utils/zonasPorFase.js';
import { atualizarZonasDeColisao, verificarColisaoDeProjetil } from '../utils/colisao.js';
import { verificarTeclaPressionada } from '../utils/verificarTeclaPressionada.js';
import { gerarInimigosPorFase } from '../utils/gerarInimigosPorFase.js';
import { gerarBoss } from '../utils/gerarBoss.js';
import { atirarProjeteis } from '../utils/atirarProjeteis.js'; // CORREÇÃO AQUI

export default {
  name: "GameCanvas",
  data() {
    return {
      zonasDeColisao: [],
      fundos: [],
      fundoAtual: null,
      width: window.innerWidth,
      height: window.innerHeight,
      imagens: {},
      projectiles: [],
      powerUps: [],
      inimigos: [],
      estado: "menu", // "menu", "historinha", "jogando", "morte", "opcoes", "creditos", "cutsceneFase", "countdown"
      animationId: null,

      projectileInterval: null, // Para os power-ups
      projectileSpawnInterval: null, // NOVO: Para os projéteis dos inimigos/boss
      tempoInterval: null,
      keysPressed: {},

      tempo: 0,
      pontos: 0,
      pontosSalvos: 0, // Para guardar a pontuação ao final da fase
      nivel: 1, // Representa o nível atual da gameplay (pode ser usado para regras de dificuldade/desenho)
      velocidadeProjeteis: 3, // Esta será a velocidade base ou inicial
      // NOVO: Velocidade dos projéteis por fase
      velocidadeProjeteisPorFase: {
        1: 3, // Velocidade inicial para a Fase 1
        2: 4, // Velocidade para a Fase 2 (3 + 1)
        3: 5, // Velocidade para a Fase 3 (4 + 1)
        4: 6, // Velocidade para a Fase 4 (5 + 1)
        5: 7, // Velocidade para a Fase 5 (6 + 1)
      },
      vidas: 3,
      slowAtivo: false,
      slowTimeoutId: null,
      trocaFaseDelay: false,
      boss: null,
      bossDirecao: 1,
      player: {
        x: window.innerWidth / 2,
        y: window.innerHeight / 2,
        size: 120, // DOBRO DO TAMANHO ORIGINAL (60 * 2)
        hitboxRadius: 40 // DOBRO DO TAMANHO ORIGINAL (20 * 2) - Ajuste para colisão
      },

      projectileSize: 48, // AJUSTADO: um pouco menor (era 64)

      // Variáveis para a introdução inicial
      videoIndex: 0,
      historinhaJaVista: false,

      // Variáveis para as cutscenes de transição de fase
      currentCutsceneFaseIndex: 0,
      nextFaseAfterCutscene: 0,
      videosDeTransicao: [
        new URL('../assets/videos/video11.mp4', import.meta.url).href,
        new URL('../assets/videos/video12.mp4', import.meta.url).href,
        new URL('../assets/videos/video13.mp4', import.meta.url).href,
        new URL('../assets/videos/video14.mp4', import.meta.url).href,
      ],
      cutsceneFaseMap: {
        2: { startIndex: 0, endIndex: 3 }, // Transição para a Fase 2 usa video11, video12, video13, video14
      },
      cutscenesFaseJaVistas: {
        2: false,
        3: false,
        4: false,
        5: false,
      },

      videoMorte: new URL('../assets/videos/videomorte.mp4', import.meta.url).href,

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

      // Variáveis para a contagem regressiva
      countdownValue: 3,
      countdownIntervalId: null,

      // A fase atual do jogo - importante para manter a fase após a morte
      faseAtualDoJogo: 1,
    };
  },

  mounted() {
    Promise.all([carregarSprites(), carregarFundos()])
      .then(([sprites, fundos]) => {
        this.imagens = sprites;
        this.fundos = fundos;
        this.fundoAtual = fundos[0];
        console.log("Todas as imagens (sprites e fundos) carregadas com sucesso!");

        this.bgMusic = new Audio(new URL('../assets/audio/background_music.mp4', import.meta.url).href);
        this.bgMusic.loop = true;
        this.bgMusic.volume = this.volumeMusica / 100;

        this.bgMusic.play().catch(e => console.warn("Erro ao iniciar música de fundo (provavelmente por falta de interação do usuário):", e));

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
    iniciarOuContinuarJogo() {
      if (!this.historinhaJaVista) {
        this.estado = "historinha";
        this.videoIndex = 0;
        this.$nextTick(() => {
          if (this.$refs.videoPlayer) this.$refs.videoPlayer.play();
        });
      } else {
        // Se a historinha já foi vista, inicia diretamente a gameplay na fase atual
        this.iniciarGameplay(this.faseAtualDoJogo);
      }
    },

    avancarVideoHistorinha() {
      if (this.videoIndex < this.videosHistorinha.length - 1) {
        this.videoIndex++;
      } else {
        this.historinhaJaVista = true;
        this.iniciarGameplay(this.faseAtualDoJogo); // Passa a fase atual para iniciar gameplay
      }
    },

    pularVideoHistorinha() {
      this.historinhaJaVista = true;
      this.iniciarGameplay(this.faseAtualDoJogo); // Passa a fase atual para iniciar gameplay
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
            // Se não houver cutscene para esta fase ou já foi vista
            // Se for a transição para a Fase 2, sempre chama a contagem
            if (proximaFase === 2) {
                this.iniciarContagemRegressiva(proximaFase);
            } else {
                // Para outras fases, avança direto
                this.avancarParaProximaFase(proximaFase);
            }
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
            // Fim da cutscene para esta fase
            this.cutscenesFaseJaVistas[this.nextFaseAfterCutscene] = true;
            // Inicia a contagem regressiva para a próxima fase alvo (que será a Fase 2)
            this.iniciarContagemRegressiva(this.nextFaseAfterCutscene);
        }
    },

    pularCutsceneFase() {
      this.cutscenesFaseJaVistas[this.nextFaseAfterCutscene] = true;
      // Inicia a contagem regressiva para a próxima fase alvo ao pular
      this.iniciarContagemRegressiva(this.nextFaseAfterCutscene);
    },

    // Inicia a contagem regressiva
    iniciarContagemRegressiva(proximaFaseAlvo) {
      // Primeiro, muda o fundo para o da próxima fase alvo
      this.fundoAtual = this.fundos[proximaFaseAlvo - 1] || this.fundos[0];

      // Em seguida, inicia o estado de contagem
      this.estado = 'countdown';
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
            // Avança para a próxima fase após o "JÁ!"
            this.avancarParaProximaFase(proximaFaseAlvo);
          }, 800);
        } else {
          clearInterval(this.countdownIntervalId);
          this.countdownIntervalId = null;
        }
      }, 1000);
    },

    avancarParaProximaFase(novaFase) {
      // Salva a pontuação atual ANTES de atualizar o nível para a nova fase
      // Esta é a pontuação com a qual o jogador "entrou" na próxima fase
      this.pontosSalvos = this.pontos;

      this.nivel = novaFase; // Atualiza o nível lógico do jogo
      this.faseAtualDoJogo = novaFase; // Atualiza a fase atual do jogo (para renascer nela)
      this.zonasDeColisao = atualizarZonasDeColisao(this.nivel, zonasPorFase());
      // O fundo já foi atualizado antes da contagem regressiva

      this.pontos += 25; // Mantém o bônus de 25 pontos ao avançar de fase
      
      // ATUALIZA A VELOCIDADE DOS PROJÉTEIS PARA A NOVA FASE
      this.velocidadeProjeteis = this.velocidadeProjeteisPorFase[this.nivel];

      if (this.nivel === 5) {
        this.projectiles = [];
      }
      this.setupInimigos(); // Garante que inimigos são configurados para a nova fase
      this.estado = "jogando"; // Garante que o estado está "jogando"
      this.iniciarTimer(); // Reinicia o timer para a nova fase
      this.iniciarLoop(); // Garante que o loop de animação está ativo
    },

    iniciarGameplay(faseParaIniciar) {
      this.estado = "jogando";
      this.nivel = faseParaIniciar; // Define o nível do jogo como a fase passada
      this.faseAtualDoJogo = faseParaIniciar; // Atualiza a fase atual do jogo
      
      // Define a velocidade inicial dos projéteis para a fase que está começando
      this.velocidadeProjeteis = this.velocidadeProjeteisPorFase[faseParaIniciar];

      this.player.x = this.width / 2;
      this.player.y = this.height / 2;
      this.zonasDeColisao = atualizarZonasDeColisao(this.nivel, zonasPorFase()); // ATUALIZA ZONAS AQUI
      this.setupInimigos(); // CONFIGURA INIMIGOS AQUI
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
        this.limparTimers(); // Limpa todos os timers antes de reiniciar

        this.vidas = 3;
        // Lógica de pontos e nível:
        if (this.faseAtualDoJogo === 1) { // Se morreu na Fase 1
          this.pontos = 0; // Zera pontuação
          this.pontosSalvos = 0; // Zera pontuação salva
          this.nivel = 1; // Reinicia na Fase 1
        } else { // Se morreu a partir da Fase 2
          this.pontos = this.pontosSalvos; // Volta para a pontuação salva da fase anterior
          // this.nivel e this.faseAtualDoJogo já estão corretos (permanecem na fase em que morreu)
        }
        this.tempo = 0; // Opcional: mantém o reset de tempo

        // Garante que a velocidade dos projéteis e o fundo sejam os da fase atual do jogo
        this.velocidadeProjeteis = this.velocidadeProjeteisPorFase[this.faseAtualDoJogo];
        this.fundoAtual = this.fundos[this.faseAtualDoJogo - 1] || this.fundos[0];

        this.player.x = this.width / 2;
        this.player.y = this.height / 2;
        this.projectiles = [];
        this.powerUps = [];
        this.slowAtivo = false;

        // Reconfigura inimigos para a fase em que o jogador renasceu
        this.setupInimigos();
        this.iniciarTimer(); // Inicia o timer de tempo
        this.iniciarLoop(); // Inicia o loop de animação e os timers de spawn de power-ups/projéteis
      }, 500);
    },

    resetarEstadoDoJogo() {
      const historinhaJaVistaTemp = this.historinhaJaVista;
      const imagensTemp = this.imagens;
      const fundosTemp = this.fundos;
      const volumeMusicaTemp = this.volumeMusica;
      const bgMusicTemp = this.bgMusic;

      this.limparTimers();

      // Restaura o estado inicial de data, mas mantém alguns dados
      Object.assign(this.$data, this.$options.data.call(this));

      this.historinhaJaVista = historinhaJaVistaTemp;
      this.imagens = imagensTemp;
      this.fundos = fundosTemp;
      this.volumeMusica = volumeMusicaTemp;
      this.bgMusic = bgMusicTemp;

      // Resetar para Fase 1 e cutscenes vistas
      this.nivel = 1;
      this.faseAtualDoJogo = 1; // Garante que o jogo reinicia na Fase 1
      this.velocidadeProjeteis = this.velocidadeProjeteisPorFase[1]; // Resetar velocidade para a Fase 1
      this.pontos = 0; // Zera pontuação para o início de um novo jogo
      this.pontosSalvos = 0; // Garante que pontosSalvos seja zerado

      this.cutscenesFaseJaVistas = {
          2: false, 3: false, 4: false, 5: false,
      };

      if (this.bgMusic) {
        this.bgMusic.volume = this.volumeMusica / 100;
        if (this.bgMusic.paused) {
          this.bgMusic.play().catch(e => console.warn("Erro ao tentar tocar música após reset:", e));
        }
      }

      this.fundoAtual = this.fundos[this.nivel - 1] || this.fundos[0];
    },

    limparTimers() {
      if (this.tempoInterval) {
        clearInterval(this.tempoInterval);
        this.tempoInterval = null;
      }
      if (this.projectileInterval) { // Para os power-ups
        clearInterval(this.projectileInterval);
        this.projectileInterval = null;
      }
      if (this.projectileSpawnInterval) { // NOVO: Para os projéteis dos inimigos/boss
        clearInterval(this.projectileSpawnInterval);
        this.projectileSpawnInterval = null;
      }
      if (this.slowTimeoutId) {
        clearTimeout(this.slowTimeoutId);
        this.slowTimeoutId = null;
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
      const size = 80; // DOBRO DO TAMANHO ORIGINAL (40 * 2) - Ajuste dos inimigos
      const padding = 50;
      if (this.faseAtualDoJogo === 5) { // Usa faseAtualDoJogo para configurar inimigos/boss
        this.inimigos = [];
        this.boss = gerarBoss(this.width, this.height, 400); // DOBRO DO TAMANHO ORIGINAL (200 * 2) - Ajuste do boss
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
        // Se o tempo é múltiplo de 25 segundos, e estamos em uma transição de fase
        // Verifica se a fase atual é menor que 5 (última fase)
        if (this.tempo % 25 === 0 && this.nivel < 5 && !this.trocaFaseDelay) {
            this.trocaFaseDelay = true; // Ativa o flag para evitar chamadas múltiplas

            // Lógica para cutscene/contagem regressiva
            if (this.nivel === 1) {
                // Se for a transição da Fase 1 para a Fase 2, mostra a cutscene
                setTimeout(() => {
                    this.mostrarCutsceneFase(this.nivel + 1); // Chama para a próxima fase (Fase 2)
                    this.trocaFaseDelay = false;
                }, 3000); // Pequeno atraso antes de mudar de fase/exibir cutscene
            } else {
                // Para as transições das Fases 2, 3 e 4, vai direto para a contagem regressiva
                setTimeout(() => {
                    this.iniciarContagemRegressiva(this.nivel + 1); // Chama para a próxima fase (Fase 3, 4, 5)
                    this.trocaFaseDelay = false;
                }, 3000);
            }
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

      // Limpa e inicia o intervalo de spawn de power-ups
      clearInterval(this.projectileInterval);
      this.projectileInterval = setInterval(() => {
        if (this.estado !== "jogando" || this.inGameMenuOpen) return;

        const spawnChance = Math.random();
        if (spawnChance < 0.05) {
          const powerUp = {
            x: Math.random() * this.width,
            y: -50,
            r: 20, // DOBRO DO TAMANHO ORIGINAL (10 * 2)
            type: Math.random() < 0.5 ? "life" : "slow",
            rotation: 0,
          };
          this.powerUps.push(powerUp);
        }
      }, 300); // Frequência de spawn de power-ups

      // NOVO: Limpa e inicia o intervalo para os inimigos/boss atirarem
      clearInterval(this.projectileSpawnInterval);
      this.projectileSpawnInterval = setInterval(() => {
        if (this.estado !== "jogando" || this.inGameMenuOpen) return;

        if (this.faseAtualDoJogo < 5) { // Usa faseAtualDoJogo
          this.inimigos.forEach(inimigo => {
            atirarProjeteis({
              atirador: inimigo,
              player: this.player,
              slowAtivo: this.slowAtivo,
              velocidadeProjeteis: this.velocidadeProjeteis, // Usa a velocidade da fase
              imagens: this.imagens,
              projectiles: this.projectiles
            }, { velocidadeMultiplicador: 1, width: this.projectileSize, height: this.projectileSize });
          });
        } else if (this.boss) { // Usa faseAtualDoJogo
          atirarProjeteis({
            atirador: this.boss,
            player: this.player,
            slowAtivo: this.slowAtivo,
            velocidadeProjeteis: this.velocidadeProjeteis, // Usa a velocidade da fase
            imagens: this.imagens,
            projectiles: this.projectiles
          }, { velocidadeMultiplicador: 2, width: this.projectileSize, height: this.projectileSize });
        }
      }, 700); // Frequência dos inimigos/boss atirarem (ajuste este valor para a dificuldade desejada)

      // Inicia o loop de animação para desenhar e mover os elementos
      cancelAnimationFrame(this.animationId);
      this.animate(ctx);
    },

    animate(ctx) {
      if (this.estado !== "jogando" || this.inGameMenuOpen) {
        cancelAnimationFrame(this.animationId);
        return;
      }

      ctx.clearRect(0, 0, this.width, this.height);
      if (this.fundoAtual || this.imagens.fundoPadrao) {
        ctx.drawImage(this.fundoAtual || this.imagens.fundoPadrao, 0, 0, this.width, this.height);
      }

      const speed = 5;
      const nextPositions = {
        up: { x: this.player.x, y: this.player.y - speed },
        down: { x: this.player.x, y: this.player.y + speed },
        left: { x: this.player.x - speed, y: this.player.y },
        right: { x: this.player.x + speed, y: this.player.y },
      };
      // A colisão com as zonas de colisão é verificada aqui
      verificarTeclaPressionada(this.keysPressed, this.zonasDeColisao, nextPositions, this.player, speed);

      // Certifique-se de que o player permaneça dentro dos limites da tela
      const metadePlayer = this.player.size / 2;
      this.player.x = Math.max(metadePlayer, Math.min(this.width - metadePlayer, this.player.x));
      this.player.y = Math.max(metadePlayer, Math.min(this.height - metadePlayer, this.player.y));


      // Estilos para o texto do HUD
      ctx.fillStyle = "rgba(255, 255, 255, 0.9)"; // Cor branca semi-transparente
      ctx.font = "bold 24px 'Press Start 2P', cursive"; // Fonte estilo pixel, se disponível, ou fallback
      ctx.shadowColor = "rgba(0, 0, 0, 0.7)"; // Sombra para o texto
      ctx.shadowBlur = 5;
      ctx.shadowOffsetX = 2;
      ctx.shadowOffsetY = 2;

      ctx.fillText(`Tempo: ${this.tempo}s`, 20, 30);
      ctx.fillText(`Pontos: ${this.pontos}`, 20, 60);
      ctx.fillText(`Fase: ${this.faseAtualDoJogo}`, 20, 90); // Usa faseAtualDoJogo aqui
      ctx.fillText(`Vidas: ${this.vidas}`, 20, 120);

      // Resetar sombra para não afetar outros desenhos
      ctx.shadowColor = "transparent";
      ctx.shadowBlur = 0;
      ctx.shadowOffsetX = 0;
      ctx.shadowOffsetY = 0;

      if (this.imagens.player) {
        drawImage(ctx, this.imagens.player, this.player.x, this.player.y, this.player.size, this.player.size);
      }

      if (this.faseAtualDoJogo < 5) { // Usa faseAtualDoJogo para decidir se desenha inimigos ou boss
        this.processarInimigos(ctx); // APENAS DESENHA inimigos, NÃO MAIS SPAWNA PROJÉTEIS AQUI
      } else {
        this.processarBoss(ctx); // APENAS DESENHA boss, NÃO MAIS SPAWNA PROJÉTEIS AQUI
      }

      this.processarProjetiles(ctx); // Move e desenha projéteis existentes
      this.processarPowerUps(ctx);
      this.animationId = requestAnimationFrame(() => this.animate(ctx));
    },

    processarBoss(ctx) {
      this.moverBoss();
      if (!this.boss || !this.imagens.boss) return;

      drawImage(ctx, this.imagens.boss, this.boss.x, this.boss.y, this.boss.size, this.boss.size, 0);
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

        // Colisão do projétil com o player
        if (verificarColisaoDeProjetil(p.x, p.y, p.r, this.player.x, this.player.y, this.player.hitboxRadius)) {
          this.vidas--;
          if (this.vidas <= 0) {
            this.estado = "morte";
            this.limparTimers();
          }
          return false;
        }

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

        if (verificarColisaoDeProjetil(pu.x, pu.y, pu.r, this.player.x, this.player.y, this.player.hitboxRadius)) {
          this.coletarPowerUp(pu);
          return false;
        }

        const img = pu.type === "life" ? this.imagens.vida : this.imagens.slow;
        const size = 60; // DOBRO DO TAMANHO ORIGINAL (30 * 2)
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
      this.boss.y += this.bossDirecao * this.boss.velocidade;

      if (
        this.boss.y + this.boss.size / 2 > this.height - 50 ||
        this.boss.y - this.boss.size / 2 < 50
      ) {
        this.bossDirecao *= -1;
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
        this.limparTimers(); // Limpa todos os timers (inclusive o de projéteis)
        if (this.bgMusic) {
          this.bgMusic.pause();
        }
      } else {
        // Ao fechar o menu, reinicia os timers
        this.iniciarLoop(); // Este vai reiniciar o timer de projéteis e power-ups
        this.iniciarTimer(); // Este reinicia o timer de tempo
        if (this.bgMusic) {
          this.bgMusic.play().catch(e => console.warn("Erro ao tentar tocar música ao fechar menu in-game:", e));
        }
      }
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

/* Estilos gerais do container do jogo para ocupar a tela toda */
.game-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  font-family: 'Press Start 2P', cursive, Arial, sans-serif; /* Prioriza fonte pixel */
}

/* Estilos do Canvas */
canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: black;
  z-index: 0;
}

/* --- ESTILOS DO MENU PRINCIPAL (MODERNO, MENOS SATURADO) --- */
.main-menu {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background: linear-gradient(135deg, rgba(15, 0, 0, 0.95), rgba(40, 0, 0, 0.95), rgba(15, 0, 0, 0.95)); /* Degradê mais profundo */
  /* Adicionar um padrão sutil ou noise */
  background-image: url('data:image/svg+base64,PHN2ZyB2aWV3Qm94PSIwIDAgOCA4IiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciPjxwYXRoIGQ9Ik0wIDhoOHYwSDB6TTMgMGgxVjFoLTF6TTIgMmgxVjNoLTF6TTUgMmgxVjNoLTF6TTIgNWgxVjZoLTF6TTUgNWgxVjZoLTF6TTMgN2gxVjhoLTF6IiBmaWxsPSIjMmYwMDAwIiBmaWxsLW9wYWNpdHk9IjAuMSIvPjwvc3ZnPg==');
  background-size: 8px 8px; /* Tamanho do padrão */
  z-index: 10;
}

.menu-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 50px; /* Mais padding */
  background: linear-gradient(145deg, #1a0000, #3a0000); /* Degradê mais escuro e profundo */
  border-radius: 20px; /* Bordas mais arredondadas */
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.7), 0 0 50px rgba(255, 0, 0, 0.2); /* Sombra mais intensa */
  animation: fadeInScale 0.7s ease-out forwards; /* Animação um pouco mais lenta */
  border: 2px solid #5a0000; /* Borda sutil para dar profundidade */
}

.game-logo {
  width: 90%; /* Logo um pouco maior */
  max-width: 550px;
  margin-bottom: 40px; /* Más espacio debajo de la logo */
  filter: drop-shadow(0 0 15px rgba(255, 50, 50, 0.7)) brightness(1.1); /* Brillo más intenso */
  animation: logoPulse 2s infinite alternate ease-in-out; /* Animación de pulso en la logo */
}

@keyframes logoPulse {
  0% { transform: scale(1); filter: drop-shadow(0 0 15px rgba(255, 50, 50, 0.7)); }
  100% { transform: scale(1.03); filter: drop-shadow(0 0 25px rgba(255, 0, 0, 0.9)); }
}

.button-group {
  display: flex;
  flex-direction: column;
  gap: 20px; /* Más espacio entre los botones */
  width: 100%;
  max-width: 350px; /* Botones un poco más anchos */
}

.menu-button {
  padding: 18px 30px; /* Más padding */
  font-size: 1.3em; /* Fuente un poco más grande */
  font-weight: bold;
  color: #fff;
  background-color: #550000; /* Color de fondo más oscuro */
  border: 3px solid #ff5555; /* Borde más grueso y vibrante */
  border-radius: 10px; /* Bordes más redondeados */
  cursor: pointer;
  transition: all 0.3s cubic-bezier(.25,.8,.25,1); /* Transição suave e elegante */
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.5);
  text-transform: uppercase;
  letter-spacing: 2px; /* Más espaciado entre letras */
  text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.5); /* Sombra en el texto del botón */
  position: relative;
  overflow: hidden; /* Para el efecto de brillo */
}

.menu-button::before { /* Efecto de brillo al pasar el mouse */
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
  background-color: #ff3333; /* Fondo más vibrante al pasar el mouse */
  transform: translateY(-5px) scale(1.02); /* Pop más pronunciado */
  box-shadow: 0 10px 25px rgba(255, 51, 51, 0.6), 0 0 30px rgba(255, 0, 0, 0.4);
  border-color: #fff; /* Borde blanco al pasar el mouse */
}

.menu-button:active {
  transform: translateY(0) scale(0.98);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.4);
  background-color: #aa0000; /* Fondo más oscuro al hacer clic */
  border-color: #ff0000;
}

/* --- ESTILOS PARA HISTORINHA/CUTSCENES (Vídeos) --- */
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
  object-fit: contain; /* Garante que o vídeo seja exibido corretamente */
  background-color: black;
}

/* ESTILO PARA O BOTÃO "Pular história" (para a história inicial E cutscenes de fase) */
.skip-intro-button {
  position: absolute;
  top: 20px;
  left: 20px;
  padding: 10px 20px;
  background-color: rgba(0, 0, 0, 0.7); /* Fundo mais escuro */
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.4); /* Borda mais visível */
  border-radius: 5px;
  cursor: pointer;
  font-size: 0.9em;
  transition: background-color 0.3s ease, border-color 0.3s ease;
  z-index: 25;
}

.skip-intro-button:hover {
  background-color: rgba(255, 50, 50, 0.7); /* Cor vermelha no hover */
  border-color: #ffcccc;
}

/* --- ESTILOS DA TELA DE OPÇÕES E CRÉDITOS --- */
.options-screen, .credits-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: linear-gradient(135deg, rgba(15, 0, 0, 0.95), rgba(40, 0, 0, 0.95), rgba(15, 0, 0, 0.95));
  background-image: url('data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgOCA4IiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciPjxwYXRoIGQ9Ik0wIDhoOHY4SDB6TTMgMGgxVjFoLTF6TTIgMmgxVjNoLTF6TTUgMmgxVjNoLTF6TTIgNWgxVjZoLTF6TTUgNWgxVjZoLTF6TTMgN2gxVjhoLTF6IiBmaWxsPSIjMmYwMDAwIiBmaWxsLW9wYWNpdHk9IjAuMSIvPjwvc3ZnPg==');
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
    background: rgba(30, 0, 0, 0.85); /* Fundo mais escuro */
    padding: 40px;
    border-radius: 15px;
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.6), 0 0 30px rgba(255, 0, 0, 0.1);
    border: 2px solid #6a0000;
    width: 90%;
    max-width: 600px;
    text-align: center;
}

.screen-title {
  font-size: 3.5em; /* Títulos maiores */
  margin-bottom: 40px;
  color: #ff6666; /* Cor vermelha mais suave */
  text-shadow: 0 0 15px rgba(255, 100, 100, 0.8);
  letter-spacing: 3px;
  text-transform: uppercase;
}

.option-group {
  background: rgba(40, 0, 0, 0.7);
  padding: 25px 35px;
  border-radius: 10px;
  margin-bottom: 30px;
  width: 100%; /* Ocupa a largura total do content */
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.5);
  border: 1px solid #6a0000;
}

.option-group h3 {
  font-size: 1.6em; /* Título do grupo maior */
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
  min-width: 70px; /* Para alinhar */
  text-align: right;
}

.volume-control input[type="range"] {
  width: 60%;
  -webkit-appearance: none;
  height: 10px; /* Más grosso */
  background: #333;
  border-radius: 5px;
  outline: none;
  transition: background 0.2s ease;
}

.volume-control input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 24px; /* Mayor */
  height: 24px; /* Mayor */
  border-radius: 50%;
  background: #ff3333;
  cursor: pointer;
  border: 3px solid #fff; /* Borda más visible */
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
  font-size: 1.2em; /* Texto maior */
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
    white-space: pre-line; /* Mantém quebras de linha do JS */
}

.credits-text strong {
    color: #ffcc00;
}

.back-button {
  margin-top: 30px;
  width: auto; /* Deixa o botão se ajustar ao conteúdo */
  padding: 15px 40px; /* Mais padding para botões de voltar */
}

/* --- ESTILOS DO MENU IN-GAME --- */
.in-game-ui {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none; /* Permite cliques "através" da UI */
  z-index: 5;
}

.hamburger-button {
  position: absolute;
  top: 25px; /* Um pouco mais para baixo */
  right: 25px; /* Um pouco mais para a esquerda */
  background-color: rgba(0, 0, 0, 0.7); /* Fundo mais escuro */
  color: white;
  border: 3px solid #ff5555; /* Borda mais espessa */
  border-radius: 50%;
  width: 55px; /* Um pouco maior */
  height: 55px; /* Um pouco maior */
  font-size: 2em; /* Ícone maior */
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  z-index: 6;
  pointer-events: auto; /* Reabilita cliques no botão */
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
  width: 350px; /* Um pouco mais largo */
  height: 100vh;
  background: linear-gradient(90deg, rgba(15, 0, 0, 0.98), rgba(40, 0, 0, 0.98)); /* Degradê para o menu in-game */
  box-shadow: -8px 0 20px rgba(0, 0, 0, 0.7);
  padding: 50px 30px; /* Mais padding */
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  align-items: center;
  z-index: 10;
  pointer-events: auto; /* Reabilita cliques dentro do menu */
}

.in-game-menu-title {
  font-size: 2.5em; /* Título maior */
  color: #ffcc00;
  margin-bottom: 50px;
  text-shadow: 0 0 8px rgba(255, 204, 0, 0.6);
  letter-spacing: 2px;
}

.option-group-compact {
  width: 100%;
  margin-bottom: 40px; /* Mais espaço */
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
  width: 180px; /* Um pouco mais largo */
  height: 8px;
  background: #333;
  border-radius: 5px;
  outline: none;
  transition: background 0.2s ease;
  -webkit-appearance: none;
}

.volume-control-compact input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 22px; /* Um pouco maior */
  height: 22px; /* Um pouco maior */
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

.in-game-menu-buttons {
  display: flex;
  flex-direction: column;
  gap: 20px;
  width: 95%;
}

.in-game-button {
  width: 100%;
  padding: 15px 25px;
  font-size: 1.1em;
}

/* --- ESTILOS DA TELA DE CONTAGEM REGRESSIVA (3, 2, 1, JÁ!) --- */
.countdown-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.8); /* Fundo escuro mais opaco */
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 15;
}

.countdown-text {
  font-size: 12em; /* Tamanho ainda maior */
  color: #c0c0c0; /* Cor cinza claro */
  text-shadow: 0 0 15px rgba(192, 192, 192, 0.6), 0 0 30px rgba(192, 192, 192, 0.3); /* Sombra mais intensa */
  animation: countdownPulse 0.9s ease-out forwards; /* Animação um pouco mais rápida */
  font-family: 'Press Start 2P', cursive; /* Usa a fonte pixel */
  letter-spacing: -5px; /* Mais apertado */
}

/* Animação para o texto da contagem - mais suave */
@keyframes countdownPulse {
  0% {
    transform: scale(0.7);
    opacity: 0;
    filter: blur(10px);
  }
  50% {
    transform: scale(1.1); /* Um pequeno "pop" mais forte */
    opacity: 1;
    filter: blur(0);
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

/* --- ANIMAÇÕES DE TRANSIÇÃO (EXISTENTES) --- */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.7s ease; /* Transição um pouco mais lenta */
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.slide-up-enter-active,
.slide-up-leave-active {
  transition: transform 0.6s ease-out, opacity 0.6s ease;
}

.slide-up-enter-from {
  transform: translateY(100vh);
  opacity: 0;
}
.slide-up-leave-to {
    transform: translateY(100vh);
    opacity: 0;
}

.slide-down-enter-active,
.slide-down-leave-active {
  transition: transform 0.6s ease-out, opacity 0.6s ease;
}

.slide-down-enter-from {
  transform: translateY(-100vh);
  opacity: 0;
}
.slide-down-leave-to {
    transform: translateY(-100vh);
    opacity: 0;
}


.slide-right-enter-active,
.slide-right-leave-active {
  transition: transform 0.4s ease-out;
}

.slide-right-enter-from,
.slide-right-leave-to {
  transform: translateX(100%);
}
</style>