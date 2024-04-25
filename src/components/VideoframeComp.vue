<template>
  <div class="video-container">
    <h1>Video Analysis</h1>
    <div class="video-overlay-container">
      <video ref="videoElement" autoplay playsinline></video>
      <canvas ref="canvas" class="video-overlay" @mousedown="onMouseDown" 
              @mousemove="onMouseMove" @mouseup="onMouseUp"></canvas>
    </div>
    <div class="controls">
      <button @click="sendFrame">Analyze Frame</button>
      <button @click="toggleDrawMode">{{ drawMode ? 'Return to Analyze' : 'Switch to Draw Mode' }}</button> 
    </div>
  </div>
</template>



<script>
import { io } from 'socket.io-client';
export default {
  name: 'VideoFrameComp',
  emits: ["dataReceived"],
  data() {
    return {
      videoElement: null,
      canvas: null,
      detectedLabel: '',
      detectedConf: 0,
      drawMode: false,
      selectedLabel: '',
      socket: io('http://localhost:5000'),
      drawing: false,
      start: { x: 0, y: 0 },
      drawnBox: [0, 0, 0, 0], // [x, y, width, height]
    };
  },
  mounted() {
    this.videoElement = this.$refs.videoElement;
    this.canvas = this.$refs.canvas;
    this.adjustCanvasSize();
    navigator.mediaDevices.getUserMedia({ video: true })
      .then(stream => {
        this.videoElement.srcObject = stream;
      })
      .catch(error => {
        console.error('Error accessing the camera:', error);
      });

    const self=this
    this.socket.on('processed', data => {
      console.log('received',data)
      if (data.status==0){
        console.log('Error processing frame:', data.error);
        return;
      }
      console.log('Data received from server:', data.response);
      var arr=data.response

      arr.forEach(function(item, index) {
          console.log(item, index); 
           // item is the array element, index is its index in the array
          console.log(item,'item')
          console.log(item.cls,'item.cls')
          console.log(item.cls.name,'item.cls.name')
          console.log(item.cls.name)
          self.detectedLabel = item.cls.name
          self.detectedConf = (item.cls.conf * 100).toFixed(2);
          self.drawBox(item.cls.box);
          self.emitDataToChild(item);
        });
    });
  },
  methods: {
    emitDataToChild(data) {
      this.$emit('dataReceived', data);
    },
    sendFrame() {
      if (!this.videoElement) return;
      this.adjustCanvasSize()
      const canvasTemp = document.createElement('canvas');
      canvasTemp.width = this.videoElement.videoWidth;
      canvasTemp.height = this.videoElement.videoHeight;
      const ctx = canvasTemp.getContext('2d');
      ctx.drawImage(this.videoElement, 0, 0, canvasTemp.width, canvasTemp.height);
      const image = canvasTemp.toDataURL('image/jpeg', 0.7);
      this.socket.emit('frame_from_client', { image });
    },
    adjustCanvasSize() {
      // Set canvas size equal to video size for proper coordinate mapping
      this.canvas.width = this.videoElement.clientWidth;
      this.canvas.height = this.videoElement.clientHeight;
    },
    drawBox(box) {
      const ctx = this.canvas.getContext('2d');
      ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      ctx.strokeStyle = 'red';
      ctx.lineWidth = 4;
      ctx.strokeRect(box[0], box[1], box[2] + box[0], box[3] + box[1]);
      ctx.font = '18px Arial';
      ctx.fillStyle = 'red';
      ctx.fillText(`${this.detectedLabel} (${this.detectedConf}%)`, box[0], box[1] - 10);
    },
    toggleDrawMode() {
      this.drawMode = !this.drawMode;
      this.canvas.style.pointerEvents = this.drawMode ? 'auto' : 'none';
      if(this.drawMode){
        this.$emit('drawMode',true);
      }
      else{
        this.$emit('drawMode',false);
      }
    },
    onMouseDown(event) {
      if (!this.drawMode) return;
      this.drawing = true;
      this.start = this.getCanvasCoordinates(event);
    },
    onMouseMove(event) {
      if (!this.drawing || !this.drawMode) return;
      const coords = this.getCanvasCoordinates(event);
      this.drawnBox = [
        Math.min(this.start.x, coords.x),
        Math.min(this.start.y, coords.y),
        Math.abs(coords.x - this.start.x),
        Math.abs(coords.y - this.start.y)
      ];
      this.redrawCanvas();
    },
    onMouseUp() {
      this.drawing = false;
    },
    getCanvasCoordinates(event) {
      const rect = this.canvas.getBoundingClientRect();
      return {
        x: event.clientX - rect.left,
        y: event.clientY - rect.top
      };
    },
    redrawCanvas() {
      const ctx = this.canvas.getContext('2d');
      ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      ctx.strokeStyle = 'red';
      ctx.lineWidth = 4;
      ctx.strokeRect(...this.drawnBox);
    }
  }
};
</script>


<style>
.video-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  position: relative;
  box-shadow: 0 4px 8px rgba(0,0,0,0.1); /* Adds shadow for better layer separation */
  background: #fff; /* Ensures the video area stands out on any page background */
  border-radius: 8px; /* Adds rounded corners for a softer look */
}
.video-overlay-container {
  position: relative;
  width: 100%;
  padding-top: 56.25%;  /* Aspect ratio of 16:9 */
  overflow: hidden;  /* Prevents any overflow, which might affect positioning */
}

video, canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;    /* Ensures that both video and canvas cover the entire parent container */
}
canvas {
  pointer-events: none;
  z-index: 10; /* Ensures the canvas is above the video for drawing */
}
button {
  margin-top: 10px;
  padding: 8px 16px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}
button:hover {
  background-color: #0056b3;
}
h1 {
  text-align: center;
  margin-bottom: 10px;
  color: #333;
}
</style>



