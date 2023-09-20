<script setup>
import { Chart as ChartJS, Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale } from 'chart.js'
import { Line } from 'vue-chartjs'
ChartJS.register(Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale)

useHead({
  style: [{ children: 'html,body { margin: 0;  padding: 0;}' }],
})



//カラーコードの生成
const prefectureLength = 47
const colorCodes = [];
const usedColors = new Set();

while (colorCodes.length < prefectureLength) {
  const r = Math.floor(Math.random() * 256);
  const g = Math.floor(Math.random() * 256);
  const b = Math.floor(Math.random() * 256);
  const colorCode = `rgb(${r},${g},${b})`;

  if (!usedColors.has(colorCode)) {
    colorCodes.push(colorCode);
    usedColors.add(colorCode);
  }
}


const url = "https://opendata.resas-portal.go.jp/api/v1/"
const key = import.meta.env.VITE_API_KEY
const headers = new Headers({
  "x-api-key": key,
  "Content-Type": "application/json;charset=UTF-8"
})


// 人口データの取得
const getPopulation = async (code) => {
  try{
    const {data} = await useFetch(
      url + `population/composition/perYear?prefCode=${code}`,
      {
        method: "GET",
        headers: headers
      }
    )
    return data._rawValue.result.data[0].data
  }catch(error){
    console.log(error.message)
    errorMessage.value = "人口データの取得に失敗しました"
  }
}


// 年代初期
let years = []
await getPopulation(1)
.then(res=>{
  years = res.map(prefecture=> prefecture.year)
})


// 都道府県の取得
const getPrefecture = async ()=>{
  try{
    const {data} = await useFetch(
      url + "prefectures",
      {
        method: "GET",
        headers: headers
      }
    )
    return data._rawValue.result
  }catch(error){
    console.log(error.message)
    errorMessage.value = "都道府県データの取得に失敗しました"
  }
}

let prefectureLists = ref([])
await getPrefecture()
.then(res=>{
  for(let i = 0; i < res.length; i++){
    res[i].isSelected = false
    prefectureLists.value.push(res[i])
  }
})



//グラフの設定
let chartData = reactive({
  labels: years,
  datasets: []
})
const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  animation:{
    easing:"easeOutCirc",
    duration: 500,
  },
  scaleLabel: {
    display: true,
    labelString: "文字列"
  },
  

}

let isShownChart = ref(false)
let isShownFirst = ref(false)
let datasets = ref([])
let previouslySelectedPrefectureLists = ref([])
watch(prefectureLists, (newVal, oldVal) => {
  isShownChart.value = false
  isShownFirst.value = true
  const newlySelectedPrefecture = newVal.find(prefecture => !previouslySelectedPrefectureLists.value.includes(prefecture) && prefecture.isSelected)
  previouslySelectedPrefectureLists.value = newVal.filter(prefecture => prefecture.isSelected)
  if (newlySelectedPrefecture) {
    console.log(newlySelectedPrefecture)
    getPopulation(newlySelectedPrefecture.prefCode)
    .then(res=>{
      const targetPrefecrureData = {
        label: newlySelectedPrefecture.prefName,
        data: res.map(obj=>obj.value),
        fill: false,
        borderColor: colorCodes[newlySelectedPrefecture.prefCode - 1],
        tension: 0.1
      }
      datasets.value.push(targetPrefecrureData)
      chartData.labels = years
      chartData.datasets = datasets.value
      removeUnselectedPrefectures()
      isShownChart.value = true
    })
  }else{
    removeUnselectedPrefectures()
    setTimeout(()=>{
      isShownChart.value = true
    },100)//処理を遅らせないとグラフが反映されないため
  }
}, { deep: true })

const removeUnselectedPrefectures = () => {
  datasets.value = datasets.value.filter(dataset => {
    const prefecture = prefectureLists.value.find(pref => pref.prefName === dataset.label)  
    return prefecture && prefecture.isSelected
  })
  chartData.datasets = datasets.value 
}
</script>


<template>
  <header><h1>都道府県別人口グラフ</h1></header>
  
  <main class="main">
    <p v-if="errorMessage">{{ errorMessage }}</p>
    <div class="graph">
      
      <div v-if="!isShownFirst" class="graph-descrption">
        <p>人口グラフを表示したい<br class="br">都道府県を選んでください</p>
      </div>
      <Line v-if="isShownChart" id="my-chart-id" :options="chartOptions" :data="chartData" class="chart"/>
      <Spinner v-else></Spinner>
    </div>

    <div>
      <div class="prefecture">
        <div v-for="prefecture in prefectureLists" :key="prefecture.prefCode" class="prefecture-box">
            <input type="checkbox" v-model="prefecture.isSelected" :id="prefecture.prefCode" class="prefecture-input">
            <label :for="prefecture.prefCode" class="prefecture-button" :class="{'prefecture-button-selected': prefecture.isSelected}">{{ prefecture.prefName }}</label>
        </div>
      </div>
    </div>
  </main>
</template>

<style scoped>
@import "@/assets/css/reset.css";

header{
  display: block;
  border-bottom: 1px solid #262626;
  height: 40px;
  width: 100%;
  color: #262626;
  text-align: center;
}
header h1{
  font-size: 17px;
  line-height: 40px;
}

main{
  width: calc(100% - 40px);
  max-width: 800px;
  margin: 0 auto;
  color: #262626;
}


.graph {
  width: 100%;
  height: 400px;
  margin:30px 0;
}
.graph-descrption{
  width: 100%;
  height: 100%;
  position: relative;
  text-align: center;
}
.graph-descrption p{
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: block;
  width: 100%;
  line-height: 1.8;
}
.br{
  display: none;
}
.chart{
  width: 100%;
}

.prefecture{
  display: flex;
  justify-content: flex-start;
  flex-wrap: wrap;
}
.prefecture-box{
  display: block;
  width: calc(16.6% - 10px);
  margin: 0 5px 10px;
}
.prefecture-input{
  display: none;
}
.prefecture-button{
  display: block;
  background-color: #f5f5f5;
  color: #262626;
  border: 2px solid #262626;
  text-align: center;
  height: 30px;
  line-height: 30px;
  border-radius: 20px;
  transition: 0.3s;
  font-size: 14px;
}
.prefecture-button-selected{
  background-color: #262626;
  color: #F5F5F5;
  border: 2px solid #262626;
}


@media screen and (max-width: 768px){

  main{
    width: calc(100% - 20px);
  }
  .graph {
    width: 100%;
    height: 300px;
    margin:10px 0 20px;
  }
  .br{
    display: block;
  }
  .prefecture-box{
    display: block;
    width: calc(25% - 10px);
    margin: 0 5px 10px;
  }
  .prefecture-button{
    padding: 3px 0;
    font-size: 13px;
    font-weight: 700;
    height: 23px;
    line-height: 23px;
    }
}


</style>