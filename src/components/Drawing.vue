<template>
  <div>
    <div ref="container">
      <canvas
        :width="width/2"
        :height="height/2"
        ref="canvas">
      </canvas>
    </div>
  </div>
</template>

<script>
import Konva from 'konva';

export default {
  name: 'Drawing',
  // propsは親の「KonvaCanvas.vue」から値を受け取るためのプロパティをかく。
  props: {
    mode: {
      type: String,
      default: ''
    },
    brushColor: {
      type: String,
      default: ''
    },
    backgroundImage: {
      type: String,
      default: ''
    }
  },
  data: () => ({
    width: window.innerWidth,
    height: window.innerHeight,
    stage: null,
    canvas: null,
    context: null,
    drawingLayer: null,
    drawingScope: null,
    lastPointerPosition: {},
    localPos: {
      x: 0,
      y: 0
    },
    pos: null,
    isPaint: false,
    imageObj: null,
    backGroundLayer: null,
    backGroundImageScope: null,
    undoStack: [],
    redoStack: []
  }),
  mounted: function () {
    //描画領域の親を指定して取得する
    var container = this.$refs.container;
    // https://konvajs.org/api/Konva.Stage.html#toc1__anchor
    // 複数レイヤに対応するためにこいつを使う。主に画像のため。
    this.stage = new Konva.Stage({
      container,
      width: this.width,
      height: this.height
    })
    // konva layer
    this.drawingLayer = new Konva.Layer()
    // 描画用のレイヤーを上に重ねる
    this.stage.add(this.drawingLayer)
    //キャンバス要素を取得
    this.canvas = this.$refs.canvas
    //描画範囲を規定する。x, y は画面によって要変更。imageの上に描画を重ねるイメージ
    this.drawingScope = new Konva.Image({
      image: this.canvas,
      x: this.width / 4,
      y: 5,
      stroke: 'black'
    })
    //描画エリアを重ねる
    this.drawingLayer.add(this.drawingScope)
    this.stage.draw()
    // 2dグラフィックの画像描画指定
    this.context = this.canvas.getContext('2d')
    this.context.strokeStyle = this.brushColor
    //線の形状指定
    this.context.lineJoin = 'round'
    this.context.lineWidth = 5

    // イベント追加
    //タップには(electron)ならpointerdown, pointerupがある、pointer系はsafari非対応っぽい
    //pcでもpointer_xxx系はつかえるので統一でもいいかも、タブレット系ならontouchmoveとかになる
    this.drawingScope.on('mousedown', this.mousedown)
    this.stage.addEventListener('mouseup', this.mouseup)
    // electronならpointermove
    this.stage.addEventListener('mousemove', this.mousemove)
    this.drawingScope.on('touchstart', this.mousedown)
    this.stage.addEventListener('touchend', this.mouseup)
    this.stage.addEventListener('touchmove', this.mousemove)

    //実際のタイミングだとマウント時じゃないかも
    this.imageObj = new Image()
    this.imageObj.addEventListener('load', this.imageOnload)
    this.imageObj.src = this.backgroundImage
  },
  methods: {
    mousedown: function () {
      this.isPaint = true

      // ポインターの位置を取得
      this.lastPointerPosition = this.stage.getPointerPosition()
      // undo配列にこの動作を記憶させる、が、不完全なのでコメントアウト
      // this.undoStack.push(this.context.getImageData(0,0,this.canvas.width, this.canvas.height))
    },
    mouseup: function () {
      //ペイントの終了state
      this.isPaint = false
    },
    mousemove: function () {
      if (!this.isPaint) {
        return;
      }
      // ペンモード時
      if (this.isTargetMode('brush') || this.isTargetMode('line')) {
        this.context.globalCompositeOperation = 'source-over';
      }
      // 消しゴムモード時
      if (this.isTargetMode('eraser')) {
        //ここに誰の？とかで分岐できる可能性がある
        this.context.globalCompositeOperation = 'destination-out';
      }
      // 今のパスをリセットする。
      this.context.beginPath()

      this.localPos.x = this.lastPointerPosition.x - this.drawingScope.x()
      this.localPos.y = this.lastPointerPosition.y - this.drawingScope.y()

      // 描画開始座標を指定する
      this.context.moveTo(this.localPos.x, this.localPos.y)

      this.pos = this.stage.getPointerPosition()
      this.localPos.x = this.pos.x - this.drawingScope.x()
      this.localPos.y = this.pos.y - this.drawingScope.y()

      // 描画開始座標から、lineToに指定された座標まで描画する
      // ライブラリというよりcanvas要素の描画系操作
      this.context.lineTo(this.localPos.x, this.localPos.y)
      this.context.closePath()
      this.context.stroke()
      this.drawingLayer.draw()

      this.lastPointerPosition = this.pos
    },
    onClearCanvas: function () {
      // 消すというよりも描画領域を切り取っている
      this.context.globalCompositeOperation = 'destination-out'
      this.context.fillRect(0, 0, this.width, this.height)
      this.drawingLayer.draw()

      // 消すタイミングで直前の状態をスタック
      //this.undoStack.push(this.context.getImageData(0, 0, this.canvas.width, this.canvas.height))

      this.$emit('on-reset')
    },
    // 現在のモードが指定されたモードと一致するかどうか
    isTargetMode: function (targetMode) {
      return this.mode === targetMode
    },
    
    // 画像がある場合、こいつをコールする。やってることはmountの時のキャンバス描画とほぼ一緒
    imageOnload: function() {
      if(!this.backgroundImage) return
      this.backGroundLayer = new Konva.Layer()

      this.backGroundImageScope = new Konva.Image({
        image: this.imageObj,
        x: this.width / 4,
        y: 5,
        width: this.canvas.width,
        height: this.canvas.height
      })

      this.backGroundLayer.add(this.backGroundImageScope)
      this.stage.add(this.backGroundLayer)

      this.backGroundLayer.moveToBottom()
    },
    // undo,redoは使わない
    undo: function(){
      if(this.undoStack.length == 0){
        return
      }

      this.redoStack.push(this.context.getImageData(0,0,this.canvas.width, this.canvas.height))

      this.context.putImageData(this.undoStack.pop(), 0, 0)
      this.drawingLayer.draw()
    },
    redo: function(){
      if(this.redoStack.length == 0){
        return
      }

      this.context.putImageData(this.redoStack.pop(), 0, 0)
      this.drawingLayer.draw()
    }
  },
  watch: {
    // ペンの色変更
    brushColor: function () {
      this.context.strokeStyle = this.brushColor
    }
  }
}
</script>
