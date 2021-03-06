<template>
  <div
    id="gameTable"
    dropzone="move"
    @dragover.prevent
    @drop.prevent="drop"
    @contextmenu.prevent
    :style="gameTableStyle">

    <div :style="gridPaperStyle">
    </div>

    <div id="mapBoardFrame"
      @mousedown.left.prevent="leftDown" @mouseup.left.prevent="leftUp"
      @mousedown.right.prevent="rightDown" @mouseup.right.prevent="rightUp"
      @touchstart.prevent="leftDown" @touchend.prevent="leftUp" @touchcancel.prevent="leftUp">
      <MapBoard/>
    </div>

    <MapMask v-for="key in pieceKeyList('mapMask')" type="mapMask" :objKey="key" :key="key"
      @leftDown="leftDown" @leftUp="leftUp" @rightDown="rightDown" @rightUp="rightUp" @drag="dragging"/>
    <Character v-for="key in pieceKeyList('character')" type="character" :objKey="key" :key="key"
      @leftDown="leftDown" @leftUp="leftUp" @rightDown="rightDown" @rightUp="rightUp" @drag="dragging"/>
    <Chit v-for="key in pieceKeyList('chit')" type="chit" :objKey="key" :key="key"
      @leftDown="leftDown" @leftUp="leftUp" @rightDown="rightDown" @rightUp="rightUp" @drag="dragging"/>

  </div>
</template>

<script>
import { mapState, mapActions, mapGetters } from 'vuex'
import AddressCalcMixin from '../AddressCalcMixin'
import MapBoard from './MapBoard'
import MapMask from './mapMask/MapMask'
import Character from './character/Character'
import Chit from './chit/Chit'

