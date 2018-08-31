<template>
  <div id="interactive" class="viewport scanner" :style="`height: ${readerSize.height}px;`">
    <video :style="`width: 100%;`"></video>
    <canvas class="drawingBuffer"></canvas>
  </div>
</template>

<script>
  import Quagga from 'quagga';

  export default {
    name: "scanner",
    props: {
      onDetected: {
        type: Function,
      },
      onProcessed: {
        type: Function,
      },
      readerType: {
        type: [Array],
        default: () => { return [{ format: 'code_128_reader', config: {} }] },
      },
      readerSize: {
        type: Object,
        required: false,
        default: () => {
          return {
            width: 640,
            height: 480,
          }
        }
      },
      value: {
        type: Boolean,
        required: true,
      }
    },
    data: function () {
      return {
        quaggaState: {
          inputStream: {
            type: 'LiveStream',
            constraints: {
              width: {min: this.readerSize.width},
              height: {min: this.readerSize.height},
              facingMode: 'environment',
              aspectRatio: {min: 1, max: 2}
            }
          },
          locator: {
            patchSize: 'medium',
            halfSample: true
          },
          numOfWorkers: 2,
          frequency: 10,
          decoder: {
            readers: this.readerType
          },
          locate: true
        },
      }
    },
    async mounted () {
      await this.init();
      this.startProcessing();
    },
    methods: {
      _onProcessed: function (result) {
        let drawingCtx = Quagga.canvas.ctx.overlay,
          drawingCanvas = Quagga.canvas.dom.overlay;

        if (result) {
          if (result.boxes) {
            drawingCtx.clearRect(0, 0, parseInt(drawingCanvas.getAttribute("width")), parseInt(drawingCanvas.getAttribute("height")));
            result.boxes.filter(function (box) {
              return box !== result.box;
            }).forEach(function (box) {
              Quagga.ImageDebug.drawPath(box, {x: 0, y: 1}, drawingCtx, {color: "green", lineWidth: 2});
            });
          }
          if (result.box) {
            Quagga.ImageDebug.drawPath(result.box, {x: 0, y: 1}, drawingCtx, {color: "#00F", lineWidth: 2});
          }

          if (result.codeResult && result.codeResult.code) {
            Quagga.ImageDebug.drawPath(result.line, {x: 'x', y: 'y'}, drawingCtx, {color: 'red', lineWidth: 3});
          }
        }
      },
      async init() {
        return new Promise((resolve, reject) => {
          Quagga.init(this.quaggaState, (err) => {
            if (err) {
              return console.log(err);
            }
            resolve();
          });
          Quagga.onDetected(this._onDetected);
          Quagga.onProcessed(this.onProcessed ? this.onProcessed : this._onProcessed);
        });
      },
      startProcessing() {
        Quagga.start();
        this.$emit("input", true);
      },
      _onDetected: function (result) {
        console.log('detected: ', result);
        this.$emit("detected", result);
      },
    },
    watch: {
      value(newValue, oldValue) {
        if(!newValue) {
          Quagga.stop();
        } else {
          this.init().then(() => {
            this.startProcessing();
          });
        }
      }
    },
  }
</script>

<style scoped>
  .viewport {
    position: relative;
  }

  .viewport canvas, .viewport video {
    position: absolute;
    left: 0;
    top: 0;
  }
</style>
