<template>
  <div class="calendar-wrap">
    <h1>Календарь</h1>
    <div class="row">
      <div class="doers">
        <div class="doer" v-for="doer in doers" :key="doer.id">
          <img class="img" :src="doer.avatar" alt="" />
        </div>
      </div>
      <div id="canvas-wrap" ref="wrap">
        <canvas ref="canvas" />
      </div>
    </div>
    <div class="row options">
      <div class="scaleSelection">
        <span>Масштаб:</span>
        <select v-model="selected">
          <option
            v-for="(option, index) in scaleOptions"
            :key="'so-' + index"
            :value="option.value"
            v-text="option.text"
          ></option>
        </select>
      </div>
    </div>
    <div class="row options">
      <pre>{{ debug }}</pre>
    </div>
  </div>
</template>

<script>
import calendarData from '../assets/tasks.json'

/*const CALENDAR_SCALES = {
  DAY: 10,
  WEEK: 20,
  MONTH: 30,
  QUARTER: 40,
}*/

const TIME_SCALE = {
  DAY: 'день',
  WEEK: 'неделя',
  MONTH: 'месяц',
  QUARTER: 'квартал',
}

const UNITS_IN_PX = {
  DAY: 300,
  WEEK: 350,
  MONTH: 500,
  QUARTER: 1000,
}
const S_DAY = 60 * 60 * 24
// const MS_DAY = 1000 * S_DAY
const TIME_IN_UNITS = {
  DAY: S_DAY,
  WEEK: S_DAY * 7,
  MONTH: S_DAY * 30,
  QUARTER: S_DAY * 92,
}

const HEIGHT_OF_LABELS = 40 // Смещение сверху от края канваса до начала области с данными
const LABEL_PADDING = 8 // отступ от подписей в шапке
const DEFAULT_LINE_WIDTH = 0.3 // толщина линии по умолчанию
const DEFAULT_STROKE_COLOR = '#000000' // цвет рисования по умолчанию
const ROW_HEIGHT = 40 // высота строки на календареы

