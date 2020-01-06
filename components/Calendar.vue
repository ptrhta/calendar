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
      <div class="time">
        <span>Время: {{ time }}</span>
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
 const options = {
  year: 'numeric',
  month: 'long',
  day: 'numeric',
}
const optionsFull = {
  year: 'numeric',
  month: 'long',
  day: 'numeric',
  hour: 'numeric', 
  minute: 'numeric', 
  second: 'numeric',
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
    canvasWidth: 800,
    restDayFinish: 0,
    time: 0,
    restStart: 0,
    restFinish: 0,
    numberOfUnits: 0,
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
      this.drawBlocks()
      this.drawScroll()
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
        let rect = canvas.getBoundingClientRect();
        let evt = e || event;
        if (dragging) {
            let heightScrollY = rect.height - 15
            let cordsY = evt.clientY - rect.top
            let delta = evt.clientX - lastX;
            lastX = evt.clientX;
            context.left += delta;

            if ( cordsY <= rect.height + 50 && cordsY >= heightScrollY - 50) {
              context.init(context.ctx)
              for (let i = 0; i <= context.numberOfUnits / 2 + 1; i++) {
                context.left += -1*delta;
                context.drawScroll()
              }
            }

            context.init(context.ctx)
            if (context.left >= 0) {
              context.left = 0;
            }
            context.drawDates()
            context.drawBlocks()
            context.drawScroll()
        }
        e.preventDefault();
    }, false);

    window.addEventListener('mouseup', function() {
        dragging = false;
    }, false);

    this.init(this.ctx);
    this.drawDates();
    this.drawBlocks();
    this.drawScroll();

    canvas.addEventListener('mousemove', function(evt) {
        let rect = canvas.getBoundingClientRect();
        let cords = evt.clientX - rect.left - context.left
        context.getTimeLine(cords)
    }, false);

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
      for (let date of this.allDates) {
        let dayMax = new Date()
        dayMax.setTime(dayMax - date.finishAt)

        let dayStart = new Date()
        dayStart.setTime(dayStart - date.startAt)

        // eslint-disable-next-line no-console
        console.log(dayStart.toLocaleString(), dayMax.toLocaleString() )
      } 
      this.getDaysForScale();
    },

    getDaysForScale() {
      this.datesForScale = []

      let dayMax = new Date()
      dayMax.setTime(dayMax - this.gMinX)

      let dayMin = new Date()
      dayMin.setTime(dayMin - this.gMaxX)

      while (dayMax >= dayMin) {
        this.datesForScale.push(dayMax.toLocaleString('ru', options))
        dayMax = new Date(dayMax - 86400000)
      }
    },

    getRangesForScale(range) {
      this.datesForScale = []

      let dayMax = new Date()
      dayMax.setTime(dayMax - this.gMinX)

      let dayMin = new Date()
      dayMin.setTime(dayMin - this.gMaxX)

      this.datesForScale.push(dayMax.toLocaleString('ru', options))

      while (dayMax >= dayMin) {
        dayMax = new Date(dayMax - range)
        this.datesForScale.push(dayMax.toLocaleString('ru', options))
        dayMax = new Date(dayMax - 86400000)
        this.datesForScale.push(dayMax.toLocaleString('ru', options))
      }
      this.restDayFinish = dayMax
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
          this.getRangesForScale(604800000)
          this.widthUnit = UNITS_IN_PX.WEEK
        } else { 
            this.getRangesForScale(2628000000)
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
            this.$refs['canvas'].height - 10 - day,
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
        this.getRangesForScale(3 * 2628000000)
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

    drawBlocks() {
      let ctx = this.ctx
      ctx.save()

      this.numberOfUnits = this.datesForScale.length;
      let startDay = new Date()
      startDay.setTime(startDay - this.gMinX)
      let restDayStart = new Date(startDay.getFullYear(), startDay.getMonth(), startDay.getDate() + 1)

      let finishDay = new Date()
      finishDay.setTime(finishDay - this.gMaxX)

      let restDayFinish;

      if (this.selected === "DAY") {
        restDayFinish = new Date(finishDay.getFullYear(), finishDay.getMonth(), finishDay.getDate())
        this.numberOfUnits = this.datesForScale.length
      } else if (this.selected === "WEEK" || this.selected === "MONTH" || this.selected === "QUARTER") {
        restDayFinish = new Date(this.restDayFinish)
        this.numberOfUnits = (this.datesForScale.length / 2).toFixed(0) - 1
      }

      this.restStart = restDayStart - startDay

      this.restFinish = finishDay - restDayFinish

      this.allDates.sort((a, b) => a.doerId - b.doerId)

      let padding = HEIGHT_OF_LABELS + LABEL_PADDING;

      for (let i=0; i<this.allDates.length; i++) {
        if (this.allDates[i-1]&&this.allDates[i].doerId != this.allDates[i-1].doerId) {
          padding+=ROW_HEIGHT
        }

      let startBlock = this.numberOfUnits*this.widthUnit / (this.gMaxX - this.gMinX + this.restStart + this.restFinish) * ( this.allDates[i].startAt - this.gMinX + this.restStart)

      let finishBlock = this.numberOfUnits*this.widthUnit / (this.gMaxX - this.gMinX + this.restStart + this.restFinish) * ( this.allDates[i].finishAt - this.gMinX + this.restStart)

        ctx.fillRect(startBlock + this.left, padding, finishBlock - startBlock, 24);
      }
       ctx.restore()
    },

    getTimeLine(cords) {
      let ctx = this.ctx

      this.init(ctx)
      this.drawDates()
      this.drawBlocks()
      this.drawScroll()

      let ms = (this.gMaxX - this.gMinX + this.restStart + this.restFinish) * cords / (this.numberOfUnits*this.widthUnit) - this.restStart + this.gMinX

      this.drawVerticalLine(
            cords + this.left,
            .4,
            'blue',
            HEIGHT_OF_LABELS,
            this.$refs['canvas'].height - 10,
            )

      let time = new Date()
       time.setTime(time - ms)

      this.time = time.toLocaleString('ru', optionsFull)
    },

    drawScroll() {
      let ctx = this.ctx
      ctx.save()
      let startScroll = 0 

      this.drawHorizontalLine(this.$refs['canvas'].height - 10, .4, 'blue',)

      ctx.fillStyle = "#f5f7f7"
      ctx.fillRect(0, this.$refs['canvas'].height - 10, this.$refs['canvas'].width, 10);

      let numberOfUnits = this.numberOfUnits

      if (this.selected === "WEEK" || this.selected === "MONTH" || this.selected === "QUARTER") {
        numberOfUnits = this.numberOfUnits + 1
      }

      let widthScroll = this.$refs['canvas'].width / (numberOfUnits*this.widthUnit) * this.$refs['canvas'].width 
      startScroll =  - this.left * this.$refs['canvas'].width / (numberOfUnits*this.widthUnit)
      ctx.fillStyle = "#696969"
      ctx.fillRect(startScroll, this.$refs['canvas'].height - 10, widthScroll, 10);

      ctx.restore()
    },

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
  overflow-y: hidden;
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
.row .time {
  margin-left: 32rem;
  padding-top: 2px;

}
</style>
