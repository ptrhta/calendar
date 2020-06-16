<template>
  <div class="calendar-wrap">
    <h1>Календарь</h1>
    <div class="row">
      <div class="doers">
        <div class="doer" v-for="doer in doers" :key="doer.id">
          <img class="img" v-if="doer.watch" :src="doer.avatar" alt="" />
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

const COLORS = {
  IN_WORK: '#00BFF5',
  DONE: '#88b378',
}

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
    userY: 160,
    horizontalLineY: 0,
    selected: 'DAY',
    allDates: [],
    doers: [],
    datesForScale: [],
    widthUnit: 0,
    left: 0,
    canvasWidth: 800,
    restDayFinish: 0,
    time: 0,
    restFinish: 0,
    numberOfUnits: 0,
    range: 0,
    rangesForScale: [],
    startMonthesForScale: [],
    drugging: false,
    lastX: 0,
    tasks: [],
    close: {},
    isInfo: false,
    info: [],
    callCount: 1,
  }),

  watch: {
    // эта функция запускается при любом изменении вопроса
    selected: function() {
      if (this.isInfo) {
        this.drawInfo(this.info)
      } else this.drawing()
    },
    doers: function() {
      this.doers
    },
  },

  computed: {
    scaleOptions() {
      return Object.keys(TIME_SCALE).map(key => ({
        text: TIME_SCALE[key],
        value: key,
      }))
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
  mounted() {
    let canvas = this.$refs['canvas']
    // Установить размеры для canvas элемента
    canvas.width = canvas.parentElement.clientWidth
    canvas.height = this.$refs['canvas'].parentElement.clientHeight

    // Сохранить в this.ctx 2d context canvas'а
    this.ctx = canvas.getContext('2d')

    this.doers = calendarData.doers

    Object.values(this.doers).forEach(doer => {
      if (doer.watch == undefined) doer.watch = true
    })

    this.widthUnit = UNITS_IN_PX.DAY

    this.drawing()
  },

  methods: {
    // TODO: отрисовывать на canvas шкалу времени
    // с учётом координат по оси X
    drawing() {
      this.init(calendarData.calendar, this.doers)
      Object.values(this.doers).forEach(doer => {
        doer.watch = true
      })
      this.addListeners()
      this.getNumberOfUnits()
      this.left = -this.numberOfUnits * this.widthUnit
      this.drawDates()
      this.calculateCords(this.allDates)
      this.drawScroll()
    },

    init(tasks, doers, paddingY) {
      this.allDates = []

      this.isInfo
        ? this.clearInfo()
        : this.ctx.clearRect(
            0,
            0,
            this.$refs['canvas'].width,
            this.$refs['canvas'].height,
          )

      this.ctx.globalCompositeOperation = 'source-over'

      let padding = paddingY || 0

      this.drawHorizontalLine(
        HEIGHT_OF_LABELS + padding,
        1,
        'rgba(71, 60, 59, 1)',
      )

      for (let i = 0; i < Object.values(doers).length; i++) {
        if (this.isInfo) {
          this.drawHorizontalLine(
            HEIGHT_OF_LABELS + ROW_HEIGHT / 2 + i * ROW_HEIGHT + padding,
            DEFAULT_LINE_WIDTH,
            DEFAULT_STROKE_COLOR,
            0,
            33,
          )

          this.drawHorizontalLine(
            HEIGHT_OF_LABELS + ROW_HEIGHT / 2 + i * ROW_HEIGHT + padding,
            DEFAULT_LINE_WIDTH,
            DEFAULT_STROKE_COLOR,
            60,
          )
        } else
          this.drawHorizontalLine(
            HEIGHT_OF_LABELS + ROW_HEIGHT / 2 + i * ROW_HEIGHT + padding,
          )
      }

      // Prepare data
      let keys = Object.keys(tasks)
      if (Array.isArray(keys) && keys.length > 0) {
        let allDates = []
        let gMinX, gMaxX
        keys.forEach(doerId => {
          let doerBlocks = tasks[doerId]

          if (Array.isArray(doerBlocks) && doerBlocks.length > 0) {
            doerBlocks.forEach(block => {
              if (!gMinX) gMinX = block.finishAt
              if (!gMaxX) gMaxX = block.startAt
              if (gMinX > block.finishAt) gMinX = block.finishAt
              if (gMaxX < block.startAt) gMaxX = block.startAt

              block.doerId = doerId
              allDates.push(block)
            })
          } else if (
            Object.prototype.toString.call(doerBlocks) === '[object Object]'
          ) {
            if (!gMinX) gMinX = doerBlocks.finishAt
            if (!gMaxX) gMaxX = doerBlocks.startAt
            if (gMinX > doerBlocks.finishAt) gMinX = doerBlocks.finishAt
            if (gMaxX < doerBlocks.startAt) gMaxX = doerBlocks.startAt

            allDates.push(doerBlocks)
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

    addListeners() {
      const canvas = this.$refs['canvas']

      canvas.addEventListener('mousedown', this.addDrugging, false)

      canvas.addEventListener('mousemove', this.mousemoveFunction, false)

      canvas.addEventListener('mouseup', this.mouseUpFunction, false)

      canvas.addEventListener('click', this.drawTreeOfSubTasks, false)
    },

    mousemoveFunction(e) {
      this.addTrackingMouseMove(e)
      this.addTrackingTimeline(e)
    },
    mouseUpFunction() {
      this.dragging = false
    },

    addDrugging(e) {
      var evt = e || event
      this.dragging = true
      this.lastX = evt.clientX
      e.preventDefault()
    },

    addTrackingTimeline(e) {
      let rect = this.$refs['canvas'].getBoundingClientRect()
      let cords = e.clientX - rect.left - this.left
      this.getTimeLine(cords, e.offsetX, e.offsetY)
    },

    addTrackingMouseMove(e) {
      let rect = this.$refs['canvas'].getBoundingClientRect()
      let evt = e || event

      if (this.dragging) {
        let heightScrollY = rect.height - 15
        let cordsY = evt.clientY - rect.top
        let delta = evt.clientX - this.lastX
        this.lastX = evt.clientX
        this.left += delta

        if (cordsY <= rect.height + 50 && cordsY >= heightScrollY - 50) {
          this.isInfo
            ? this.init(this.tasks, this.doers, 100)
            : this.init(calendarData.calendar, this.doers)
          for (let i = 0; i <= this.numberOfUnits / 4 + 4; i++) {
            this.left += -1 * delta
            this.drawScroll()
          }
        }

        this.isInfo
          ? this.init(this.tasks, this.doers, 100)
          : this.init(calendarData.calendar, this.doers)

        if (this.left >= 0) {
          this.left = 0
        }

        if (this.isInfo) {
          this.drawInfo(this.info)
        } else {
          this.drawDates()
          this.calculateCords(this.allDates)
          this.drawScroll()
        }
      }
      e.preventDefault()
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
      start = 0,
      finish = this.$refs['canvas'].width,
    ) {
      let ctx = this.ctx
      ctx.save()
      ctx.beginPath()
      ctx.moveTo(start, offset)
      ctx.lineWidth = width
      ctx.strokeStyle = color
      ctx.lineTo(finish, offset)
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
    },
    ucFirst(str) {
      if (!str) return str
      return str[0].toUpperCase() + str.slice(1)
    },
    drawDates() {
      let ctx = this.ctx
      ctx.save()

      ctx.font = '11px Verdana'
      let padding = this.isInfo ? 95 : 0

      if (this.selected === 'DAY') {
        this.calculateDates()

        for (let day = 1; day <= this.datesForScale.length; day++) {
          if (!this.isInfo) {
            this.drawVerticalLine(day * this.widthUnit + this.left)
          } else {
            if (
              day * this.widthUnit + this.left >= 33 &&
              day * this.widthUnit + this.left <= 60
            ) {
              this.drawBypass(day * this.widthUnit + this.left)
            } else
              this.drawVerticalLine(
                day * this.widthUnit + this.left,
                DEFAULT_LINE_WIDTH,
                DEFAULT_STROKE_COLOR,
                this.$refs['canvas'].height,
                HEIGHT_OF_LABELS + 70,
              )
          }
        }

        let i = 0

        for (let date of this.datesForScale) {
          ctx.fillText(
            date,
            20 + i * this.widthUnit + this.left,
            HEIGHT_OF_LABELS - LABEL_PADDING + padding,
          )
          i++
        }
      }
      if (this.selected === 'WEEK') {
        this.calculateDates()

        for (let day = 1; day <= this.datesForScale.length / 2; day++) {
          if (!this.isInfo) {
            this.drawVerticalLine(
              day * this.widthUnit + this.left,
              0.2,
              60,
              this.$refs['canvas'].height - 10 - day,
            )
          } else {
            if (
              day * this.widthUnit + this.left >= 33 &&
              day * this.widthUnit + this.left <= 60
            ) {
              this.drawBypass(day * this.widthUnit + this.left)
            } else {
              this.drawVerticalLine(
                day * this.widthUnit + this.left,
                0.2,
                60,
                this.$refs['canvas'].height - 10 - day,
                HEIGHT_OF_LABELS + 70,
              )
            }
          }
        }

        for (let i = 0; i < this.datesForScale.length; i += 2) {
          let text = this.datesForScale[i + 1]
            ? this.datesForScale[i] + ' - ' + this.datesForScale[i + 1]
            : this.datesForScale[i]

          ctx.fillText(
            text,
            30 + (i / 2) * this.widthUnit + this.left,
            30 + padding,
          )
        }
      }
      if (this.selected === 'MONTH') {
        this.calculateDates()

        for (let day = 1; day <= this.datesForScale.length; day++) {
          if (!this.isInfo) {
            this.drawVerticalLine(
              day * this.widthUnit + this.left,
              0.2,
              60,
              this.$refs['canvas'].height - 10 - day,
            )
          } else {
            if (
              day * this.widthUnit + this.left >= 33 &&
              day * this.widthUnit + this.left <= 60
            ) {
              this.drawBypass(day * this.widthUnit + this.left)
            } else {
              this.drawVerticalLine(
                day * this.widthUnit + this.left,
                0.2,
                60,
                this.$refs['canvas'].height - 10 - day,
                HEIGHT_OF_LABELS + 70,
              )
            }
          }
        }

        for (let i = 0; i < this.datesForScale.length; i++) {
          ctx.fillText(
            this.datesForScale[i],
            30 + i * this.widthUnit + this.left,
            30 + padding,
          )
        }
      }
      if (this.selected === 'QUARTER') {
        this.calculateDates()

        ctx.font = '12px Verdana'

        for (let day = 1; day <= this.datesForScale.length / 2; day++) {
          if (!this.isInfo) {
            this.drawVerticalLine(day * this.widthUnit + this.left)
          } else {
            if (
              day * this.widthUnit + this.left >= 33 &&
              day * this.widthUnit + this.left <= 60
            ) {
              this.drawBypass(day * this.widthUnit + this.left)
            } else {
              this.drawVerticalLine(
                day * this.widthUnit + this.left,
                DEFAULT_LINE_WIDTH,
                DEFAULT_STROKE_COLOR,
                this.$refs['canvas'].height,
                HEIGHT_OF_LABELS + 70,
              )
            }
          }
        }

        for (let i = 0; i < this.datesForScale.length; i += 2) {
          let period = this.datesForScale[i] + ' - ' + this.datesForScale[i + 1]
          if (this.datesForScale[i + 1]) {
            ctx.fillText(
              period,
              this.widthUnit -
                period.length * 11 +
                (i / 2) * this.widthUnit +
                this.left,
              30 + padding,
            )
          }
        }
      }
      ctx.restore()
    },

    calculateDates() {
      let widthMax = this.$refs['canvas'].width
      let widthMaxCanvas
      if (this.selected === 'DAY') {
        this.range = 86400000
        this.getDaysForScale()
        this.datesForScale = this.datesForScale.reverse()

        this.widthUnit = UNITS_IN_PX.DAY

        let maxDays = +(widthMax / this.widthUnit).toFixed(0)

        widthMaxCanvas =
          (this.datesForScale.length - maxDays) * this.widthUnit +
          (maxDays * this.widthUnit - widthMax)
      }
      if (this.selected === 'WEEK') {
        this.getRangesForScale(518400000)
        this.datesForScale = this.datesForScale.reverse()
        this.range = 604800000
        this.widthUnit = UNITS_IN_PX.WEEK

        let maxWeeks = +(widthMax / this.widthUnit).toFixed(0)

        widthMaxCanvas =
          ((this.datesForScale.length / 2).toFixed(0) - maxWeeks) *
            this.widthUnit +
          (maxWeeks * this.widthUnit - widthMax)
      }
      if (this.selected === 'MONTH') {
        this.getRangesForScale()
        this.datesForScale = this.datesForScale.reverse()
        this.widthUnit = UNITS_IN_PX.MONTH

        let maxWeeks = +(widthMax / this.widthUnit).toFixed(0)

        widthMaxCanvas =
          (this.datesForScale.length - maxWeeks) * this.widthUnit +
          (maxWeeks * this.widthUnit - widthMax)
      }
      if (this.selected === 'QUARTER') {
        this.getRangesForScale()
        this.widthUnit = UNITS_IN_PX.QUARTER
        this.datesForScale = this.datesForScale.reverse()

        let maxWeeks = (widthMax / this.widthUnit).toFixed(0)

        widthMaxCanvas =
          ((this.datesForScale.length / 2).toFixed(0) - maxWeeks) *
            this.widthUnit +
          (maxWeeks * this.widthUnit - widthMax)
      }

      if (this.left * -1 > widthMaxCanvas) {
        this.left = -1 * widthMaxCanvas
      }
    },

    drawBypass(info) {
      let users = this.tasks
      let paddingAroundImage = 26

      this.drawVerticalLine(
        info,
        DEFAULT_LINE_WIDTH,
        DEFAULT_STROKE_COLOR,
        146,
        110,
      )

      users.forEach((user, count) => {
        this.drawVerticalLine(
          info,
          DEFAULT_LINE_WIDTH,
          DEFAULT_STROKE_COLOR,
          146 + paddingAroundImage * (count + 1) + 14 * (count + 1),
          146 + paddingAroundImage * (count + 1) + 14 * count,
        )
      })

      this.drawVerticalLine(
        info,
        DEFAULT_LINE_WIDTH,
        DEFAULT_STROKE_COLOR,
        this.$refs['canvas'].height - 10,
        146 + paddingAroundImage * users.length + 14 * users.length,
      )
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

    calculateCords(blocks, blockY) {
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

      this.restFinish = finishDay - restDayFinish

      blocks.sort((a, b) => a.doerId - b.doerId)

      let padding

      if (!this.isInfo) {
        padding = blockY ? blockY : HEIGHT_OF_LABELS + LABEL_PADDING
      } else {
        padding = blockY ? blockY : HEIGHT_OF_LABELS + LABEL_PADDING + 100
      }

      for (let i = 0; i < blocks.length; i++) {
        if (blocks[i - 1] && blocks[i].doerId != blocks[i - 1].doerId) {
          padding += ROW_HEIGHT
        }

        let startCords =
          (this.widthUnit / this.range) * (blocks[i].startAt - restDayFinish)

        if (this.selected === 'MONTH') {
          this.range = this.getRangeForMonth(blocks[i].startAt)
          startCords = this.getCordsForMonthScale(blocks[i].startAt)
        } else if (this.selected === 'QUARTER') {
          this.range = this.getRangeForQuater(blocks[i].startAt)
          startCords = this.getCordsForMonthScale(blocks[i].startAt)
        }

        let finishCords =
          (this.widthUnit / this.range) * (blocks[i].finishAt - restDayFinish)

        if (this.selected === 'MONTH') {
          this.range = this.getRangeForMonth(blocks[i].finishAt)
          finishCords = this.getCordsForMonthScale(blocks[i].finishAt)
        } else if (this.selected === 'QUARTER') {
          this.range = this.getRangeForQuater(blocks[i].finishAt)
          finishCords = this.getCordsForMonthScale(blocks[i].finishAt)
        }

        blocks[i].startCordsX = startCords
        blocks[i].finishCordsX = finishCords
        blocks[i].startCordsY = padding

        let day = new Date()
        day.setTime(blocks.finishAt)

        arguments[2]
          ? this.drawTasks(
              startCords,
              finishCords,
              padding,
              blocks[i],
              arguments[2],
            )
          : this.drawTasks(startCords, finishCords, padding, blocks[i])
      }
    },

    drawTasks(startCords, finishCords, padding, task) {
      if (
        task.hasOwnProperty('subtasks') &&
        !arguments[4] &&
        task.subtasks != undefined
      ) {
        task.subtasks.forEach(subtask => {
          if (!subtask.color) {
            subtask.color = this.getColorForTask(subtask.status)
          }
        })

        const taskWithSubtasks = JSON.parse(JSON.stringify(task))
        const subtasks = taskWithSubtasks.subtasks

        let i = 0
        let length = task.subtasks.length

        subtasks.forEach(subtask => {
          subtask.startAt =
            task.startAt + i * ((task.finishAt - task.startAt) / length)
          i++
          subtask.finishAt =
            task.startAt + i * ((task.finishAt - task.startAt) / length)
        })

        let noAddColor = true

        this.calculateCords(subtasks, padding, noAddColor)
      } else {
        let ctx = this.ctx
        ctx.save()

        const color = task.hasOwnProperty('color')
          ? task.color
          : this.getColorForTask(task.status)

        if (
          (startCords + this.left >= 33 && startCords + this.left <= 60) ||
          (finishCords + this.left >= 33 && finishCords + this.left <= 60)
        ) {
          let paddingY = 0

      this.tasks.forEach(task => {
          this.drawAvatar(this.doers[task.doerId].avatar, paddingY)
        paddingY += 40
      })
        } else {
          ctx.fillRect(
            startCords + this.left,
            padding,
            Math.abs(finishCords - startCords),
            24,
          )

          ctx.beginPath()
          ctx.rect(
            startCords + this.left + 1.5,
            padding,
            Math.abs(finishCords - startCords) - 1.5,
            24,
          )
          ctx.fillStyle = color

          ctx.fill()
          ctx.strokeStyle = color
          ctx.lineJoin = 'round'
          ctx.lineWidth = 3
          ctx.stroke()
        }

        ctx.restore()
      }
    },

    getColorForTask(status) {
      if (status === 'Выполнено') {
        return COLORS.DONE
      } else {
        return COLORS.IN_WORK
      }
    },

    getTimeLine(cords, rectLeft) {
      if (this.isInfo) {
        this.drawInfo(this.info)
      } else {
        this.init(calendarData.calendar, this.doers)
        this.drawDates()
        this.calculateCords(this.allDates)
        this.drawScroll()
      }

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
      if (this.isInfo) {
        if (rectLeft >= 33 && rectLeft <= 60) {
          let users = this.tasks
          let paddingAroundImage = 26

          this.drawVerticalLine(
            cords + this.left,
            0.9,
            'rgba(242, 128, 134, 1)',
            145,
            140,
          )

          users.forEach((user, count) => {
            this.drawVerticalLine(
              cords + this.left,
              0.9,
              'rgba(242, 128, 134, 1)',
              146 + paddingAroundImage * (count + 1) + 14 * (count + 1),
              146 + paddingAroundImage * (count + 1) + 14 * count,
            )
          })

          this.drawVerticalLine(
            cords + this.left,
            0.9,
            'rgba(242, 128, 134, 1)',
            this.$refs['canvas'].height - 10,
            146 + paddingAroundImage * users.length + 14 * users.length,
          )
        } else {
          this.drawVerticalLine(
            cords + this.left,
            0.9,
            'rgba(242, 128, 134, 1)',
            140,
            this.$refs['canvas'].height - 10,
          )
        }
      } else {
        this.drawVerticalLine(
          cords + this.left,
          0.9,
          'rgba(242, 128, 134, 1)',
          HEIGHT_OF_LABELS,
          this.$refs['canvas'].height - 10,
        )
      }

      let time = new Date()
      time.setTime(ms)

      let timeNow = new Date(
        time.getFullYear(),
        time.getMonth(),
        time.getDate(),
        time.getHours(),
        time.getMinutes(),
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

    drawTreeOfSubTasks(evt) {
      let canvas = this.$refs['canvas']
      let rect = canvas.getBoundingClientRect()
      let cordsX = evt.clientX - rect.left - this.left
      let cordsY = evt.y - rect.top
      let close = false

      this.allDates.forEach(block => {
        if (
          block.startCordsX <= cordsX &&
          block.finishCordsX >= cordsX &&
          block.startCordsY <= cordsY &&
          block.startCordsY + 24 >= cordsY
        ) {
          canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height)

          Object.values(this.doers).map(doer => {
            return (doer.watch = doer.id == +block.doerId ? true : false)
          })
          let dates = []

          if (this.tasks.length) {
            this.allDates.forEach(task => {
              if (task.subtasks) {
                task.subtasks.forEach(subtask => {
                  dates.push({
                    id: subtask.id,
                    color: subtask.color,
                    startAt: subtask.startAt,
                    finishAt: subtask.finishAt,
                    comment: subtask.comment,
                    doerId: task.doerId,
                    status: subtask.status,
                    subtasks: subtask.subtasks,
                  })
                })
              }
            })
            this.allDates = dates
            if (block.subtasks) {
              let length = block.subtasks.length
              let click = cordsX
              let width = Math.abs(block.finishCordsX - block.startCordsX)
              let part = width / length

              for (let i = length - 1; i >= 0; i--) {
                if (click >= block.startCordsX + part * i) {
                  block = {
                    id: block.subtasks[i].id,
                    color: block.subtasks[i].color,
                    startAt: block.subtasks[i].startAt,
                    finishAt: block.subtasks[i].finishAt,
                    comment: block.subtasks[i].comment,
                    doerId: block.doerId,
                    status: block.subtasks[i].status,
                    subtasks: block.subtasks[i].subtasks,
                  }
                }
              }

              this.callCount = 1
            } else {
              this.closeWindow(true)
              close = true
            }
          }

          if (!close) {
            this.isInfo = true
            this.removeListeners(canvas)
            this.addListenersForSubtasks(canvas)
            this.ctx.clearRect(0, 0, canvas.width, canvas.height)
            this.info = block
            this.drawInfo(block)
          }
        }
      })
    },

    drawInfo(info) {
      this.ctx.globalCompositeOperation = 'difference'
      const ctx = this.ctx
      ctx.save()

      if (this.callCount === 1) {
        this.drawClose()
      }

      const doer = this.doers[info.doerId]
      const name = doer.firstName + ' ' + doer.middleName + ' ' + doer.lastName
      this.checkDoers(info.id)
      const tasks = this.tasks
      let paddingY = 0

      let arrayWithDoers = []

      tasks.forEach(task => {
        if (this.callCount === 1) {
          this.drawAvatar(this.doers[task.doerId].avatar, paddingY)
        }

        paddingY += 40
        arrayWithDoers.push(this.doers[task.doerId])
      })

      this.taskDoers = Array.from(new Set(arrayWithDoers.map(item => item.id)))

      this.init(tasks, this.taskDoers, 100)
      this.addListeners()
      this.getNumberOfUnits()
      this.drawDates()
      this.calculateCords(this.allDates)
      this.drawScroll()

      this.ctx.globalCompositeOperation = 'source-over'
      if (this.callCount === 1) {
        this.callCount--

        this.drawHorizontalLine(
          1.6 * HEIGHT_OF_LABELS,
          0.5,
          'rgba(71, 60, 59, 1)',
        )

        ctx.font = '24px Verdana'

        ctx.fillText(info.comment, 30, HEIGHT_OF_LABELS)

        ctx.beginPath()
        ctx.rect(info.comment.length * 19, 22, info.status.length * 12, 25)
        ctx.fillStyle = this.getColorForTask(info.status)
        ctx.fill()
        ctx.strokeStyle = this.getColorForTask(info.status)
        ctx.lineJoin = 'round'
        ctx.lineWidth = 5
        ctx.stroke()

        ctx.font = '14px Verdana'
        ctx.fillStyle = '#fff'

        ctx.fillText(
          info.status,
          info.comment.length * 19 + info.status.length * 1.7,
          HEIGHT_OF_LABELS,
        )

        ctx.font = '12px Verdana'
        ctx.fillStyle = 'black'
        ctx.fillText(name, 30, 2.2 * HEIGHT_OF_LABELS)
        ctx.fillStyle = '#00BFF5'
        ctx.fillText(doer.position, 30, 2.7 * HEIGHT_OF_LABELS)
      }

      ctx.restore()
    },

    checkDoers(taskId) {
      this.tasks = []
      let count = taskId.toString().split('.').length - 1

      this.allDates.forEach(taskDoer => {
        if (taskDoer.id == taskId) {
          this.tasks.push(taskDoer)
        } else {
          let checkSubtasks = tasks => {
            let foundTasks = []
            if (!Array.isArray(tasks)) {
              if (tasks.hasOwnProperty('subtasks')) {
                foundTasks = tasks.subtasks
              }
            } else {
              tasks.forEach(task => {
                if (Array.isArray(task)) {
                  task.forEach(sub => {
                    if (sub.hasOwnProperty('subtasks')) {
                      foundTasks.push(sub.subtasks)
                    }
                  })
                } else {
                  if (task.hasOwnProperty('subtasks')) {
                    foundTasks = task.subtasks
                  }
                }
              })
            }
            return foundTasks
          }

          let subTask = taskDoer

          for (let i = 0; i < count; i++) {
            let foundTasks = checkSubtasks(subTask)
            if (foundTasks) {
              subTask = foundTasks
            } else {
              subTask = ''
              break
            }
          }
          if (subTask) {
            if (!Array.isArray(subTask)) {
              if (subTask.id === taskId) {
                this.tasks.push({
                  doerId: taskDoer.id,
                  task: subTask,
                })
              }
            } else {
              subTask.forEach(task => {
                if (task.id === taskId) {
                  this.tasks.push({
                    doerId: taskDoer.id,
                    task,
                  })
                }
              })
            }
          }
        }
      })
    },

    removeListeners(canvas) {
      canvas.removeEventListener('mousedown', this.addDrugging, false)

      canvas.removeEventListener('mousemove', this.mousemoveFunction, false)

      canvas.removeEventListener('mouseup', this.mouseUpFunction, false)

      canvas.removeEventListener('click', this.drawTreeOfSubTasks, false)
    },

    addListenersForSubtasks(canvas) {
      canvas.addEventListener('click', this.closeWindow, false)
      canvas.addEventListener('click', this.drawTreeOfSubTasks, false)
    },

    removeListenersForSubtasks(canvas) {
      canvas.removeEventListener('click', this.closeWindow, false)
      canvas.removeEventListener('click', this.drawTreeOfSubTasks, false)
    },

    closeWindow(event) {
      const close = this.close
      const canvas = this.$refs['canvas']
      this.doers = calendarData.doers

      if (
        (event.offsetX >= close.x &&
          event.offsetX <= close.x + close.width &&
          event.offsetY >= close.y &&
          event.offsetY <= close.y + close.height) ||
        event === true
      ) {
        this.removeListenersForSubtasks(canvas)
        this.isInfo = false
        this.tasks = []
        this.callCount = 1
        this.drawing()
      }
    },

    drawAvatar(img, paddingY) {
      let userY = this.userY
      let ctx = this.ctx
      let avatar = new Image()

      ctx.save()

      avatar.onload = function() {
        ctx.beginPath()
        ctx.globalCompositeOperation = 'source-over'
        //x, y, radius, startAngle, endAngle, anticlockwise
        ctx.arc(46, userY + paddingY, 13, 0, Math.PI * 2, true)
        ctx.closePath()
        ctx.fill() // Наложение для картинки
        ctx.globalCompositeOperation = 'source-atop'
        ctx.drawImage(avatar, 33, userY - 16 + paddingY, 27, 28)
      }
      avatar.src = img
      ctx.globalCompositeOperation = 'source-over'

      ctx.restore()
    },

    clearInfo() {
      this.ctx.clearRect(0, 110, this.$refs['canvas'].width, 34)
      this.ctx.clearRect(0, 144, 33, this.$refs['canvas'].height)
      this.ctx.clearRect(
        60,
        144,
        this.$refs['canvas'].width,
        this.$refs['canvas'].height,
      )
      this.ctx.clearRect(33, 110, 27, 36)

      let users = this.tasks
      let paddingAroundImage = 26

      users.forEach((user, count) => {
        this.ctx.clearRect(
          33,
          146 + paddingAroundImage * (count + 1) + 14 * count,
          27,
          14,
        )
      })

      this.ctx.clearRect(
        33,
        146 + paddingAroundImage * users.length + 14 * (users.length - 1),
        27,
        this.$refs['canvas'].height,
      )
    },

    drawClose() {
      let imageClose = new Image()
      let ctx = this.ctx

      ctx.save()

      const canvas = this.$refs['canvas']

      this.close = {
        x: canvas.width - 50,
        y: 15,
        height: 30,
        width: 30,
        src: 'https://img.icons8.com/fluent/48/000000/close-window.png',
      }

      const close = this.close

      imageClose.onload = async function() {
        await ctx.drawImage(
          imageClose,
          close.x,
          close.y,
          close.width,
          close.height,
        )
      }
      imageClose.src = close.src

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
