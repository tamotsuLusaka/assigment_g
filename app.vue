<script setup>

let errorMessage = ref("")

const url = "https://opendata.resas-portal.go.jp/api/v1/"
const key = "VB5sRHXcmpioPHgR1hyKHHVkhD3jn796dLooBiee"
const headers = new Headers({
  "x-api-key": key,
  "Content-Type": "application/json;charset=UTF-8"
})


let prefectureLists = ref([])
const { data:res } = await useFetch(
  url + "prefectures",
  {
    method: "GET",
    headers: headers
  }
)

for(let i = 0; i < res._rawValue.result.length; i++){
  res._rawValue.result[i].isSelected = false
  prefectureLists.value.push(res._rawValue.result[i])
}



let populationData = ref([])
const getPopulation = async (code)=>{
  const { data:res } = await useFetch(
    url + `population/composition/perYear?prefCode=${code}`,
    {
      method: "GET",
      headers: headers
    }
  )
  return res._rawValue.result.data[0].data
}

const constructData = async ()=>{
  const targetPrefectureLists =  prefectureLists.value.filter(prefecture=> prefecture.isSelected === true)
  populationData.value = await Promise.all(targetPrefectureLists.map(target => getPopulation(target.prefCode)))
}


</script>


<template>
  <header></header>
  <main>
    <p>{{ errorMessage }}</p>
    <div class="prefecture">
      <div v-for="prefecture in prefectureLists" :key="prefecture.prefCode" class="prefecture-list">
        <div class="prefecture-list-box">
          <input type="checkbox" v-model="prefecture.isSelected" :id="prefecture.prefCode">
          <label :for="prefecture.prefCode">{{ prefecture.prefName }}</label>
        </div>
        
      </div>
      <button @click="constructData()">ここ</button>
      {{ populationData }}
    </div>

  </main>
</template>

<style scoped>


@media screen and (max-width: 599px){

}
</style>