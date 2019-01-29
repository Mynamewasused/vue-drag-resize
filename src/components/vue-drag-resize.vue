<template>
<div class="vdr" :style="style"
     :class="active || isActive ? 'active' : 'inactive'"
     @mousedown="bodyDown($event)"
     @touchstart="bodyDown($event)">
    <slot></slot>
    <div
      v-for="stick in sticks"
      v-bind:key="stick"
      class="vdr-stick"
      :class="['vdr-stick-' + stick, isResizable ? '' : 'not-resizable', rcutoffs]"
      @mousedown.stop.prevent="stickDown(stick, $event)"
      @touchstart.stop.prevent="stickDown(stick, $event)"
      :style="vdrStick(stick)">
    </div>
    <div
      v-for="rotate in rotates"
      v-bind:key="rotate"
      class="vdr-rotate"
      :class="['vdr-rotate-' + rotate, isRotatable ? '' : 'not-rotatable', rcutoffs]"
      @mousedown.stop.prevent="rotateDown(rotate, $event)"
      @touchstart.stop.prevent="rotateDown(rotate, $event)"
      :style="vdrRotate(rotate)">
    </div>
</div>
</template>

<script>
const stickSize = 8
const styleMapping = {
  y: {
    t: 'top',
    m: 'marginTop',
    b: 'bottom',
  },
  x: {
    l: 'left',
    m: 'marginLeft',
    r: 'right',
  }
}
import Vector from 'victor'

  export default {
    props: {
      isActive: {
        type: Boolean, default: false
      },
      isDraggable: {
        type: Boolean, default: true
      },
      isResizable: {
        type: Boolean, default: true
      },
      isRotatable: {
        type: Boolean, default: true
      },
      x: {
        type: Number,
        default: 0,
        validator: function (val) {
          return typeof val === 'number'
        }
      },
      y: {
        type: Number,
        default: 0,
        validator: function (val) {
          return typeof val === 'number'
        }
      },
      r: {
        type: Number,
        default: 0,
        validator: function (val) {
          return typeof val === 'number'
        }
      },
      w: {
        type: Number,
        default: 300,
        validator: function (val) {
          return val > 0
        }
      },
      h: {
        type: Number,
        default: 300,
        validator: function (val) {
          return val > 0
        }
      },
      sticks: {
        type: Array,
        default: function () {
          return ['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml']
        }
      },
      rotates: {
        type: Array,
        default: function () {
          return ['rtl', 'rtr', 'rbr', 'rbl']
        }
      },
    },
    mounted () {
      document.documentElement.addEventListener('mousemove', this.move)
      document.documentElement.addEventListener('mouseup', this.up)
      document.documentElement.addEventListener('mouseleave', this.up)
      document.documentElement.addEventListener('mousedown', this.deselect)
      document.documentElement.addEventListener('touchmove', this.move, true)
      document.documentElement.addEventListener('touchend touchcancel', this.up, true)
      document.documentElement.addEventListener('touchstart', this.up, true)
    },
    data: function () {
      return {
        active: this.isActive,
        width: this.w,
        height: this.h,
        rotation: this.r,
        center: {
          x: this.x,
          y: this.y
        },
        rcutoffs: 'r0'
      }
    },
    computed: {
      top: {
          get: function () {
              return this.center.y - this.height / 2
          },
          set: function (newValue) {
              this.center.y = newValue + this.height / 2
          }
      },
      left: {
          get: function () {
              return this.center.x - this.width / 2
          },
          set: function (newValue) {
              this.center.x = newValue + this.width / 2
          }
      },
      style () {
        return {
          top: this.top + 'px',
          left: this.left + 'px',
          width: this.width + 'px',
          height: this.height + 'px',
          color: 'white',
          transform: 'rotate(' + this.rotation + 'deg)'
        }
      },
      vdrStick() {
        return (stick) => {
          const stickStyle = {
            width: `${stickSize}px`,
            height: `${stickSize}px`,
          }
          stickStyle[styleMapping.y[stick[0]]] = `${stickSize * -1}px`
          stickStyle[styleMapping.x[stick[1]]] = `${stickSize * -1}px`
          return stickStyle
        }
      },
      vdrRotate() {
        return (rotate) => {
          const rotateStyle = {
            width: `${stickSize}px`,
            height: `${stickSize}px`,
          }
          rotateStyle[styleMapping.y[rotate[1]]] = `${-stickSize - 5}px`
          rotateStyle[styleMapping.x[rotate[2]]] = `${-stickSize - 5}px`
          return rotateStyle;
        }
      },
      rect() {
        return {
          left: Math.round(this.left),
          top: Math.round(this.top),
          width: Math.round(this.width),
          height: Math.round(this.height)
        }
      }
    },
    created: function () {
      this.bodyDrag = false
      this.rotateDrag = false
      this.isDraggable = true
      this.preventActiveBehavior = false
      this.startPos = {mouseX: 0, mouseY: 0, x: 0, y: 0, w: 0, h: 0}
    },
    beforeDestroy: function () {
      document.documentElement.removeEventListener('mousemove', this.move)
      document.documentElement.removeEventListener('mouseup', this.up)
      document.documentElement.removeEventListener('mouseleave', this.up)
      document.documentElement.removeEventListener('mousedown', this.deselect)
      document.documentElement.removeEventListener('touchmove', this.move, true)
      document.documentElement.removeEventListener('touchend touchcancel', this.up, true)
      document.documentElement.removeEventListener('touchstart', this.up, true)
    },
    methods: {
      deselect () {
        if (this.preventActiveBehavior) {
          return
        }
        this.active = false
      },
      move (ev) {
        if (!this.stickDrag && !this.bodyDrag && !this.rotateDrag) {
          return
        }
        ev.stopPropagation()
        if (this.bodyDrag) {
          this.bodyMove(ev)
        }
        if (this.rotateDrag) {
          this.rotateMove(ev)
        }
        if (this.stickDrag) {
          this.stickMove(ev)
        }
      },
      up (ev) {
        if (this.bodyDrag) {
          this.bodyUp(ev)
        }
        if (this.rotateDrag) {
          this.rotateUp(ev)
        }
        if (this.stickDrag) {
          this.stickUp(ev)
        }
      },
      bodyDown: function (ev) {
        if (!this.preventActiveBehavior) {
          this.active = true
        }
        if (ev.button && ev.button !== 0) {
          return
        }
        this.$emit('clicked', ev)
        if (!this.isDraggable || !this.active) {
          return
        }
        ev.stopPropagation()
        ev.preventDefault()
        this.startPos.x = this.center.x
        this.startPos.y = this.center.y
        this.startPos.width = this.width
        this.startPos.height = this.height
        this.startPos.mouseX = ev.pageX
        this.startPos.mouseY = ev.pageY
        this.bodyDrag = true
      },
      stickDown: function (stick, ev) {
        if (ev.button && ev.button !== 0) {
          return
        }
        if (!this.isResizable || !this.active) {
          return
        }
        ev.stopPropagation()
        ev.preventDefault()
        this.startPos.x = this.center.x
        this.startPos.y = this.center.y
        this.startPos.width = this.width
        this.startPos.height = this.height
        this.startPos.mouseX = ev.pageX
        this.startPos.mouseY = ev.pageY
        this.stickDrag = true
        this.currentStick = stick.split('')
      },
      rotateDown: function (stick, ev) {
        if (ev.button && ev.button !== 0) {
          return
        }
        if (!this.isRotatable || !this.active) {
          return
        }
        ev.stopPropagation()
        ev.preventDefault()
        this.currentStick = stick.substring(1)
        this.startPos.mouseX = ev.pageX
        this.startPos.mouseY = ev.pageY
        this.startPos.rotation = this.rotation
        this.startPos.rotationalCenter = this.getCenter(this.startPos.mouseX, this.startPos.mouseY, this.currentStick)
        this.rotateDrag = true
      },
      bodyMove (ev) {
        let delta = {
          x: this.startPos.mouseX - ev.pageX,
          y: this.startPos.mouseY - ev.pageY
        }
        this.center.x = this.startPos.x - delta.x
        this.center.y = this.startPos.y - delta.y
        this.$emit('dragging', this.rect)
      },
      rotateMove (ev) {
        const newPos = {
          x: ev.pageX,
          y: ev.pageY
        };
        const vStartVect = new Vector(this.startPos.mouseX - this.startPos.rotationalCenter.x, this.startPos.mouseY - this.startPos.rotationalCenter.y)
        const vEndVect = new Vector(newPos.x - this.startPos.rotationalCenter.x, newPos.y - this.startPos.rotationalCenter.y)
        this.rotation = this.startPos.rotation + vEndVect.angleDeg() - vStartVect.angleDeg()
        let tempAng = Math.trunc(((this.rotation + 22.5 ) % 360) / 45) * -45
        if (tempAng < 0) {
            tempAng = tempAng + 360
        }
        this.rcutoffs = 'r' + tempAng
      },
      stickMove (ev) {
        let delta = {
          x: this.startPos.mouseX - ev.pageX,
          y: this.startPos.mouseY - ev.pageY
        }
        let yPrime = (delta.y * Math.cos(-this.rotation * Math.PI / 180) + delta.x * Math.sin(-this.rotation * Math.PI / 180))
        let xPrime = (delta.x * Math.cos(-this.rotation * Math.PI / 180) - delta.y * Math.sin(-this.rotation * Math.PI / 180))
        let newX = this.startPos.x
        let newY = this.startPos.y
        switch (this.currentStick[0]) {
          case 'b':
            this.height = this.startPos.height - yPrime
            newX = newX + (yPrime / 2) * Math.sin(this.rotation * Math.PI / 180)
            newY = newY - yPrime / 2 * Math.cos(this.rotation * Math.PI / 180)
            break;
          case 't':
            this.height = this.startPos.height + yPrime
            newX = newX + (yPrime / 2) * Math.sin(this.rotation * Math.PI / 180)
            newY = newY - yPrime / 2 * Math.cos(this.rotation * Math.PI / 180)
            break;
        }
        switch (this.currentStick[1]) {
          case 'r':
            this.width = this.startPos.width - xPrime
            newX = newX - (xPrime / 2) * Math.cos(this.rotation * Math.PI / 180)
            newY = newY - (xPrime / 2) * Math.sin(this.rotation * Math.PI / 180)
            break;
          case 'l':
            this.width = this.startPos.width + xPrime
            newX = newX - (xPrime / 2) * Math.cos(this.rotation * Math.PI / 180)
            newY = newY - (xPrime / 2) * Math.sin(this.rotation * Math.PI / 180)
            break;
        }
        this.center.x = newX
        this.center.y = newY
      },
      bodyUp () {
        this.bodyDrag = false
        this.stickStartPos = {mouseX: 0, mouseY: 0, x: 0, y: 0, w: 0, h: 0}
        this.$emit('dragging', this.rect);
        this.$emit('dragstop', this.rect);
      },
      rotateUp () {
        this.rotateDrag = false
        this.stickStartPos = {mouseX: 0, mouseY: 0, x: 0, y: 0, w: 0, h: 0}
      },
      stickUp () {
        this.stickDrag = false
        this.stickStartPos = {mouseX: 0, mouseY: 0, x: 0, y: 0, w: 0, h: 0}
      },
      getCenter(x, y, stick) {
          let vTemp = new Vector(x, y)
          const tempx = this.width / 2
          const tempy = this.height / 2
          let dist = Math.hypot(tempx, tempy)
          let additionalAngle = 0
          switch(stick) {
            case 'tl':
              additionalAngle = Math.atan(tempy / tempx) * 180 / Math.PI
              break;
            case 'tm':
              additionalAngle = 90
              break;
            case 'tr':
              additionalAngle = 180 - Math.atan((this.height / 2) / (this.width / 2)) * 180 / Math.PI
              break;
            case 'ml':
              additionalAngle = 0
              break;
            case 'mr':
              additionalAngle = 180
              break;
            case 'bl':
              additionalAngle = -Math.atan((this.width / 2) / (this.height / 2)) * 180 / Math.PI
              break;
            case 'bm':
              additionalAngle = -90
              break;
            case 'br':
              additionalAngle = Math.atan((this.width / 2) / (this.height / 2)) * 180 / Math.PI - 180
              break;
          }
            let vAdd = new Vector(dist, 0)
            vAdd.rotateByDeg(this.rotation + additionalAngle)
            vTemp.add(vAdd)
            return vTemp
      }
    },
    watch: {
      active (isActive) {
        if (isActive) {
          this.$emit('activated')
        } else {
          this.$emit('deactivated')
        }
      },
      isActive (val) {
        this.active = val
      },
      x () {
        this.center.x = this.x
      },
      y () {
        this.center.y = this.y
      },
      w () {
        this.width = this.w
      },
      h () {
        this.height = this.h
      },
      r () {
        this.rotation = this.r
        let tempAng = Math.trunc(((this.rotation + 22.5 ) % 360) / 45) * -45
        if (tempAng < 0) {
            tempAng = tempAng + 360
        }
        this.rcutoffs = 'r' + tempAng
      }
    }
  }
