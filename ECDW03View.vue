<template>
  <div>
    <div style="display: flex; justify-content: center; margin-bottom: 20px;">
      <canvas ref="canvas" :width="maxWidth" :height="maxHeight" style="border: 1px solid #ccc;"></canvas>
    </div>
    <button @click="rotateLeft">左转</button>
    <button @click="rotateRight">右转</button>
    <button @click="overTurn">翻转</button>
    <button @click="toggleZoomMode">{{modeText}}</button>
    <button @click="undo">复位</button>
  </div>
</template>
<script>
import { fabric } from 'fabric';

export default {
  data() {
    return {
      canvas: null,
      group: null,

      // 图像显示框设定
      maxWidth: 800,
      maxHeight: 600,

      // 关于缩放模式的设定
      modeText: "进入缩放模式",
      isZoomMode: false,

      // 缩放倍率上限设定
      maxZoom: 20,
      minZoom: 0.1,

      // 鼠标点击一次的缩放倍率设定
      clickEnlarge: 1.1,
      clickReduce: 0.9,

      // 范围低于此值的框选/拖拽操作被视为误差
      minDragging: 10,

      // 框选相关的设定
      startX: -1,
      startY: -1,

      // 拖拽相关的参数
      isDragging: false,
      draggingStartX: 0,
      draggingStartY: 0,
      draggingX: 0,
      draggingY: 0,
    };
  },
  mounted() {

    //开启右键监听
    this.canvas = new fabric.Canvas(this.$refs.canvas, {fireRightClick: true});

    // 禁用右键菜单
    this.canvas.wrapperEl.oncontextmenu = () => false;
    this.loadImage();
  },
  methods: {

    /** 
     * 载入图片与图形 
    */
    loadImage() {
      fabric.Image.fromURL(require('@/assets/test.png'), (img) => {
        let circle = new fabric.Circle({
          radius: 50, fill: 'red', left: 200, top: 200
        });

        // 创建一个组合图形（图片和圆形）
        this.group = new fabric.Group([img, circle], {
          left: 150,
          top: 100,
        });

        // 等比例缩放图片
        this.scaleImageToFit(this.group);

        // 不可选择
        this.group.set({
          selectable: false, 
        });

        // 将图片添加到 canvas 上
        this.canvas.add(this.group);
        this.canvas.centerObject(this.group);
        this.setupEvents();
      });
    },

    /** 
     * 初始化图像大小与位置
    */
    scaleImageToFit(img) {
      let imageAspectRatio = img.width / img.height;
      let canvasAspectRatio = this.maxWidth / this.maxHeight;
      let scale;

      if (imageAspectRatio > canvasAspectRatio) {
        // 图片宽度较大
        scale = this.maxWidth / img.width;
      } else {
        // 图片高度较大
        scale = this.maxHeight / img.height;
      }

      // 按比例缩放
      img.scale(scale);

    },

    /** 
     * 左旋转
    */
    rotateLeft() {
      this.group.rotate(this.group.angle - 90);
      this.canvas.renderAll();
    },

    /** 
     * 右旋转
    */
    rotateRight() {
      this.group.rotate(this.group.angle + 90);
      this.canvas.renderAll();
    },

    /** 
     * 翻转
    */
    overTurn() {
      this.group.rotate(this.group.angle + 180);
      this.canvas.renderAll();
    },

    /** 
     * 进入/退出缩放模式
    */
    toggleZoomMode() {
      if(this.isZoomMode) {
        this.modeText = "进入缩放模式";
      }
      else {
        this.modeText = "进入选择模式";
      }
      this.isZoomMode = !this.isZoomMode;
    },

    /** 
     * 缩放模式中的事件监听
    */
    setupEvents() {

      // 滚轮事件
      this.canvas.on('mouse:wheel', (opt) => {
        if (this.isZoomMode) {

          // 滚轮事件参数
          let delta = opt.e.deltaY;
          // 目前缩放尺寸
          let zoom = this.canvas.getZoom();

          // 计算缩放倍率
          zoom *= 0.999 ** delta;
          if (zoom > this.maxZoom) {
            zoom = this.maxZoom;
          }
          if (zoom < this.minZoom) {
            zoom = this.minZoom;
          }

          // 以鼠标为中心缩放
          this.canvas.zoomToPoint({ x: opt.e.offsetX, y: opt.e.offsetY }, zoom);

          // 禁止冒泡与默认事件
          opt.e.preventDefault();
          opt.e.stopPropagation();
        }
      });

      // 鼠标按下事件
      this.canvas.on('mouse:down', (opt) => {

        // 左键
        if (this.isZoomMode && opt.button === 1) {
          this.startX = opt.e.offsetX;
          this.startY = opt.e.offsetY;
        }
        // 右键
        else if (this.isZoomMode && opt.button === 3) {

          // 开启拖拽
          this.isDragging = true;
          this.draggingX = opt.e.offsetX
          this.draggingY = opt.e.offsetY;
          this.draggingStartX = opt.e.offsetX
          this.draggingStartY = opt.e.offsetY;
        }

        // 禁止冒泡与默认事件
        opt.e.preventDefault();
        opt.e.stopPropagation();
      });

      // 鼠标移动事件
      this.canvas.on('mouse:move', (opt) => {

        // 仅限拖拽开启时
        if (this.isZoomMode && this.isDragging) {

          // 以现在为基准计算视图移动值
          let vpt = this.canvas.viewportTransform;
          vpt[4] += opt.e.offsetX - this.draggingX;
          vpt[5] += opt.e.offsetY - this.draggingY;

          // 重绘
          this.canvas.requestRenderAll();

          // 为下一轮触发拖拽记录
          this.draggingX = opt.e.offsetX;
          this.draggingY = opt.e.offsetY;
        }

        // 禁止冒泡与默认事件
        opt.e.preventDefault();
        opt.e.stopPropagation();
      });

      // 鼠标抬起事件
      this.canvas.on('mouse:up', (opt) => {

        // 左键
        if (this.isZoomMode && opt.button === 1) {
          
          let selectedwidth = Math.abs(this.startX - opt.e.offsetX);
          let selectedhight = Math.abs(this.startY - opt.e.offsetY);

          // 是框选操作时
          if (this.startX > 0 && selectedwidth > this.minDragging && selectedhight > this.minDragging) {

            // 计算选区的中心点和缩放比例
            let zoomPoint = new fabric.Point(Math.min(this.startX, opt.e.offsetX) + selectedwidth / 2, Math.min(this.startY, opt.e.offsetY) + selectedhight / 2);
            let zoomScale = this.canvas.getZoom() + Math.min(this.canvas.getWidth() / selectedwidth, this.canvas.getHeight() / selectedhight);

            // 限制缩放倍率
            if (zoomScale > this.maxZoom) {
              zoomScale = this.maxZoom;
            }

            // 执行放大操作
            this.canvas.zoomToPoint(zoomPoint, zoomScale);

            // 重置用来判定是否是框选的变量
            this.startX = -1;
            this.startY = -1;
          }
          // 非框选操作时
          else {
            
            // 设置放大倍率
            let zoom = this.canvas.getZoom();
            zoom *= this.clickEnlarge;

            // 限制缩放倍率
            if (zoom > this.maxZoom) {
              zoom = this.maxZoom;
            }

            // 执行放大操作
            this.canvas.zoomToPoint({ x: opt.e.offsetX, y: opt.e.offsetY }, zoom);
          }
        }
        // 右键
        else if (this.isZoomMode && opt.button === 3) {

          // 结束拖拽
          this.isDragging = false;

          // 判断是否是拖拽操作（非拖拽操作时也会图像也会被拖拽，但是不执行缩小操作）
          if (this.isZoomMode && Math.abs(opt.e.offsetX - this.draggingStartX) <= this.minDragging && Math.abs(opt.e.offsetY - this.draggingStartY <= this.minDragging)) {

            // 设置缩小倍率
            let zoom = this.canvas.getZoom();
            zoom *= this.clickReduce;
            if (zoom < 0.1) {
              zoom = 0.1;
            }

            // 执行缩小
            this.canvas.zoomToPoint({ x: opt.e.offsetX, y: opt.e.offsetY }, zoom);
          }
        }

        // 禁止冒泡与默认事件
        opt.e.preventDefault();
        opt.e.stopPropagation();
      });
    },
    
    /** 
     * 复位
    */
    undo() {
      // 重置缩放比例为 1
      this.canvas.setZoom(1);

      // 重置视图变换
      this.canvas.viewportTransform = [1, 0, 0, 1, 0, 0];

      // 初始化缩放
      this.scaleImageToFit(this.group)

      // 初始化位置
      this.canvas.centerObject(this.group);

      // 重置旋转
      this.group.rotate(0);
      this.canvas.renderAll();
    },
  }
}
</script>
