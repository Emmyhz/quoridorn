<template>
  <div id="app" @wheel.passive="onWheel" @contextmenu.prevent>
    <!--
    <img src="./assets/logo.png">
    -->
    <GameTable ref="gameTable"/>

    <Deck></Deck>
  </div>
</template>

<script>
import { mapActions, mapState } from 'vuex'
import GameTable from './components/map/GameTable'
import Deck from './components/map/card/Deck'

export default {
  name: 'App',
  components: {
    GameTable,
    Deck
  },
  mounted () {
    this.onSettingMount()
    this.onMount()

    let count = 0
    document.onkeydown = event => {
      if ((event.ctrlKey || event.metaKey) && (event.keyCode === 83 || event.which === 83)) {
        if (count++ === 0) {
          this.exportStart()
        }
        return false
      }
    }
    document.onkeyup = event => {
      if ((event.ctrlKey || event.metaKey) && (event.keyCode === 83 || event.which === 83)) {
        count = 0
        return false
      }
    }
  },
  watch: {
    // background (background) {
    //   document.style['background-color'] = background
    // },
    backgroundColor: {
      handler: backgroundColor => {
        document.body.style.backgroundColor = backgroundColor
      },
      immediate: true
    }
  },
  computed: mapState({
    backgroundColor: state => {
      return state.public.map.background
    }
  }),
  methods: {
    ...mapActions([
      'onSettingMount',
      'onMount',
      'exportStart'
    ]),
    onWheel (e) {
      this.$refs['gameTable'].onWheel(e.wheelDelta)
    }
  }
}
</script>

<style>
  html, body {
    padding: 0;
    margin: 0;
    width: 100%;
    height: 100%;
    overflow: hidden;
  }

  /* サイズ調整（コンテンツを比率を変えずに内側にフィット） */
  img { object-fit: contain; }
  div.img { opacity: 0; background-size: contain; background: no-repeat center; }

  hr {
    margin: 3px 0;
  }

  .anime { opacity: 0; }

  #app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    font-size: 16px;
    width: 100%;
    height: 100%;
    perspective: 1000px;
    display: flex;
    justify-content: center;
    align-items: center;
    white-space: nowrap;
    -moz-user-select: none;
    -webkit-user-select: none;
    -ms-user-select: none;
  }

  .unSelectable {
    user-select: none;
    -ms-user-select: none;
    -moz-user-select: none;
    -webkit-user-select: none;
  }
  .flexCenter {
    display: flex;
    justify-content: center;
    align-items: center;
  }
</style>