</script>
<style>
.vdr {
  position: absolute;
  box-sizing: border-box;
  transform: rotate(0deg);
}
.vdr.active:before{
  content: '';
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
  box-sizing: border-box;
  outline: 1px dashed #d6d6d6;
  cursor: move;
}

.vdr-stick {
  box-sizing: border-box;
  position: absolute;
  font-size: 1px;
  background: #ffffff;
  border: 1px solid #6c6c6c;
  box-shadow: 0 0 2px #bbb;
}

.vdr-rotate {
  box-sizing: border-box;
  position: absolute;
  font-size: 1px;
  background: #ffffff;
  border: 1px solid #6c6c6c;
  box-shadow: 0 0 2px #bbb;
  cursor: grab;
}

.inactive .vdr-stick {
  display: none;
}

.inactive .vdr-rotate {
  display: none;
}

.vdr-stick-tl, .vdr-stick-br {
  cursor: nwse-resize;
}
.vdr-stick-tm, .vdr-stick-bm {
  left: 50%;
  cursor: ns-resize;
}
.vdr-stick-tr, .vdr-stick-bl{
  cursor: nesw-resize;
}
.vdr-stick-ml, .vdr-stick-mr {
  top: 50%;
  cursor: ew-resize;
}
.vdr-stick-ml.r45, .vdr-stick-mr.r45, .vdr-stick-ml.r225, .vdr-stick-mr.r225 {
    cursor: nesw-resize;
} 

.vdr-stick-ml.r90, .vdr-stick-mr.r90, .vdr-stick-ml.r270, .vdr-stick-mr.r270 {
    cursor: ns-resize;
}

.vdr-stick-ml.r135, .vdr-stick-mr.r135, .vdr-stick-ml.r315, .vdr-stick-mr.r315 {
    cursor: nwse-resize;
} 

.vdr-stick-ml.r180, .vdr-stick-mr.r180 {
    cursor: ew-resize;
} 

.vdr-stick.not-resizable{
  display: none;
}

.vdr-rotate.not-rotatable{
  display: none;
}
</style>