export default {
  name: 'gameTable',
  mixins: [AddressCalcMixin],
  components: {
    MapBoard,
    MapMask,
    Character,
    Chit
  },
  mounted () {
    document.addEventListener('mousemove', this.mouseMove)
    document.addEventListener('touchmove', this.touchMove)
  },
  methods: {
    ...mapActions([
      'addPieceInfo',
      'windowOpen',
      'setProperty',
      'windowClose',
      'importStart'
    ]),
    dragging () {
      console.qLog(`★★★★ dragging ★★★★`)
    },
    onWheel (delta) {
      const changeValue = 100
      const add = delta > 0 ? changeValue : -changeValue
      const wheel = this.wheel + add
      if (wheel < -2400 || wheel > 800) {
        return
      }
      this.setProperty({property: 'private.map.wheel', value: wheel, logOff: true})
    },
    getAngle (mouseOnTable, storeObj) {
      const rectObj = {
        top: storeObj.top + storeObj.move.dragging.y,
        left: storeObj.left + storeObj.move.dragging.x,
        width: storeObj.columns * this.gridSize,
        height: storeObj.rows * this.gridSize
      }
      const center = {
        x: rectObj.left + rectObj.width / 2,
        y: rectObj.top + rectObj.height / 2
      }
      // 中心座標を基準としたマウス座標
      const loc = {
        x: (mouseOnTable.x - center.x),
        y: (mouseOnTable.y - center.y)
      }
      // 中心点と指定された座標とを結ぶ線の角度を求める
      return Math.atan2(loc.y, loc.x) * 180 / Math.PI
    },
    leftDown () {
      // console.qLog(`  [methods] mousedown left on GameTable`)
      const obj = {
        move: {
          from: {
            x: this.mouseLocate.x,
            y: this.mouseLocate.y
          }
        },
        isDraggingLeft: true
      }
      this.setProperty({property: 'map', value: obj, logOff: true})
    },
    leftUp () {
      // console.qLog(`  [methods] mouseup left on GameTable`)
      if (this.rollObj.isRolling) {
        // マップ上のオブジェクトを回転中の場合
        const pieceObj = this.$store.state.public[this.rollObj.propName].list.filter(obj => obj.key === this.rollObj.key)[0]
        const storeIndex = this.$store.state.public[this.rollObj.propName].list.indexOf(pieceObj)
        this.setProperty({property: `map.rollObj.isRolling`, value: false, logOff: true})
        const planeAngle = this.arrangeAngle(pieceObj.angle.dragging + pieceObj.angle.total)
        const total = this.arrangeAngle(Math.round(planeAngle / 30) * 30)
        // console.qLog(`angle:${angle}, planeAngle:${planeAngle}, totalB:${this.angle.total}, totalA:${total}`)
        const obj = {
          total: total,
          dragging: 0
        }
        this.setProperty({property: `public.${this.rollObj.propName}.list.${storeIndex}.angle`, value: obj, logOff: true, isNotice: true})
      } else {
        // マップを動かしている場合
        const obj = {
          move: {
            dragging: {
              x: 0,
              y: 0
            },
            total: {
              x: this.move.total.x + this.move.dragging.x,
              y: this.move.total.y + this.move.dragging.y
            }
          },
          isDraggingLeft: false
        }
        this.setProperty({property: 'map', value: obj, logOff: true})
      }
    },
    rightDown () {
      console.qLog(`  [methods] GameTableイベント => event: mousedown, mouse: right`)
      const obj = {
        angle: {
          dragStart: this.calcCoordinate(this.mouseLocate.x, this.mouseLocate.y, this.currentAngle).angle
        },
        isMouseDownRight: true
      }
      this.setProperty({property: 'map', value: obj, logOff: true})
    },
    rightUp (event) {
      console.qLog(`  [methods] GameTableイベント => event: mouseup, mouse: right`)
      const isDraggingRight = this.isDraggingRight
      this.setProperty({property: 'map.isMouseDownRight', value: false, logOff: true})

      let isRoll = false
      if (isDraggingRight) {
        const nextAngle = this.arrangeAngle(this.angle.total + Math.round(this.anglevolatile.dragging / 15) * 15)
        if (this.angle.total !== nextAngle) {
          isRoll = true
        }
        const obj = {
          angle: {
            dragging: 0
          },
          isDraggingRight: false
        }
        this.setProperty({property: 'map', value: obj, logOff: true})
        this.setProperty({property: 'private.map.angle.total', value: nextAngle, logOff: true})
      }
      let pageX = event.pageX
      let pageY = event.pageY

      if (!this.isOverEvent) {
        if (!isRoll) {
          // 右ドラッグが解除されたのが子画面上でなく、調整後に回転していない場合のみ右コンテキストメニューを表示する
          const obj = {
            x: pageX,
            y: pageY
          }
          this.setProperty({property: `private.display.gameTableContext`, value: obj, logOff: true})
          this.windowOpen(`private.display.gameTableContext`)
          console.qLog(`  [methods] open context => gameTableContext`)
        }
      } else {
        this.setProperty({property: `map.isOverEvent`, value: false, logOff: true})
      }
    },
    mouseMove (event) {
      this.setMouseLocateOnPage(event.pageX, event.pageY)
    },
    touchMove (event) {
      this.setMouseLocateOnPage(event.changedTouches[0].pageX, event.changedTouches[0].pageY)
    },
    setMouseLocateOnPage (pageX, pageY) {
      const obj = {
        x: pageX,
        y: pageY
      }
      this.setProperty({property: 'mouse', value: obj, logOff: true})

      const canvasAddress = this.calcCanvasAddress(event.pageX, event.pageY, this.currentAngle)
      const mapObj = {
        mouse: {
          onScreen: {
            x: canvasAddress.locateOnScreen.x,
            y: canvasAddress.locateOnScreen.y
          },
          onTable: {
            x: canvasAddress.locateOnTable.x,
            y: canvasAddress.locateOnTable.y
          },
          onCanvas: {
            x: canvasAddress.locateOnCanvas.x,
            y: canvasAddress.locateOnCanvas.y
          }
        },
        grid: {
          c: canvasAddress.grid.column,
          r: canvasAddress.grid.row
        }
      }
      if (this.isMouseDownRight && !this.isDraggingRight) {
        mapObj.isDraggingRight = true
      }
      this.setProperty({property: 'map', value: mapObj, logOff: true})
    },
    drop (event) {
      // ドロップされた物の種類
      const kind = event.dataTransfer.getData('kind')

      const offsetX = event.dataTransfer.getData('offsetX')
      const offsetY = event.dataTransfer.getData('offsetY')

      const canvasAddress = this.calcCanvasAddress(event.pageX, event.pageY, this.currentAngle, offsetX, offsetY)
      const locateOnTable = canvasAddress.locateOnTable
      if (this.isFitGrid) {
        locateOnTable.x = (Math.ceil(locateOnTable.x / this.gridSize) - 1) * this.gridSize
        locateOnTable.y = (Math.ceil(locateOnTable.y / this.gridSize) - 1) * this.gridSize
      }

      console.qLog(`  [methods] drop on GameTable => type: ${kind}, address: (${canvasAddress.grid.column},${canvasAddress.grid.row})`)

      const pieceObj = {
        kind: kind,
        propName: kind,
        left: locateOnTable.x,
        top: locateOnTable.y,
        isNotice: true,
        owner: `player-${this.playerName}`,
        place: 'field'
      }

      // マップマスクの作成
      if (kind === 'mapMask') {
        const name = event.dataTransfer.getData('name')
        const color = event.dataTransfer.getData('color')
        const fontColor = event.dataTransfer.getData('fontColor')
        const columns = parseInt(event.dataTransfer.getData('columns'), 10)
        const rows = parseInt(event.dataTransfer.getData('rows'), 10)

        // 必須項目
        pieceObj.columns = columns
        pieceObj.rows = rows
        // 個別部
        pieceObj.name = name
        pieceObj.color = color
        pieceObj.fontColor = fontColor

        this.addPieceInfo(pieceObj)
        return
      }

      // キャラクターの作成
      if (kind === 'character') {
        const name = event.dataTransfer.getData('name')
        const size = event.dataTransfer.getData('size')
        const useImageList = event.dataTransfer.getData('useImageList')
        const isHide = event.dataTransfer.getData('isHide') === 'true'
        const url = event.dataTransfer.getData('urlStr')
        const text = event.dataTransfer.getData('description')
        const useImageIndex = parseInt(event.dataTransfer.getData('useImageIndex'), 10)
        const currentImageTag = event.dataTransfer.getData('currentImageTag')

        // 必須項目
        pieceObj.columns = size
        pieceObj.rows = size
        // 個別部
        pieceObj.name = name
        pieceObj.useImageList = useImageList
        pieceObj.isHide = isHide
        pieceObj.url = url
        pieceObj.text = text
        pieceObj.useImageIndex = useImageIndex
        pieceObj.currentImageTag = currentImageTag
        pieceObj.fontColorType = 0
        pieceObj.fontColor = ''

        if (this.$store.state.private.display.addCharacterWindow.isContinuous) {
          const splits = name.split('_')
          const continuousNum = parseInt(splits[splits.length - 1], 10)
          this.setProperty({property: 'private.display.addCharacterWindow.continuousNum', value: continuousNum + 1, logOff: true})
        } else {
          this.windowClose('private.display.addCharacterWindow')
          this.setProperty({property: 'private.display.addCharacterWindow.continuousNum', value: 1, logOff: true})
        }

        this.addPieceInfo(pieceObj)
        return
      }

      // チットの作成
      if (kind === 'chit') {
        const currentImageTag = event.dataTransfer.getData('currentImageTag')
        const imageKey = event.dataTransfer.getData('imageKey')
        const isReverse = event.dataTransfer.getData('isReverse')
        const columns = event.dataTransfer.getData('columns')
        const rows = event.dataTransfer.getData('rows')
        const description = event.dataTransfer.getData('description')

        // 必須項目
        pieceObj.columns = columns
        pieceObj.rows = rows
        // 個別部
        pieceObj.currentImageTag = currentImageTag
        pieceObj.imageKey = imageKey
        pieceObj.isReverse = isReverse
        pieceObj.description = description

        this.addPieceInfo(pieceObj)
        return
      }

      // ファイルがドロップされてる
      const files = event.dataTransfer.files

      // ファイルの種類に応じて振り分け
      const imageFiles = []
      const zipFiles = []
      for (const file of files) {
        console.log(file.type)
        if (file.type.indexOf('image/') === 0) {
          // 画像
          imageFiles.push(file)
        } else if (file.type.indexOf('zip') >= 0) {
          // zip
          zipFiles.push(file)
        }
      }

      // 画像ファイルの処理
      if (imageFiles.length > 0) {
        // どこに使う画像ファイルなのかを選んでもらう
        const thumbnailSize = { w: 96, h: 96 }
        const promiseList = []
        for (const file of imageFiles) {
          promiseList.push(this.createBase64DataSet(file, thumbnailSize))
        }
        Promise.all([...promiseList]).then(function (values) {
          values.forEach(function (valueObj, index) {
            valueObj.key = index
          })
          this.setProperty({property: 'private.display.dropImageWindow.imageDataList', value: values})
        }.bind(this))
        this.windowOpen('private.display.dropImageWindow')
      }

      // zipファイルの処理
      if (zipFiles.length > 0) {
        this.importStart(zipFiles)
      }
    },
    createBase64DataSet (imageFile, thumbnailSize) {
      return new Promise(function (resolve, reject) {
        const createPromise = function (isThumbnail) {
          // eslint-disable-next-line
          return new Promise(function (resolve2, reject2) {
            // 画像の読み込み処理
            try {
              const reader = new FileReader()
              reader.onload = function (event) {
                if (isThumbnail) {
                  // サムネイル画像作成の場合は小さくて決まったサイズの画像データに加工する（アニメGIFも最初の１コマの静止画になる）

                  const image = new Image()
                  image.onload = function () {
                    const useSize = {
                      w: image.width,
                      h: image.height
                    }

                    // 大きい場合は、比率を保ったまま縮小する
                    if (useSize.w > thumbnailSize.w || useSize.h > thumbnailSize.h) {
                      const scale = Math.min(thumbnailSize.w / useSize.w, thumbnailSize.h / useSize.h)
                      useSize.w = useSize.w * scale
                      useSize.h = useSize.h * scale
                    }

                    // 画像を描画してデータを取り出す（Base64変換の実装）
                    const canvas = document.createElement('canvas')
                    const ctx = canvas.getContext('2d')
                    canvas.width = thumbnailSize.w
                    canvas.height = thumbnailSize.h
                    const locate = {
                      x: (canvas.width - useSize.w) / 2,
                      y: (canvas.height - useSize.h) / 2
                    }
                    ctx.drawImage(image, locate.x, locate.y, useSize.w, useSize.h)

                    // 非同期で返却
                    resolve2(canvas.toDataURL())
                  }
                  image.src = event.target.result
                } else {
                  // サムネイル画像でない場合はプレーンな画像データからBase64データを取得する

                  // 非同期で返却
                  resolve2(reader.result)
                }
              }
              reader.readAsDataURL(imageFile)
            } catch (error) {
              reject(error)
            }
          })
        }
        Promise.all([createPromise(true), createPromise(false)]).then(function (values) {
          resolve({
            name: imageFile.name,
            thumbnail: values[0],
            image: values[1]
          })
        })
      })
    }
  },
  watch: {
    mouseLocate: {
      handler (mouseLocate) {
        if (this.isDraggingLeft) {
          const obj = {
            x: mouseLocate.x - this.move.from.x,
            y: mouseLocate.y - this.move.from.y
          }
          this.setProperty({property: 'map.move.dragging', value: obj, logOff: true})
        }
        if (this.isDraggingRight) {
          const angle = this.calcCoordinate(mouseLocate.x, mouseLocate.y, this.currentAngle).angle
          let angleDiff = this.arrangeAngle(angle - this.anglevolatile.dragStart)
          this.setProperty({property: 'map.angle.dragging', value: angleDiff, logOff: true})
        }
      },
      deep: true
    }
  },
  computed: mapState({
    ...mapGetters([
      'pieceKeyList',
      'isFitGrid',
      'parseColor',
      'getBackgroundImage'
    ]),
    playerName: state => state.private.self.playerName,
    rollObj: state => state.map.rollObj,
    isDraggingLeft: state => state.map.isDraggingLeft,
    isMouseDownRight: state => state.map.isMouseDownRight,
    isOverEvent: state => state.map.isOverEvent,
    isDraggingRight: state => state.map.isDraggingRight,
    move: state => state.map.move,
    angle: state => state.private.map.angle,
    anglevolatile: state => state.map.angle,
    currentAngle () { return this.arrangeAngle(this.angle.total + this.anglevolatile.dragging) },
    sizeW () { return (this.columns + this.marginGridSize * 2) * this.gridSize },
    sizeH () { return (this.rows + this.marginGridSize * 2) * this.gridSize },
    marginGridColor: state => state.public.map.margin.gridColor,
    marginMaskColor: state => state.public.map.margin.maskColor,
    marginMaskAlpha: state => state.public.map.margin.maskAlpha,
    isUseGridColor: state => state.public.map.margin.isUseGridColor,
    isUseImage: state => state.public.map.margin.isUseImage,
    gameTableStyle () {
      const translateZ = this.wheel
      const zoom = (1000 - this.wheel) / 1000
      const totalLeftX = (this.move.total.x + this.move.dragging.x) * zoom
      const totalLeftY = (this.move.total.y + this.move.dragging.y) * zoom
      let rotateZ = this.currentAngle
      const result = {
        width: this.sizeW + 'px',
        height: this.sizeH + 'px',
        'border-width': this.borderWidth + 'px',
        transform:
          'translateZ(' + translateZ + 'px) ' +
          'translateY(' + totalLeftY + 'px) ' +
          'translateX(' + totalLeftX + 'px) ' +
          'rotateY(0deg) ' +
          'rotateX(0deg) ' +
          'rotateZ(' + rotateZ + 'deg)'
      }
      if (this.isUseImage) {
        result['background-image'] = `url(${this.getBackgroundImage})`
      }
      return result
    },
    gridPaperStyle () {
      const maskColorObj = this.parseColor(this.marginMaskColor)
      maskColorObj.a = this.marginMaskAlpha
      const marginMaskColorStr = maskColorObj.getRGBA()
      const result = {
        width: this.sizeW + 'px',
        height: this.sizeH + 'px',
        'background-color': marginMaskColorStr
      }
      if (this.isUseGridColor) {
        const colorObj = this.parseColor(this.marginGridColor)
        colorObj.a = 0.3
        const marginGridColor3 = colorObj.getRGBA()
        colorObj.a = 0.1
        const marginGridColor1 = colorObj.getRGBA()
        result['background-image'] =
          `linear-gradient(0deg, transparent -2px,` +
              `${marginGridColor3} 2px, ${marginGridColor3} 3%, transparent 4%, transparent 20%,` +
              `${marginGridColor1} 21%, ${marginGridColor1} 22%, transparent 23%, transparent 40%,` +
              `${marginGridColor1} 41%, ${marginGridColor1} 42%, transparent 43%, transparent 60%,` +
              `${marginGridColor1} 61%, ${marginGridColor1} 62%, transparent 63%, transparent 80%,` +
              `${marginGridColor1} 81%, ${marginGridColor1} 82%, transparent 83%, transparent),` +
          `linear-gradient(270deg, transparent -2px,` +
              `${marginGridColor3} 2px, ${marginGridColor3} 3%, transparent 4%, transparent 20%,` +
              `${marginGridColor1} 21%, ${marginGridColor1} 22%, transparent 23%, transparent 40%,` +
              `${marginGridColor1} 41%, ${marginGridColor1} 42%, transparent 43%, transparent 60%,` +
              `${marginGridColor1} 61%, ${marginGridColor1} 62%, transparent 63%, transparent 80%,` +
              `${marginGridColor1} 81%, ${marginGridColor1} 82%, transparent 83%, transparent)`
      }
      return result
    }
  })
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#gameTable {
  position: fixed;
  display: block;
  margin: auto;
  text-align: center;
  vertical-align: middle;
  -khtml-user-drag: element;
  background-position: 0 0;
  background-size: 100% 100%;
  cursor: crosshair;
  /*
  box-sizing: border-box;
  */
  perspective: 1000px;
  border: ridge gray;
  overflow: hidden;
}
#gameTable:before{
  content: '';
  background: inherit;/*.bgImageで設定した背景画像を継承する*/
  -webkit-filter: blur(10px);
  -moz-filter: blur(10px);
  -o-filter: blur(10px);
  -ms-filter: blur(10px);
  filter: blur(10px);
  position: absolute;
  /*ブラー効果で画像の端がボヤけた分だけ位置を調整*/
  top: -10px;
  left: -10px;
  right: -10px;
  bottom: -10px;
  z-index: -1; /*重なり順序を一番下にしておく*/
}
#gameTable > div {
  background-position: 1px 1px;
  background-size: 48px 48px;
}
#mapBoardFrame {
  position: fixed;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  border: none;
  text-align: center;
  vertical-align: middle;
}
</style>
