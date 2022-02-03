<template>
  <q-page class="flex flex-center">
    <div class="panel">

      <div class="flex justify-between">
        <q-toolbar class="bg-blue text-white shadow-2 rounded-borders flex-center">
          <div class="text-h6">Ready?</div>
        </q-toolbar>
        <CardNumberAndTitle title="Sets" :number="sets" :currentNumber="countOfSets" />
        <CardNumberAndTitle title="Work" :number="work" :currentNumber="countOfWork" />
        <CardNumberAndTitle title="Rest" :number="rest" :currentNumber="countOfRest" />
      </div>

      <q-card class="my-card">
        <q-card-section class="flex justify-between">
          <div class="text-h3">Run {{run}} {{timer}}</div>
        </q-card-section>
      </q-card>

      <q-card-actions vertical>
        <q-btn flat class="bg-green" @click="countdown">Start</q-btn>
        <q-btn flat class="bg-red" @click="reset">Reset</q-btn>
      </q-card-actions>
    </div>

    <q-page-sticky position="bottom-right" :offset="[18, 18]">
      <q-btn
        color="secondary"
        @click="$q.fullscreen.toggle()"
        :icon="$q.fullscreen.isActive ? 'fullscreen_exit' : 'fullscreen'"
        :label="$q.fullscreen.isActive ? 'Exit Fullscreen' : 'Go Fullscreen'"
      />
    </q-page-sticky>
  </q-page>
</template>

<script>
import CardNumberAndTitle from 'components/CardNumberAndTitle.vue'

export default {
  components: { CardNumberAndTitle },
  name: 'Timer',
  data: () => {
    return {
      sets: 6,
      countOfSets: 0,
      timer:'0:05',
      work: 20,
      countOfWork: 0,
      rest: 10,
      countOfRest: 0,
      run: 'start',
      counter: {},
      workAudio: null,
      restAudio: null,
      finishedAudio: null
    }
  },
  mounted () {
      // monta todos os arquivos de áudio para começar o download antes
      this.restAudio =  new Audio(`sounds/rest.ogg`);
      this.workAudio = new Audio(`sounds/work.ogg`);
      this.finishedAudio = new Audio(`sounds/finished.ogg`);
  },
  methods: {
    countdown: function() {
      clearInterval(this.counter);
      this.startTimer(this.timer);
    },
    startTimer: function(timerCount) {
      let arrTimer = timerCount.split(':');
      let timeTo = new Date();
      const plusMinutes = parseInt(arrTimer[0]);
      const plusSeconds = parseInt(arrTimer[1]);
      timeTo.setMinutes(timeTo.getMinutes() + plusMinutes);
      timeTo.setSeconds(timeTo.getSeconds() + plusSeconds);

      this.counter = setInterval(() => {
        let now = new Date();
        let distance = timeTo.getTime() - now.getTime();
        const {minutes, seconds} = this.calcDistanceTime(distance);
        this.timer = `${minutes}:${seconds}`;
        this.setStateOverAll(minutes+seconds);
      }, 992);
    },
    calcDistanceTime: function(distance) {
      return {
        minutes: Math.floor((distance % (1000 * 60 * 60)) / (1000 * 60)),
        seconds: Math.floor((distance % (1000 * 60)) / 1000)
      }
    },
    setStateOverAll: function(lastTime) {
      if (lastTime <= 0) {
        this.timer = '0:00';
        clearInterval(this.counter);
        this.changeRun();
      }
    },
    async changeRun() {
      if (this.run == 'start') {
        if (this.countOfSets == this.sets) { // valida antes  se já acabou tudo e está tentando começar um novo set
          return this.playerByType('finished', false)
        }
        this.countOfRest = 0
        this.countOfWork = 0
        this.countOfSets++;
        return this.playerByType('work', true)
      }
      if (this.run == 'work') {
        this.countOfWork++;
        if (this.countOfWork == this.work) {
          if (this.countOfRest < this.rest) {
            return this.playerByType('rest', true)
          }
        }

        if (this.countOfRest < this.rest) { // tem descanso pra fazer?
          return this.playerByType('rest', true)
        }

        if (this.countOfWork < this.work) { // ainda tem exercicio?
          return this.playerByType('work', true)
        }
        this.run = 'start';
        return this.startTimer(`0:0`);
      }

      if (this.run == 'rest') {
        this.countOfRest++;
        if (this.countOfWork < this.work) { // ainda tem exercicio?
        return this.playerByType('work', true)
        } else if (this.countOfSets == this.sets) {
          if (this.countOfRest < this.rest) { // ainda tem descanso?
            return this.playerByType('rest', true)
          }
          return this.playerByType('finished', false) // se já fez os sets, works e rests.. então acaba
        } else {
          if (this.countOfRest < this.rest) { // tem descanso pendente antes de começar outro set?
            return this.playerByType('rest', true)
          }
          this.run = 'start'
          return this.startTimer(`0:0`);
        }
      }
    },
    /**
     * Executa um arquivo de audio baseado no tipo recebido
     * 
     * @param { String } type tipo do arquivo para tocar
     * @param { Boolean } isToPlayTimer condicional para exibir ou não o contador de tempo
     */
    async playerByType(type, isToPlayTimer) {
      if (['work', 'rest', 'finished'].indexOf(type) < 0) return
      switch (type) {
        case 'work':
            this.run = 'work';
            await this.playSomeMusic(this.workAudio)
          break;
        case 'rest':
            this.run = 'rest';
            await this.playSomeMusic(this.restAudio)
          break;
        case 'finished':
            this.run = 'finished';
            await this.playSomeMusic(this.finishedAudio)
          break;
        default:
          break;
      }
      if (isToPlayTimer) {
        this.startTimer(`0:${this[type]}`);
      }
    },
    /**
     * Executa um arquivo de áudio
     * @param { Object } audio contém o elemento com o arquivo a ser executado
     */
    playSomeMusic(audio) {
      return new Promise(res=>{
        audio.play()
        audio.onended = res
      })
    },
    /**
     * Pausa os arquivos de audio que estiverem sendo reproduzidos e retorna eles para o ínicio
     * se chamarmos .load() novamente faria o mesmo efeito, só que iria carregar o arquivo outra vez
     * para não ter que fazer isso, reseto ele na mão sem ter q processar o mesmo arquivo novamente
     */
    resetMedias () {
      const medias = ['work', 'rest', 'finished']
      medias.forEach(media => {
        if (this[`${media}Audio`] && typeof this[`${media}Audio`] === 'object' && this[`${media}Audio`].readyState) {
          this[`${media}Audio`].currentTime = 0
          this[`${media}Audio`].pause()
        }
      })
      
    },
    reset: function() {
      this.countOfRest = 0
      this.countOfSets = 0
      this.countOfWork = 0
      this.resetMedias()
      this.run = 'start';
      this.timer = '0:05';
      clearInterval(this.counter);
    }
  }
}
</script>

<style>
/*
.panel {
  min-width: 850px;
}
*/
</style>