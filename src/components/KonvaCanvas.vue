<template>
    <div>
        <div class="md-layout md-gutter" style="margin-left: 340px">
          <div class="md-layout-item">
            <md-field style="float: left">
              <label for="mode">モード</label>
              <md-select v-model="mode" name="mode" id="mode">
                <md-option value="brush">ペン</md-option>
                <md-option value="eraser">消しゴム</md-option>
                <md-option value="line">直線</md-option>
              </md-select>
            </md-field>

            <md-field style="float: left">
              <label for="brushColor">ペンの色</label>
              <md-input type="color" v-model="brushColor" />
            </md-field>

            <md-field style="float: left; margin-top: -8px">
              <md-button class="md-dense md-raised md-accent" @click="clearCanvas">
                  描画リセット
              </md-button>
            </md-field>

            <md-field style="float: left; margin-top: -8px">
              <md-button class="md-dense md-raised md-primary" @click="init">
                  設定のリセット
              </md-button>
            </md-field>
            <!-- 不完全。undo -> redo で次にundoになる対象が2つ前になってしまう。ちゃんとやるならundostack, redostackにマスタースタックみたいなのが要るかも -->
            <!-- <md-field style="float: left; margin-top: -8px">
              <md-button class="md-dense md-raised md-transparent" @click="redo">
                進む
              </md-button>
            </md-field>
            <md-field style="float: left; margin-top: -8px">
              <md-button class="md-dense md-raised md-transparent" @click="undo">
                戻る
              </md-button>
            </md-field> -->
          </div>
        </div>
      <Drawing :mode="this.mode" :brushColor="this.brushColor" :backgroundImage="bgImage" ref="drawing" />
    </div>
</template>

<script>
import Drawing from './Drawing.vue'
export default {
  name: 'KonvaCanvas',
  components: { Drawing },
  //このプロパティを変更して相手方に渡せば色や状態を変更できる
  data: () => ({
    mode: '',
    brushColor: '',
    defaultMode: 'brush',
    defaultBrushColor: '#000000',
    isBackground: false
  }),
  mounted: function() {
    this.init();
  },
  methods: {
    // モードとペンの色を初期状態にする
    init: function() {
      this.mode = this.defaultMode;
      this.brushColor = this.defaultBrushColor;
    },
    clearCanvas: function(){
      this.$refs.drawing.onClearCanvas()
    },
    //子のundo, redoを呼ぶ。戻る、進む
    undo: function(){
      this.$refs.drawing.undo()
    },
    redo: function(){
      this.$refs.drawing.redo()
    }

  },
  computed: {
    bgImage: function(){
      return require('../assets/bg-pastel.png')
    }
  }
}
</script>

<style lang="scss" scoped>
  .md-field {
    max-width: 110px;
  }
</style>
