<template>
  <div>
    <div style="display: flex; justify-content: center; margin-bottom: 20px;">
      <canvas ref="canvas" :width="maxWidth" :height="maxHeight" style="border: 1px solid #ccc;"></canvas>
    </div>
    <input type="range" min="0" :max="maxScroll" v-model="scrollPosition" @input="scrollImage" style="width: 50%;"/>
  </div>
</template>
<script>
import { fabric } from 'fabric';

export default {
  data() {
    return {
      canvas: null,
      img: null,

      // canvasのサイズ
      maxWidth: 800,
      maxHeight: 600,

      // 図
      chertUrl: [],
      chertImg: [],
      chertContext: null,

      // 始めの日付
      startYear: 0,
      startMonth: 0,
      startDay: 0,

      // 今の図のナンバー
      chertNum: 0,

      // 今の日付
      year: 0,
      month: 0,
      day: 0,

      // スクロール
      scrollPosition: 0,
      maxScroll: 0
    };
  },
  mounted() {

    // 一時データ
    this.chertUrl.push('C0801.png');
    this.chertUrl.push('C0802.png');
    this.chertUrl.push('C0803.png');
    this.chertUrl.push('C0804.png');
    this.chertUrl.push('C0805.png');


    this.startYear = 2023;
    this.startMonth = 5;
    this.startDay = 23;

    // 初期化
    this.year = this.startYear;
    this.month = this.startMonth;
    this.day = this.startDay;
    
    this.chertNum = 0;

    // 右クリックレスナー
    this.canvas = new fabric.Canvas(this.$refs.canvas, {fireRightClick: true});

    // 右クリックメニュー禁止
    this.canvas.wrapperEl.oncontextmenu = () => false;
    this.loadImage(this.chertNum);
  },
  methods: {

    /** 
     * 図のロード
    */
    loadImage(chertNum) {

      const images = require.context('@/assets/', false, /\.png$/);
      fabric.Image.fromURL(images('./' + this.chertUrl[chertNum]), (img) => {

        this.img = img;

        // 初期ズーム
        let scale = this.maxHeight / img.height;
        this.img.scale(scale);

        // セレクト不可
        this.img.selectable = false;

        // canvasに描き
        this.canvas.add(this.img);
        
        // スクロール最大値設定
        this.maxScroll = this.img.width * scale - this.canvas.width;
      });
    },

    scrollImage() {
        this.img.left = -this.scrollPosition;
        this.canvas.renderAll();
    }
  }
}
</script>