export default {
  data: () => ({
    gMinX: null, // глобальный минимум по шкале X
    gMaxX: null, // глобальный максимум по шкале X
    minX: null, // минимум по шкале X
    maxX: null, // максимум по шкале X
    userY: 110,
    horizontalLineY: 0,
    selected: 'DAY',
    allDates: [],
    datesForScale: [],
    widthUnit: 0,
    left: 0,
  }),

  computed: {
    scaleOptions() {
      return Object.keys(TIME_SCALE).map(key => ({
        text: TIME_SCALE[key],
        value: key,
      }))
    },

    doers() {
      return calendarData.doers
    },

    debug() {
      let minX = new Date()
      minX.setTime(this.minX * 1000)
      let maxX = new Date()
      maxX.setTime(this.maxX * 1000)
      return {
        minX,
        maxX,
        gMinX: this.gMinX,
        gMaxX: this.gMaxX,
        doersLength: Object.values(this.doers).length,
      }
    },
  },

  watch: {
    // эта функция запускается при любом изменении вопроса
    selected: function() {
      this.left = 0;
      this.init(this.ctx)
      this.drawDates()
    },
  },

  mounted() {
    let canvas = this.$refs['canvas']
    // Установить размеры для canvas элемента
    canvas.width = canvas.parentElement.clientWidth
    canvas.height = this.$refs['canvas'].parentElement.clientHeight

    // Сохранить в this.ctx 2d context canvas'а
    this.ctx = canvas.getContext('2d')

    let dragging = false;
    let lastX;

    let context = this

    canvas.addEventListener('mousedown', function(e) {
        var evt = e || event;
        dragging = true;
        lastX = evt.clientX;
        e.preventDefault();
    }, false);

    window.addEventListener('mousemove', function(e) {
        var evt = e || event;
        if (dragging) {
            let delta = evt.clientX - lastX;
            lastX = evt.clientX;
            context.left += delta;
            context.init(context.ctx)
            if (context.left >= 0) {
              context.left = 0;
            }
            context.drawDates()
        }
        e.preventDefault();
    }, false);

    window.addEventListener('mouseup', function() {
        dragging = false;
    }, false);

    this.init(this.ctx)
    this.drawDates()
  },

  methods: {
    // TODO: отрисовывать на canvas шкалу времени
    // с учётом координат по оси X
    init(ctx) {
      this.allDates = []
      ctx.clearRect(
        0,
        0,
        this.$refs['canvas'].width,
        this.$refs['canvas'].height,
      )
     
      // this.userY = 0
      this.widthUnit = 0


      this.drawHorizontalLine(HEIGHT_OF_LABELS, 1, 'blue')

      for (let i = 0; i < Object.values(this.doers).length; i++) {
        this.drawHorizontalLine(
          HEIGHT_OF_LABELS + ROW_HEIGHT / 2 + i * ROW_HEIGHT,
        )
      }
      // Prepare data
      let keys = Object.keys(calendarData.calendar)
      if (Array.isArray(keys) && keys.length > 0) {
        let allDates = []
        let gMinX, gMaxX
        keys.forEach(doerId => {
          let doerBlocks = calendarData.calendar[doerId]
          if (Array.isArray(doerBlocks) && doerBlocks.length > 0) {
            doerBlocks.forEach(block => {
              if (!gMinX) gMinX = block.startAt
              if (!gMaxX) gMaxX = block.finishAt
              if (gMinX > block.startAt) gMinX = block.startAt
              if (gMaxX < block.finishAt) gMaxX = block.finishAt
              block.doerId = doerId
              allDates.push(block)
            })
          }
        })
        this.gMinX = gMinX
        this.minX = gMinX
        this.gMaxX = gMaxX
        this.allDates = allDates
        this.maxX = parseInt(
          gMinX +
            (this.$refs['canvas'].width / UNITS_IN_PX[this.selected]) *
              TIME_IN_UNITS[this.selected],
        )
      }
      this.sortDate()  
    },

    drawVerticalLine(
      offset,
      width = DEFAULT_LINE_WIDTH,
      color = DEFAULT_STROKE_COLOR,
      endY = this.$refs['canvas'].height,
      startY = HEIGHT_OF_LABELS,
    ) {
      let ctx = this.ctx
      ctx.save()
      ctx.beginPath()
      ctx.moveTo(offset, startY)
      ctx.lineTo(offset, endY)
      ctx.lineWidth = width
      ctx.strokeStyle = color
      ctx.stroke()
      ctx.restore()
    },

    drawHorizontalLine(
      offset,
      width = DEFAULT_LINE_WIDTH,
      color = DEFAULT_STROKE_COLOR, 
    ) {
      let ctx = this.ctx
      ctx.save()
      ctx.beginPath()
      ctx.moveTo(0, offset)
      ctx.lineWidth = width
      ctx.strokeStyle = color
      ctx.lineTo(this.$refs['canvas'].width, offset)
      ctx.stroke()
      ctx.restore()
    },

    sortDate() {
      this.allDates.sort((a, b) => a.finishAt - b.finishAt)
      this.gMaxX = this.allDates[this.allDates.length - 1].finishAt

      this.allDates.sort((a, b) => a.startAt - b.startAt)
      this.gMinX = this.allDates[0].startAt

      // Для проверки
     /* for (let date of this.allDates) {
        let dayMax = new Date()
        dayMax.setTime(dayMax - date.startAt)

        // eslint-disable-next-line no-console
        console.log(dayMax.toLocaleString())
      } */
      this.getDaysForScale();
    },

    getDaysForScale() {
      this.datesForScale = []

      const options = {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
      }

      let dayMax = new Date()
      dayMax.setTime(dayMax - this.gMinX)

      let dayMin = new Date()
      dayMin.setTime(dayMin - this.gMaxX)

      while (dayMax >= dayMin) {
        this.datesForScale.push(dayMax.toLocaleString('ru', options))
        dayMax = new Date(dayMax - 86400000)
      }
      // eslint-disable-next-line no-console
       // console.log(this.datesForScale)
    },

    getWeeksForScale() {
      this.datesForScale = []

      const options = {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
      }

      let dayMax = new Date()
      dayMax.setTime(dayMax - this.gMinX)

      let dayMin = new Date()
      dayMin.setTime(dayMin - this.gMaxX)

      this.datesForScale.push(dayMax.toLocaleString('ru', options))

      while (dayMax >= dayMin) {
        dayMax = new Date(dayMax - 604800000)
        this.datesForScale.push(dayMax.toLocaleString('ru', options))
        dayMax = new Date(dayMax - 86400000)
        this.datesForScale.push(dayMax.toLocaleString('ru', options))
      }
      // eslint-disable-next-line no-console
       // console.log(this.datesForScale)

    },

    getMonthesForScale() {
      this.datesForScale = []

      const options = {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
      }

      let dayMax = new Date()
      dayMax.setTime(dayMax - this.gMinX)

      let dayMin = new Date()
      dayMin.setTime(dayMin - this.gMaxX)

      this.datesForScale.push(dayMax.toLocaleString('ru', options))

      while (dayMax >= dayMin) {
        dayMax = new Date(dayMax - 2628000000)
        this.datesForScale.push(dayMax.toLocaleString('ru', options))
        dayMax = new Date(dayMax - 86400000)
        this.datesForScale.push(dayMax.toLocaleString('ru', options))
      }
    },

    getThreeMonthesForScale() {
      this.datesForScale = []

      const options = {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
      }

      let dayMax = new Date()
      dayMax.setTime(dayMax - this.gMinX)

      let dayMin = new Date()
      dayMin.setTime(dayMin - this.gMaxX)

      this.datesForScale.push(dayMax.toLocaleString('ru', options))

      while (dayMax >= dayMin) {
        dayMax = new Date(dayMax - 3 * 2628000000)
        this.datesForScale.push(dayMax.toLocaleString('ru', options))
        dayMax = new Date(dayMax - 86400000)
        this.datesForScale.push(dayMax.toLocaleString('ru', options))
      }
    },

    drawDates() {
      let ctx = this.ctx
      ctx.save()

      // Первый юнит входящий (можно частично) в область просмотра
      let start = new Date()
      start.setTime(this.minX)
      start.setHours(0, 0, 0, 0)
      // TODO: Пройти до конца области просмотра в цикле по размеру юнита
      // Подписать ось и расставить вертикали

      let widthMax = this.$refs['canvas'].width

      if (this.selected === 'DAY') {
        this.getDaysForScale()
        this.widthUnit = UNITS_IN_PX.DAY

        let maxDays = +(widthMax / this.widthUnit).toFixed(0)

        let widthMaxCanvas = (this.datesForScale.length - maxDays)*this.widthUnit + (maxDays*this.widthUnit - widthMax)
        if (this.left*(-1) > widthMaxCanvas) {
          this.left = (-1)*widthMaxCanvas
        }

        ctx.font = '11px Verdana'

        for (let day = 1; day <= this.datesForScale.length; day++) {
          this.drawVerticalLine(day * this.widthUnit + this.left)
        }

        let i = 0

        for (let date of this.datesForScale) {
          ctx.fillText(
            date,
            20 + i * this.widthUnit + this.left,
            HEIGHT_OF_LABELS - LABEL_PADDING,
          )
          i++
        }
      }
      if (
        this.selected === 'WEEK' ||
        this.selected === 'MONTH'
      ) {
        if (this.selected === 'WEEK') {
          this.getWeeksForScale()
          this.widthUnit = UNITS_IN_PX.WEEK
        } else { 
            this.getMonthesForScale()
            this.widthUnit = UNITS_IN_PX.MONTH
        }

        let maxWeeks = (widthMax / this.widthUnit).toFixed(0)

        let widthMaxCanvas = ((this.datesForScale.length/2).toFixed(0) - maxWeeks)*this.widthUnit + (maxWeeks*this.widthUnit - widthMax)
        if (this.left*(-1) > widthMaxCanvas) {
          this.left = (-1)*widthMaxCanvas
        }

        ctx.font = '11px Verdana'

        for (let day = 1; day <= this.datesForScale.length/2; day++) {
          this.drawVerticalLine(
            day * this.widthUnit + this.left,
            0.2,
            60,
            this.$refs['canvas'].height - 40 - day,
          )
        }

        for (let i = 0; i < this.datesForScale.length; i += 2) {
          if (this.datesForScale[i + 1]) {
            ctx.fillText(
              this.datesForScale[i] + ' - ' + this.datesForScale[i + 1],
              30 + (i / 2) * this.widthUnit + this.left,
              30,
            )
          } else {
            ctx.fillText(
              this.datesForScale[i],
              30 + (i / 2) * this.widthUnit + this.left,
              30,
            )
          }
        }
      }
      if (this.selected === 'QUARTER') {
        this.getThreeMonthesForScale()
        this.widthUnit = UNITS_IN_PX.QUARTER

        let maxWeeks = (widthMax / this.widthUnit).toFixed(0)

        let widthMaxCanvas = ((this.datesForScale.length/2).toFixed(0) - maxWeeks)*this.widthUnit + (maxWeeks*this.widthUnit - widthMax)
        if (this.left*(-1) > widthMaxCanvas) {
          this.left = (-1)*widthMaxCanvas
        }

        ctx.font = '11px Verdana'

        for (let day = 1; day <= this.datesForScale.length/2; day++) {
          this.drawVerticalLine(day * this.widthUnit + this.left)
        }

        for (let i = 0; i < this.datesForScale.length; i += 2) {
          if (this.datesForScale[i + 1]) {
            ctx.fillText(
              this.datesForScale[i] + ' - ' + this.datesForScale[i + 1],
              30 + (i / 2) * this.widthUnit + this.left,
              30,
            )
          } else {
            ctx.fillText(
              this.datesForScale[i],
              30 + (i / 2) * this.widthUnit + this.left,
              30,
            )
          }
        }
      }
      ctx.restore()
    },

   /* drawBlocks(user) {
    

      
       // eslint-disable-next-line no-console
       //console.log(this.allDates)
      return user
    }, */

    // Не задействовано
    // Рисует аватар пользователя
    drawAvatar(ctx, img) {
      let avatar = new Image()
      var userY = this.userY
      avatar.onload = function() {
        ctx.drawImage(avatar, 15, userY, 30, 30)
      }
      avatar.src = img
    },
  },
}
</script>

<style>
/* немного стилей, чтобы видеть где должен быть canvas элемент */
body {
  background-color: lightgray;
}
.row {
  display: flex;
}
.doers {
  margin-right: 0.5rem;
  padding-top: 40px;
}
.doers .doer {
  width: 2rem;
  height: 2rem;
  border-radius: 50%;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 0.25rem 0 0.5rem;
}
.doers .doer .img {
  width: 100%;
}
.calendar-wrap {
  box-sizing: border-box;
  width: 1000px;
  margin: 0 auto;
  background-color: #ccc;
  padding: 1rem;
}
#canvas-wrap {
  margin: 0;
  width: 800px;
  height: 600px;
  background: #fff;
  overflow: scroll;
}
h1 {
  font-family: 'Verdana';
  font-weight: 400;
  margin: 0 0 1rem;
}
span {
  font-family: 'Verdana';
  font-size: 12px;
  padding-right: 2px;
}
.options {
  padding: 0.5rem 0 0;
}
</style>
