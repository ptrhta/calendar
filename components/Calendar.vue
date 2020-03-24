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
const OPTIONS = {
  year: 'numeric',
  month: 'long',
  day: 'numeric',
}
const OPTIONS_FOR_MONTH = {
  month: 'long',
}
const OPTIONS_FULL = {
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
    range: 0,
    rangesForScale: [],
    startMonthesForScale: [],
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
      minX.setTime(this.minX)

      let maxX = new Date()
      maxX.setTime(this.maxX)

      let dates = []

      // Для проверки
      for (let date of this.allDates) {
        let dayMax = new Date()
        dayMax.setTime(date.finishAt)

        let dayStart = new Date()
        dayStart.setTime(date.startAt)

        dates.push([dayStart.toLocaleString(), dayMax.toLocaleString()])
      }
      return {
        minX,
        maxX,
        gMinX: this.gMinX,
        gMaxX: this.gMaxX,
        doersLength: Object.values(this.doers).length,
        dates,
      }
    },
  },

  watch: {
    // эта функция запускается при любом изменении вопроса
    selected: function() {
      this.drawing()
    },
  },

  mounted() {
    let canvas = this.$refs['canvas']
    // Установить размеры для canvas элемента
    canvas.width = canvas.parentElement.clientWidth
    canvas.height = this.$refs['canvas'].parentElement.clientHeight

    // Сохранить в this.ctx 2d context canvas'а
    this.ctx = canvas.getContext('2d')

    this.widthUnit = UNITS_IN_PX.DAY

    this.drawing()

    let dragging = false
    let lastX

    let mountedThis = this

    canvas.addEventListener(
      'mousedown',
      function(e) {
        var evt = e || event
        dragging = true
        lastX = evt.clientX
        e.preventDefault()
      },
      false,
    )

    window.addEventListener(
      'mousemove',
      function(e) {
        let rect = canvas.getBoundingClientRect()
        let evt = e || event

        if (dragging) {
          let heightScrollY = rect.height - 15
          let cordsY = evt.clientY - rect.top
          let delta = evt.clientX - lastX
          lastX = evt.clientX
          mountedThis.left += delta

          if (cordsY <= rect.height + 50 && cordsY >= heightScrollY - 50) {
            mountedThis.init(mountedThis.ctx)
            for (let i = 0; i <= mountedThis.numberOfUnits / 4 + 4; i++) {
              mountedThis.left += -1 * delta
              mountedThis.drawScroll()
            }
          }

          mountedThis.init(mountedThis.ctx)
          if (mountedThis.left >= 0) {
            mountedThis.left = 0
          }

          mountedThis.drawDates()
          mountedThis.drawBlocks()
          mountedThis.drawScroll()
        }
        e.preventDefault()
      },
      false,
    )

    window.addEventListener(
      'mouseup',
      function() {
        dragging = false
      },
      false,
    )

    canvas.addEventListener(
      'mousemove',
      function(evt) {
        let rect = canvas.getBoundingClientRect()
        let cords = evt.clientX - rect.left - mountedThis.left
        mountedThis.getTimeLine(cords)
      },
      false,
    )
  },

  methods: {
    // TODO: отрисовывать на canvas шкалу времени
    // с учётом координат по оси X
    drawing() {
      this.init(this.ctx)
      this.getNumberOfUnits()
      this.left = -this.numberOfUnits * this.widthUnit
      this.drawDates()
      this.drawBlocks()
      this.drawScroll()
    },
    init(ctx) {
      this.allDates = []
      ctx.clearRect(
        0,
        0,
        this.$refs['canvas'].width,
        this.$refs['canvas'].height,
      )

      this.drawHorizontalLine(HEIGHT_OF_LABELS, 1, 'rgba(71, 60, 59, 1)')

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
              if (!gMinX) gMinX = block.finishAt
              if (!gMaxX) gMaxX = block.startAt
              if (gMinX > block.finishAt) gMinX = block.finishAt
              if (gMaxX < block.startAt) gMaxX = block.startAt
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

      // eslint-disable-next-line no-console
      // console.log(this.gMinX, this.gMaxX )
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
      this.maxX = this.allDates[this.allDates.length - 1].startAt

      this.allDates.sort((a, b) => a.startAt - b.startAt)
      this.gMinX = this.allDates[0].startAt

      this.getDaysForScale()
    },

    getDaysForScale() {
      this.datesForScale = []

      let dayMin = new Date()
      dayMin.setTime(this.gMinX)

      let dayMax = new Date()
      dayMax.setTime(this.gMaxX)

      while (dayMax >= this.allDates[0].startAt || dayMax >= dayMin) {
        this.datesForScale.push(dayMax.toLocaleString('ru', OPTIONS))
        dayMax = new Date(dayMax - 86400000)
      }
    },

    getRangesForScale(range) {
      this.datesForScale = []
      this.rangesForScale = []

      let date = new Date(this.gMinX)
      let dayMin = new Date(date.getFullYear(), date.getMonth(), date.getDate())

      let dayMax = new Date()
      dayMax.setTime(this.gMaxX)

      let i = 0

      if (this.selected === 'MONTH' || this.selected === 'QUARTER') {
        this.startMonthesForScale = []
        while (dayMax >= dayMin) {
          dayMax = new Date(dayMax.getFullYear(), dayMax.getMonth() - i)
          this.datesForScale.push(
            this.ucFirst(dayMax.toLocaleString('ru', OPTIONS_FOR_MONTH)),
          )
          this.startMonthesForScale.push(dayMax)
          if (this.selected === 'MONTH') {
            this.rangesForScale.push(this.getRangeForMonth(dayMax))
            i++
          } else {
            this.rangesForScale.push(this.getRangeForQuater(dayMax))
            i += 3
          }

          // eslint-disable-next-line no-console
          console.log(this.rangesForScale, this.startMonthesForScale, i)
        }
        this.rangesForScale = this.rangesForScale.reverse()
        this.startMonthesForScale = this.startMonthesForScale.reverse()
      } else {
        this.datesForScale.push(dayMax.toLocaleString('ru', OPTIONS))
        while (dayMax >= dayMin) {
          dayMax = new Date(dayMax - range)
          this.datesForScale.push(dayMax.toLocaleString('ru', OPTIONS))
          dayMax = new Date(dayMax - 86400000)
          if (dayMax >= dayMin) {
            this.datesForScale.push(dayMax.toLocaleString('ru', OPTIONS))
          }
        }
      }

      this.getNumberOfUnits()

      this.restDayFinish = dayMax
      // eslint-disable-next-line no-console
      console.log(
        dayMax.toLocaleString(),
        this.datesForScale,
        dayMin.toLocaleString(),
        this.restDayFinish,
      )
    },
    ucFirst(str) {
      if (!str) return str
      return str[0].toUpperCase() + str.slice(1)
    },
    drawDates() {
      let ctx = this.ctx
      ctx.save()

      // TODO: Пройти до конца области просмотра в цикле по размеру юнита
      // Подписать ось и расставить вертикали

      let widthMax = this.$refs['canvas'].width

      if (this.selected === 'DAY') {
        this.range = 86400000
        this.getDaysForScale()

        this.datesForScale = this.datesForScale.reverse()

        this.widthUnit = UNITS_IN_PX.DAY

        let maxDays = +(widthMax / this.widthUnit).toFixed(0)

        let widthMaxCanvas =
          (this.datesForScale.length - maxDays) * this.widthUnit +
          (maxDays * this.widthUnit - widthMax)

        if (this.left * -1 > widthMaxCanvas) {
          this.left = -1 * widthMaxCanvas
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
      if (this.selected === 'WEEK') {
        this.getRangesForScale(518400000)
        this.datesForScale = this.datesForScale.reverse()
        this.range = 604800000
        this.widthUnit = UNITS_IN_PX.WEEK

        let maxWeeks = +(widthMax / this.widthUnit).toFixed(0)

        let widthMaxCanvas =
          ((this.datesForScale.length / 2).toFixed(0) - maxWeeks) *
            this.widthUnit +
          (maxWeeks * this.widthUnit - widthMax)
        if (this.left * -1 > widthMaxCanvas) {
          this.left = -1 * widthMaxCanvas
        }

        ctx.font = '11px Verdana'

        for (let day = 1; day <= this.datesForScale.length / 2; day++) {
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
      if (this.selected === 'MONTH') {
        this.getRangesForScale()
        this.datesForScale = this.datesForScale.reverse()
        this.widthUnit = UNITS_IN_PX.MONTH

        let maxWeeks = +(widthMax / this.widthUnit).toFixed(0)

        let widthMaxCanvas =
          (this.datesForScale.length - maxWeeks) * this.widthUnit +
          (maxWeeks * this.widthUnit - widthMax)
        if (this.left * -1 > widthMaxCanvas) {
          this.left = -1 * widthMaxCanvas
        }

        ctx.font = '11px Verdana'

        for (let day = 1; day <= this.datesForScale.length; day++) {
          this.drawVerticalLine(
            day * this.widthUnit + this.left,
            0.2,
            60,
            this.$refs['canvas'].height - 10 - day,
          )
        }

        for (let i = 0; i < this.datesForScale.length; i++) {
          ctx.fillText(
            this.datesForScale[i],
            30 + i * this.widthUnit + this.left,
            30,
          )
        }
      }
      if (this.selected === 'QUARTER') {
        this.getRangesForScale()
        this.widthUnit = UNITS_IN_PX.QUARTER
        this.datesForScale = this.datesForScale.reverse()

        let maxWeeks = (widthMax / this.widthUnit).toFixed(0)

        let widthMaxCanvas =
          ((this.datesForScale.length / 2).toFixed(0) - maxWeeks) *
            this.widthUnit +
          (maxWeeks * this.widthUnit - widthMax)
        if (this.left * -1 > widthMaxCanvas) {
          this.left = -1 * widthMaxCanvas
        }

        ctx.font = '12px Verdana'

        for (let day = 1; day <= this.datesForScale.length / 2; day++) {
          this.drawVerticalLine(day * this.widthUnit + this.left)
        }

        for (let i = 0; i < this.datesForScale.length; i += 2) {
          let period = this.datesForScale[i] + ' - ' + this.datesForScale[i + 1]
          if (this.datesForScale[i + 1]) {
            ctx.fillText(
              period,
              this.widthUnit - period.length*11 + (i / 2) * this.widthUnit + this.left,
              30,
            )
          } 
        }
      }
      ctx.restore()
    },

    getNumberOfUnits() {
      if (this.selected === 'DAY' || this.selected === 'MONTH') {
        this.numberOfUnits = this.datesForScale.length
      } else if (this.selected === 'WEEK' || this.selected === 'QUARTER') {
        this.numberOfUnits = (this.datesForScale.length / 2).toFixed(0)
      }
    },

    getRangeForMonth(date) {
      let currentTime = new Date()
      currentTime.setTime(date)

      let startOfMonth = new Date(
        currentTime.getFullYear(),
        currentTime.getMonth(),
      )

      let finishOfMonth = new Date(
        currentTime.getFullYear(),
        currentTime.getMonth() + 1,
      )

      return finishOfMonth - startOfMonth
    },

    getRangeForQuater(date) {
      let currentTime = new Date()
      currentTime.setTime(date)
      currentTime = new Date(currentTime.getFullYear(), currentTime.getMonth())

      let startOfMonth
      for (let j = 0; j < 2; j++) {
        for (let i = 0; i < this.startMonthesForScale.length; i++) {
          if (
            this.startMonthesForScale[i].toString() == currentTime.toString()
          ) {
            startOfMonth = currentTime
            break
          }
        }
        currentTime = new Date(
          currentTime.getFullYear(),
          currentTime.getMonth() + j,
        )
      }

      let finishOfMonth = new Date(
        currentTime.getFullYear(),
        currentTime.getMonth() + 3,
      )

      return finishOfMonth - startOfMonth
    },

    getCordsForMonthScale(date) {
      let currentTime = new Date()
      currentTime.setTime(date)

      let numberOfUnit

      let start = new Date(currentTime.getFullYear(), currentTime.getMonth())

      if (this.selected === 'MONTH') {
        for (let i = 0; i < this.startMonthesForScale.length; i++) {
          if (this.startMonthesForScale[i].toString() == start.toString()) {
            numberOfUnit = i
            break
          }
        }
      } else if (this.selected === 'QUARTER') {
        for (let j = 0; j < 2; j++) {
          for (let i = 0; i < this.startMonthesForScale.length; i++) {
            if (this.startMonthesForScale[i].toString() == start.toString()) {
              numberOfUnit = i
              break
            }
          }
          start = new Date(
            currentTime.getFullYear(),
            currentTime.getMonth() + j,
          )
        }
      }

      return (
        numberOfUnit * this.widthUnit +
        (this.widthUnit / this.range) * (date - start)
      )
    },

    getNumberOfUnit(coordinates) {
      return Math.trunc(coordinates / this.widthUnit)
    },

    drawBlocks() {
      let ctx = this.ctx
      ctx.save()

      let startDay = new Date()
      startDay.setTime(this.gMaxX)

      let restDayStart = new Date(
        startDay.getFullYear(),
        startDay.getMonth(),
        startDay.getDate() + 1,
      )

      let finishDay = new Date()
      finishDay.setTime(this.gMinX)

      let restDayFinish

      if (this.selected === 'DAY') {
        restDayFinish = new Date(
          finishDay.getFullYear(),
          finishDay.getMonth(),
          finishDay.getDate(),
        )
      } else if (this.selected === 'WEEK' || this.selected === 'QUARTER') {
        restDayFinish = new Date(
          this.restDayFinish.getFullYear(),
          this.restDayFinish.getMonth(),
          this.restDayFinish.getDate() + 1,
        )
        // restDayFinish = new Date(this.restDayFinish)
      } else if (this.selected === 'MONTH') {
        restDayFinish = new Date(
          this.restDayFinish.getFullYear(),
          this.restDayFinish.getMonth(),
        )
      }

      this.restStart = restDayStart - startDay
      this.restFinish = finishDay - restDayFinish

      this.allDates.sort((a, b) => a.doerId - b.doerId)

      let padding = HEIGHT_OF_LABELS + LABEL_PADDING

      // eslint-disable-next-line no-console
      //console.log(restDayFinish.toLocaleString())

      for (let i = 0; i < this.allDates.length; i++) {
        if (
          this.allDates[i - 1] &&
          this.allDates[i].doerId != this.allDates[i - 1].doerId
        ) {
          padding += ROW_HEIGHT
        }

        let startCords =
          (this.widthUnit / this.range) *
          (this.allDates[i].startAt - restDayFinish)

        if (this.selected === 'MONTH') {
          this.range = this.getRangeForMonth(this.allDates[i].startAt)
          startCords = this.getCordsForMonthScale(this.allDates[i].startAt)
        } else if (this.selected === 'QUARTER') {
          this.range = this.getRangeForQuater(this.allDates[i].startAt)
          startCords = this.getCordsForMonthScale(this.allDates[i].startAt)
        }

        let finishCords =
          (this.widthUnit / this.range) *
          (this.allDates[i].finishAt - restDayFinish)

        if (this.selected === 'MONTH') {
          this.range = this.getRangeForMonth(this.allDates[i].finishAt)
          finishCords = this.getCordsForMonthScale(this.allDates[i].finishAt)
        } else if (this.selected === 'QUARTER') {
          this.range = this.getRangeForQuater(this.allDates[i].finishAt)
          finishCords = this.getCordsForMonthScale(this.allDates[i].finishAt)
        }

        ctx.fillStyle = 'rgba(55, 61, 51, 1)'
        ctx.fillRect(
          startCords + this.left,
          padding,
          finishCords - startCords,
          24,
        )
      }
      ctx.restore()
    },

    getTimeLine(cords) {
      let ctx = this.ctx

      this.init(ctx)
      this.drawDates()
      this.drawBlocks()
      this.drawScroll()

      let numberOfUnit = Math.trunc(cords / this.widthUnit)

      let ms =
        (this.range * cords) / this.widthUnit - this.restFinish + this.gMinX

      if (this.selected === 'MONTH') {
        this.range = this.rangesForScale[this.getNumberOfUnit(cords)]
        ms =
          +this.startMonthesForScale[this.getNumberOfUnit(cords)] +
          ((cords - numberOfUnit * UNITS_IN_PX.MONTH) / this.widthUnit) *
            this.range
      } else if (this.selected === 'QUARTER') {
        this.range = this.rangesForScale[0]
        ms =
          +this.startMonthesForScale[0] +
          ((cords - numberOfUnit * this.widthUnit) / this.widthUnit) *
            this.range
      }

      this.drawVerticalLine(
        cords + this.left,
        0.9,
        'rgba(242, 128, 134, 1)',
        HEIGHT_OF_LABELS,
        this.$refs['canvas'].height - 10,
      )

      let time = new Date()
      time.setTime(ms)

      let timeNow = new Date(
        time.getFullYear(),
        time.getMonth(),
        time.getDate(),
        time.getHours(),
        time.getMinutes(),
      )

      // eslint-disable-next-line no-console
      console.log(
        this.startMonthesForScale,
        this.getNumberOfUnit(cords),
        this.widthUnit,
      )

      this.time = timeNow.toLocaleString('ru', OPTIONS_FULL)
    },

    drawScroll() {
      let ctx = this.ctx
      ctx.save()
      let startScroll = 0

      this.drawHorizontalLine(this.$refs['canvas'].height - 10, 0.4, 'blue')

      ctx.fillStyle = '#f5f7f7'
      ctx.fillRect(
        0,
        this.$refs['canvas'].height - 10,
        this.$refs['canvas'].width,
        10,
      )

      let numberOfUnits = this.numberOfUnits

      let widthScroll =
        (this.$refs['canvas'].width / (numberOfUnits * this.widthUnit)) *
        this.$refs['canvas'].width
      startScroll =
        (-this.left * this.$refs['canvas'].width) /
        (numberOfUnits * this.widthUnit)
      ctx.fillStyle = '#696969'
      ctx.fillRect(
        startScroll,
        this.$refs['canvas'].height - 10,
        widthScroll,
        10,
      )
      ctx.restore()
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
