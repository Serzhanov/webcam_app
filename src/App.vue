<template>
  <div id="app">
    <div>
      <VideoframeComp @dataReceived="receivedData" @drawMode="resetLabels"/>
    </div>
    <div class="labels">
    </div>
    <div v-if="drawMode">
      <button v-for="name in names" :key="name" @click="assignLabel(name)" class="label-btn">{{ name }}</button>
      <button @click="submitLabel" class="label-btn">Submit</button>
    </div>
    <div v-if="detectedLabel!==''">
      Detected: <span>{{ detectedLabel }}</span>
      <br>
      Confidence: <span>{{ detectedConf }}%</span>
    </div>
    <button v-if="!trainStage" @click="retrainModel">Retrain Model</button>
    <h2 v-if="trainStage">TRAINING ...</h2>
    <div v-if="!trainStage && Object.keys(trainStats).length > 0">
      <h3>Metrics:</h3>
      <ul>
        <li v-for="(value, key) in trainStats" :key="key">
          {{ key }}: {{ value }}
        </li>
      </ul>
    </div>
    <p><a href="https://dagshub.com/Serzhanov/ai_cloud_.mlflow" target="_blank">Check out our MLflow with the best UI/UX on DagsHub!</a></p>
    <div></div>
      <div v-if="!trainStage">
        <label>Batch: <input v-model="batch" type="number" /></label>
        <label>Epochs: <input v-model="epochs" type="number" /></label>
        <label>Optimizer:
          <select v-model="optimizer">
            <option value="SGD">SGD</option>
            <option value="Adam">Adam</option>
            <option value="AdamW">AdamW</option>
            <option value="RMSProp">RMSProp</option>
          </select>
        </label>
        <label>Learning Rate: <input v-model="lr0" type="number" step="0.0001" /></label>
      </div>
  </div>
</template>

<script>
//import { WebCamUI } from 'vue-camera-lib'
import VideoframeComp from './components/VideoframeComp.vue'
import axios from 'axios';
export default {
  components: {
    VideoframeComp,
    
  },//      <WebCamUI :fullscreenState="false" @photoTaken="photoTaken" :constraints="constraints" />
  data() {
    return {
      trainStats: {},
      trainStage:false,
      batch: 8,
      epochs: 1,
      optimizer: 'Adam',
      lr0: 0.001,
      drawMode:false,
      detectedConf: 0,
      detectedLabel: '',
      constraints: {
        video: { 
                width: 416 ,
                height: 416 ,
                facingMode: 'environment',
                frameRate: { ideal: 20 } // Specifies that the ideal frame rate is 30 fps
            }
      },
      names: ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z'],
      selectedLabel: null,
    };
  },
  
  methods: {
    submitLabel() {
      console.log('Submitting label:', this.selectedLabel);
      // Include logic here to handle the labeled image sending to the backend.
    },
    photoTaken(data) {
      console.log('image blob: ', data.blob);
      console.log('image data url', data.image_data_url);
      // Resize image or ensure it's handled in your backend.
      // Send data to backend here if needed.
    },
    resetLabels() {
      console.log('Resetting labels')
      this.drawMode=!this.drawMode
      if(this.drawMode){
        this.names=['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z']
      }
      else{
        this.names=[]
      } 
    },
    receivedData(data){
      this.drawMode=false
      console.log('Data received from child component: ', data);
      this.names=[]
      this.detectedConf = data.cls.conf*100;
      console.log(data)
      this.detectedLabel = data.cls.name;
      this.names.filter(name => name === data.cls.name);
      
    },
    normalizeKeys(data) {
      const newData = {};

      // Helper function to convert strings to camelCase
      function toCamelCase(str) {
        return str
          .toLowerCase()
          .replace(/[^a-zA-Z0-9]+(.)/g, (match, chr) => chr.toUpperCase());
      }

      // Iterate over each key in the object
      for (const key in data) {
        // Safely check if the original object has this property as its own (not inherited)
        if (Object.prototype.hasOwnProperty.call(data, key)) {
          // Normalize the key and assign the value to the new key in newData
          const normalizedKey = toCamelCase(key.trim());
          newData[normalizedKey] = data[key];
        }
      }

      return newData;
    },
    async retrainModel() {
    this.trainStage = true; // Assume this starts the loading or processing indicator
    const trainingParams = {
      batch: this.batch,
      epochs: this.epochs,
      optimizer: this.optimizer,
      lr0: this.lr0
    };
    
    console.log('axios post'); // Consider removing or replacing with a more descriptive log for production

    try {
      const response = await axios.post('http://localhost:5000/retrain', trainingParams);
      console.log('Server response:', response.data);
      if(response.data.status === 'success') {
        this.trainStats = response.data.metrics;
        console.log('Training successful');
        this.trainStats=this.normalizeKeys(response.data.last_run_metrics)
        
      } else {
        console.error('Training failed:', response.data.status);
      }
    } catch (error) {
      console.error('Error sending training parameters:', error);
    } finally {
      this.trainStage = false; // This will run regardless of the try/catch outcome
    }
},    

    assignLabel(label) {
      this.selectedLabel = label;
      // Include logic here to handle the labeled image sending to the backend.
      console.log(`Label ${label} assigned to the image.`);
    },
  }
}
</script>

<style>
#app {
  display: flex; /* Using flexbox for layout */
  flex-direction: column; /* Stack children vertically */
  align-items: center; /* Center-align items horizontally */
  justify-content: center; /* Center-align items vertically */
  min-height: 100vh; /* Minimum height to take full viewport height */
  text-align: center;
  font-family: 'Arial', sans-serif;
}

.video-container {
  width: 1280px;
  height: 800px;
  overflow: hidden;
  display: flex; /* Enable flexbox inside the video container */
  justify-content: center; /* Center the video horizontally */
  align-items: center; /* Center the video vertically */
  margin-bottom: 20px; /* Add some space below the video container */
}

.labels {
  display: flex; /* Align label buttons in a row */
  flex-wrap: wrap; /* Allow wrapping of buttons */
  justify-content: center; /* Center the buttons horizontally */
  gap: 10px; /* Add gap between buttons */
}

.label-btn {
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 5px;
}

.label-btn:hover {
  background-color: #45a049;
}

.selected-label {
  margin-top: 20px;
  font-size: 18px;
  color: #333;
}

</style>


