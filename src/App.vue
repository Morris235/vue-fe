<template>
  <div class="plus">
    <h1>덧셈 기능 만들기</h1>
    <label>num: 1</label><input type="text" v-model="num1">&nbsp;
    <label>num: 2</label><input type="text" v-model="num2">&nbsp;
    <button @click="sendPlus">더하기</button>
    <hr>
    <p>{{ num1 }} + {{ num2 }} = {{ sum }}</p>
  </div>
</template>

<script setup>
import { ref } from 'vue';

const num1 = ref(0);
const num2 = ref(0);
const sum = ref(0);

const sendPlus = async () => {
  // 6. 백엔드, 프론트 X (kubernetes의 ingress 추가 후)
  const response = await fetch('http://api.localtest.me/plus', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json;cahrset=utf-8;'
    },
    body: JSON.stringify({ num1: num1.value, num2: num2.value })
  });

  const data = await response.json();
  console.log('data:', data);
  sum.value = data.sum;
}


</script>

<style scoped>
.plus {
  text-align: center;
}
</style>