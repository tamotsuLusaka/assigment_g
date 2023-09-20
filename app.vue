<script setup>
import { Chart as ChartJS, Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale } from 'chart.js'
import { Line } from 'vue-chartjs'
ChartJS.register(Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale)


let errorMessage = ref("")


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
const key = "VB5sRHXcmpioPHgR1hyKHHVkhD3jn796dLooBiee"
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



//グラフの生成
let chartData = reactive({
  labels: years,
  datasets: [

  ]
})
const chartOptions = {
  responsive: true
}

let showChart = ref(false) // 初めはグラフを非表示に設定
let datasets = ref([])
let previouslySelectedPrefectureLists = ref([])
watch(prefectureLists, (newVal, oldVal) => {
  showChart.value = false
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
      showChart.value = true
    })
  }else{
    removeUnselectedPrefectures()
    setTimeout(()=>{
      showChart.value = true
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
  <header></header>
  <main class="main">
    <p v-if="errorMessage">{{ errorMessage }}</p>
    <div class="graph">
      <div v-if="showChart" class="graph-content">
      <Line id="my-chart-id" :options="chartOptions" :data="chartData"/>
      </div>
    </div>

    <div>
      <div class="prefecture">
        <div v-for="prefecture in prefectureLists" :key="prefecture.prefCode" class="prefecture-box">
            <input type="checkbox" v-model="prefecture.isSelected" :id="prefecture.prefCode" class="prefecture-input">
            <label :for="prefecture.prefCode" class="prefecture-button">{{ prefecture.prefName }}</label>
        </div>
      </div>
    </div>
  </main>
</template>

<style scoped>

main{
  width: calc(100% - 40px);
  max-width: 800px;
  margin: 0 auto;
}


.graph {
  width: 600px;
  height: 300px;
}

.prefecture{
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
}
.prefecture-box{
  display: block;
  width: 20%;

}
@media screen and (max-width: 599px){

}
</style>