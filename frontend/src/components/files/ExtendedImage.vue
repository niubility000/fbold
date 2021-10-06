<template>
  <div
    class="image-ex-container"
    ref="container"
    @touchstart="touchStart"
    @touchmove="touchMove"
    @touchend="touchEnd"
    @dblclick="zoomAuto"
    @mousedown="mousedownStart"
    @mousemove="mouseMove"    
    @mouseup="mouseUp"
    @wheel="wheelMove"
  >
    <img
      src=""
      class="image-ex-img image-ex-img-center"
      ref="imgex"
      @load="onLoad"
    />
  </div>
</template>
<script>
import throttle from "lodash.throttle";
import UTIF from "utif";

export default {
  props: {
    src: String,
    angle: Number,
    moveDisabledTime: {
      type: Number,
      default: () => 50,
    },
    classList: {
      type: Array,
      default: () => [],
    },
    zoomStep: {
      type: Number,
      default: () => 0.18,
    },
  },
  data() {
    return {
      scale: 1,
      lastX: null,
      lastY: null,
      inDrag: false,
      touches: 0,
      scope: "",
      lastTouchDistance: 0,
      navX: 0,
      moveDisabled: false,
      disabledTimer: null,
      imageLoaded: false,
      position: {
        center: { x: 0, y: 0 },
        relative: { x: 0, y: 0 },
      },
      maxScale: 8,
      minScale: 1,
      oScale: 0,
    };
  },
  mounted() {
    if (!this.decodeUTIF()) {
      this.$refs.imgex.src = this.src;
    }
    let container = this.$refs.container;
    this.classList.forEach((className) => container.classList.add(className));
    // set width and height if they are zero
    if (getComputedStyle(container).width === "0px") {
      container.style.width = "100%";
    }
    if (getComputedStyle(container).height === "0px") {
      container.style.height = "100%";
    }

    window.addEventListener("resize", this.onResize);
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.onResize);
    document.removeEventListener("mouseup", this.onMouseUp);
  },
  watch: {
    src: function () {
      if (!this.decodeUTIF()) {
        this.$refs.imgex.src = this.src;
      }


      this.setCenter();
    },
    angle: function () {
      
      this.setCenter()
    }
  },
  computed: {
    imgHeight() {
      let img = this.$refs.imgex
      // slow version
      // return img.clientHeight * Math.cos(this.angle * Math.PI) + img.clientWidth * Math.sin(this.angle * Math.PI)
      return !(this.angle % 180) ? img.clientHeight : img.clientWidth
    },
    imgWidth() {
      let img = this.$refs.imgex
      // slow version
      // return img.clientWidth * Math.cos(this.angle * Math.PI) + img.clientHeight * Math.sin(this.angle * Math.PI)
      return !(this.angle % 180) ? img.clientWidth : img.clientHeight
    }
  },
  methods: {
    // Modified from UTIF.replaceIMG
    decodeUTIF() {
      const sufs = ["tif", "tiff", "dng", "cr2", "nef"];
      let suff = document.location.pathname.split(".").pop().toLowerCase();
      if (sufs.indexOf(suff) == -1) return false;
      let xhr = new XMLHttpRequest();
      UTIF._xhrs.push(xhr);
      UTIF._imgs.push(this.$refs.imgex);
      xhr.open("GET", this.src);
      xhr.responseType = "arraybuffer";
      xhr.onload = UTIF._imgLoaded;
      xhr.send();
      return true;
    },
    
    fit() {
      const wScale = window.innerWidth / this.imgWidth
      const hScale = window.innerHeight / this.imgHeight
    
      this.oScale = Math.min(wScale, hScale)
      this.scale = this.oScale
      this.minScale = this.oScale
      this.maxScale = 8*this.oScale
      this.setZoom()
    },
    refit() {
      const target = this.fitScreenTarget()
      this.doMove(target[0], target[1])
    },
    fitScreenTarget() {
      if (this.scale <= this.minScale) {
        let style = this.$refs.imgex.style
        let posX = this.pxStringToNumber(style.left)
        let posY = this.pxStringToNumber(style.top)
        return [this.position.center.x - posX, this.position.center.y - posY]
      }
      else {
        let img = this.$refs.imgex
        const rect = img.getBoundingClientRect()
        const width = window.innerWidth
        const height = window.innerHeight
        let x = 0,y = 0
        // left out of viewport
        if (rect.left < 0 && rect.right < width) x = Math.min(-rect.left, width - rect.right)
        // right out of viewport
        else if (rect.left > 0 && rect.right > width) x = Math.max(-rect.left, width - rect.right)
        // top out of viewport
        if (rect.top < 0 && rect.bottom < height) y = Math.min(-rect.top, height - rect.bottom)
        // bottom out of viewport
        else if (rect.top > 0 && rect.bottom > height) y = Math.max(-rect.top, height - rect.bottom)
      return [x,y]
      }
    },
    
    onLoad() {
      let img = this.$refs.imgex;

      this.imageLoaded = true;

      if (img === undefined) {
        return;
      }

      img.classList.remove("image-ex-img-center");
      this.setCenter();
      img.classList.add("image-ex-img-ready");

      document.addEventListener("mouseup", this.onMouseUp);

      let realSize = img.naturalWidth;
      let displaySize = img.offsetWidth;

      // Image is in portrait orientation
      if (img.naturalHeight > img.naturalWidth) {
        realSize = img.naturalHeight;
        displaySize = img.offsetHeight;
      }

      // Scale needed to display the image on full size
      const fullScale = realSize / displaySize;

      // Full size plus additional zoom
       this.maxScale = fullScale *14;
    },
    onMouseUp() {
      this.inDrag = false;
      this.refit();
    },
    onResize: throttle(function () {
      if (this.imageLoaded) {
        this.setCenter();
        this.doMove(this.position.relative.x, this.position.relative.y);
      }
    }, 100),

    setCenter() {
      let container = this.$refs.container;
      let img = this.$refs.imgex;

      this.position.center.x = Math.floor(
        (container.clientWidth - img.clientWidth) / 2
      );
      this.position.center.y = Math.floor(
        (container.clientHeight - img.clientHeight) / 2
      );

      img.style.left = this.position.center.x + "px";
      img.style.top = this.position.center.y + "px";
      this.fit();
    },
    mousedownStart(event) {
      this.lastX = null;
      this.lastY = null;
      this.inDrag = true;
      event.preventDefault();
    },
    mouseMove(event) {
      if (!this.inDrag) return;
      this.navX = event.movementX;
      this.doMove(event.movementX, event.movementY);
      event.preventDefault();
    },
    mouseUp(event) {
      this.inDrag = false;
      event.preventDefault();
    },
    touchStart(event) {
      this.lastX = null;
      this.lastY = null;
      this.lastTouchDistance = null;
      if (event.targetTouches.length < 2) {
        setTimeout(() => {
          this.touches = 0;
        }, 300);
        this.touches++;
        if (this.touches > 1) {
          this.zoomAuto(event);
        }
      }
      event.preventDefault();
    },
    zoomAuto(event) {
      if(this.scale<2*this.oScale) {this.scope = "<2";}
      if(this.scale>=2*this.oScale && this.scale<4*this.oScale ) {this.scope = "2to4";}
      if(this.scale>=4*this.oScale) {this.scope = ">4";}
      
      switch (this.scope) {
        case "<2":
          this.scale = 2*this.oScale;
          break;
        case "2to4":
          this.scale = 4*this.oScale;
          break;
        default:
        case ">4":
          this.scale = this.oScale;
          this.setCenter();
          break;
      }
      this.setZoom();
      event.preventDefault();
    },
    touchMove(event) {
      event.preventDefault();
      if (this.lastX === null) {
        this.lastX = event.targetTouches[0].pageX;
        this.lastY = event.targetTouches[0].pageY;
        return;
      }
      let step = this.$refs.imgex.width / 5;
      if (event.targetTouches.length === 2) {
        this.moveDisabled = true;
        clearTimeout(this.disabledTimer);
        this.disabledTimer = setTimeout(
          () => (this.moveDisabled = false),
          this.moveDisabledTime
        );

        let p1 = event.targetTouches[0];
        let p2 = event.targetTouches[1];
        let touchDistance = Math.sqrt(
          Math.pow(p2.pageX - p1.pageX, 2) + Math.pow(p2.pageY - p1.pageY, 2)
        );
        if (!this.lastTouchDistance) {
          this.lastTouchDistance = touchDistance;
          return;
        }
	
	      if((touchDistance > this.lastTouchDistance) ||this.scale > this.oScale){
	
          this.scale += (touchDistance - this.lastTouchDistance) / step
          this.lastTouchDistance = touchDistance
          this.setZoom()
        } 
	
      } else if (event.targetTouches.length === 1) {
        if (this.moveDisabled) return;
        let x = event.targetTouches[0].pageX - this.lastX;
        let y = event.targetTouches[0].pageY - this.lastY;
        if (Math.abs(x) >= step && Math.abs(y) >= step) return;
        this.lastX = event.targetTouches[0].pageX;
        this.lastY = event.targetTouches[0].pageY;
        this.doMove(x, y);
      }
    },
    touchEnd(event) {
      if (event.targetTouches.length === 0) {
        this.refit()
      }
    },
    doMove(x, y) {
      if(this.scale!==this.oScale){
        let style = this.$refs.imgex.style;
        let posX = this.pxStringToNumber(style.left) + x;
        let posY = this.pxStringToNumber(style.top) + y;

        style.left = posX + "px";
        style.top = posY + "px";

        this.position.relative.x = Math.abs(this.position.center.x - posX);
        this.position.relative.y = Math.abs(this.position.center.y - posY);

        if (posX < this.position.center.x) {
          this.position.relative.x = this.position.relative.x * -1;
        }

        if (posY < this.position.center.y) {
          this.position.relative.y = this.position.relative.y * -1;
        }
      }
      
      
      if(this.scale===this.oScale){

         let style = this.$refs.imgex.style;
         let posX = this.pxStringToNumber(style.left) + x;

          style.left = posX + "px";

          this.position.relative.x = Math.abs(this.position.center.x - posX);

          if (posX < this.position.center.x) {
            this.position.relative.x = this.position.relative.x * -1;
          }
	
      }
    },
    wheelMove(event) {
      this.scale += -Math.sign(event.deltaY) * this.zoomStep;
      this.setZoom();
    },
    
    setZoom() {
      this.scale = this.scale < this.minScale ? this.minScale : this.scale;
      this.scale = this.scale > this.maxScale ? this.maxScale : this.scale;
     // this.$refs.imgex.style.transition ="0.8s";  对入场，拖动，缩放都有效了。希望只对入场有效，怎么做？在css里加会对入场和缩放有效，对拖动无效，好一些。
      this.$refs.imgex.style.transform = `scale(${this.scale}) rotate(${this.angle}deg)`;

      this.refit()
      event.preventDefault();
    },
    pxStringToNumber(style) {
      return +style.replace("px", "");
    },
  },
};
</script>
<style>
.image-ex-container {
  margin: auto;
  overflow: hidden;
  position: relative;
}

.image-ex-img {
  position: absolute;
}

.image-ex-img-center {
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  position: absolute;
  transition: none;
}

.image-ex-img-ready {
  left: 0;
  top: 0;
  transition: transform 0.1s ease;
}
</style>
