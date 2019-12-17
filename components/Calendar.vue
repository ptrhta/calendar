<template>
  <div id="canvas-wrap" ref="wrap">
    <canvas ref="canvas"></canvas>
  </div>
</template>
<script>
export default {
  data: () => ({
    gMinX: null, // глобальный минимум по шкале X
    gMaxX: null, // глобальный максимум по шкале X
    minX: null, // минимум по шкале X
    maxX: null // максимум по шкале X
  }),

  mounted() {
    // TODO: Установить размеры для canvas элемента
    // Сохранить в this.ctx 2d context canvas'а, причем
    // не нужно его ставить внутри data, так как на контексте
    // не будет никаких observer'ов изменений
    // P.S.: canvas должен занимать 100% от его родителя wrap
  },

  methods: {
    // TODO: отрисовывать на canvas шкалу времени
    // с учётом координат по оси X
  }
};
</script>
<style>
/* немного стилей, чтобы видеть где должен быть canvas элемент */
body {
  background-color: lightgray;
}
#canvas-wrap {
  margin: 0 auto;
  width: 800px;
  height: 600px;
  background: #fff;
}
</style>