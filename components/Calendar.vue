<template>
  <div id="canvas-wrap" ref="wrap">
    <canvas ref="canvas"/>
      <div class="scaleSelection">
      <span>Масштаб:</span>
      <select v-model="selected">
        <option disabled value="">Выберите масштаб</option>
        <option>1 день</option>
        <option>1 неделя</option>
        <option>1 месяц</option>
        <option>3 месяца</option>
      </select>
    </div>
  </div>
</template>

<script>
import calendar from "../assets/tasks.json";

export default {
  data: () => ({
    gMinX: null, // глобальный минимум по шкале X
    gMaxX: null, // глобальный максимум по шкале X
    minX: null, // минимум по шкале X
    maxX: null, // максимум по шкале X
    userY: 110,
    horizontalLineY: 0,
    selected: '',
    allDates: [],
    datesForScale: [],
    widthUnit: 0,
  }),

  watch: {
    // эта функция запускается при любом изменении вопроса
    selected: function () {
      const ctx = this.$refs['canvas'].getContext('2d');
      this.init(ctx);
      this.drawDates(ctx);
    }
  },

  mounted() {
    // TODO: Установить размеры для canvas элемента
    // Сохранить в this.ctx 2d context canvas'а, причем
    // не нужно его ставить внутри data, так как на контексте
    // не будет никаких observer'ов изменений
    // P.S.: canvas должен занимать 100% от его родителя wrap

    const ctx = this.$refs['canvas'].getContext('2d');

    this.$refs['canvas'].width = this.$refs['canvas'].parentElement.clientWidth
    this.$refs['canvas'].height = this.$refs['canvas'].parentElement.clientHeight

    this.init(ctx);
  },

  methods: {
    // TODO: отрисовывать на canvas шкалу времени
    // с учётом координат по оси X
    init(ctx) {
      this.allDates = [];
      ctx.clearRect(0, 0, this.$refs['canvas'].width,  this.$refs['canvas'].height);
      this.userY = 110;
      this.widthUnit = 0;

      this.showTitle(ctx);

      this.horizontalLineY = 90;
      this.drawHorizontalLine(ctx, this.horizontalLineY);
      this.horizontalLineY += this.horizontalLineY / 2.5;

      this.drawVerticalLine(ctx, 60, 54, .4, this.$refs['canvas'].height);

      for ( let user of calendar ) {
        // eslint-disable-next-line no-console
        console.log(user.doers[Object.keys(user.doers)].avatar);

        this.drawAvatar(ctx, user.doers[Object.keys(user.doers)].avatar);
        this.userY = this.userY + 40;

        this.drawHorizontalLine(ctx, this.horizontalLineY);
        this.horizontalLineY += 40;

        this.sortDate(user.calendar[Object.keys(user.calendar)[0]], Object.keys(user.calendar)[0]);
      }




    },
    showTitle(ctx) {
      ctx.font = "25px Verdana";
      ctx.fillText("Календарь", 20, 40);
    },
    drawAvatar(ctx, img) {
      let avatar = new Image();
      var userY = this.userY
      avatar.onload = function() {
        ctx.drawImage(avatar, 15, userY, 30, 30);
      }
      avatar.src = img;
    },
    drawVerticalLine(ctx, x, y, width, height) {
      ctx.beginPath();
      ctx.moveTo(x, y);
      ctx.lineWidth = width;
      ctx.lineTo(x, height);
      ctx.stroke();
      ctx.restore();
    },
    drawHorizontalLine(ctx, indent) {
      ctx.beginPath();
      ctx.moveTo(60, indent);
      ctx.lineWidth = .2;
      ctx.lineTo(this.$refs['canvas'].width, indent);
      ctx.stroke();
      ctx.restore();
    },
    sortDate(tasks, id) {
      for (let task of tasks )
        this.allDates.push([ task, id ]);

      this.allDates.sort((a,b)=> a[0].finishAt - b[0].finishAt);
      this.gMaxX = this.allDates[this.allDates.length - 1][0].finishAt;


      this.allDates.sort((a,b)=> a[0].startAt - b[0].startAt);
      this.gMinX = this.allDates[0][0].startAt;

      for (let date of this.allDates){
      
        let dayMax = new Date();
        dayMax.setTime( dayMax - date[0].startAt );

        // eslint-disable-next-line no-console
        console.log(dayMax.toLocaleString());
      }
    },
    getDaysForScale(){
      this.datesForScale = [];

      const options = {
          year: 'numeric',
          month: 'long',
          day: 'numeric',
        };

        let dayMax = new Date();
        dayMax.setTime( dayMax - this.gMinX );

        let dayMin = new Date();
        dayMin.setTime( dayMin - this.gMaxX );

        while ( dayMax >= dayMin ) {
          
          this.datesForScale.push(dayMax.toLocaleString("ru", options));
           dayMax = new Date(dayMax - 86400000 );
        }
    },
    getWeeksForScale(){
      this.datesForScale = [];

      const options = {
          year: 'numeric',
          month: 'long',
          day: 'numeric',
        };

        let dayMax = new Date();
        dayMax.setTime( dayMax - this.gMinX );

        let dayMin = new Date();
        dayMin.setTime( dayMin - this.gMaxX );

        this.datesForScale.push(dayMax.toLocaleString("ru", options));

        while ( dayMax >= dayMin ) {
          dayMax = new Date(dayMax - 604800000 );
          this.datesForScale.push(dayMax.toLocaleString("ru", options));
           dayMax = new Date(dayMax - 86400000 );
           this.datesForScale.push(dayMax.toLocaleString("ru", options))
        }
    },
     getMonthesForScale(){
      this.datesForScale = [];

      const options = {
          year: 'numeric',
          month: 'long',
          day: 'numeric',
        };

        let dayMax = new Date();
        dayMax.setTime( dayMax - this.gMinX );

        let dayMin = new Date();
        dayMin.setTime( dayMin - this.gMaxX );

        this.datesForScale.push(dayMax.toLocaleString("ru", options));

        while ( dayMax >= dayMin ) {
          dayMax = new Date(dayMax - 2628000000 );
          this.datesForScale.push(dayMax.toLocaleString("ru", options));
           dayMax = new Date(dayMax - 86400000 );
           this.datesForScale.push(dayMax.toLocaleString("ru", options))
        }
    },
    getThreeMonthesForScale(){
      this.datesForScale = [];

      const options = {
          year: 'numeric',
          month: 'long',
          day: 'numeric',
        };

        let dayMax = new Date();
        dayMax.setTime( dayMax - this.gMinX );

        let dayMin = new Date();
        dayMin.setTime( dayMin - this.gMaxX );

        this.datesForScale.push(dayMax.toLocaleString("ru", options));

        while ( dayMax >= dayMin ) {
          dayMax = new Date(dayMax - 3*2628000000 );
          this.datesForScale.push(dayMax.toLocaleString("ru", options));
           dayMax = new Date(dayMax - 86400000 );
           this.datesForScale.push(dayMax.toLocaleString("ru", options))
        }
    },
    drawDates(ctx) {
      let widthMax = this.$refs['canvas'].width + 60; 

      if (this.selected === '1 день') {
        this.getDaysForScale();
        this.widthUnit = 350;

        let maxDays = (widthMax / this.widthUnit).toFixed(0);

        ctx.font = "11px Verdana";

        for ( let day = 1; day<= maxDays; day++ ) {
          this.drawVerticalLine(ctx, 60 + day*this.widthUnit, 60, .2, this.$refs['canvas'].height - 40 - day)
        }

        let i = 0;
        for (let date of this.datesForScale) {
          ctx.fillText(date, 70 + i*this.widthUnit, 80);
          i++;
        }
      }
      if ((this.selected === '1 неделя') || (this.selected === '1 месяц')) {

        (this.selected === '1 неделя') ? this.getWeeksForScale() : this.getMonthesForScale();

        this.widthUnit = 700;

        let maxWeeks = (widthMax / this.widthUnit).toFixed(0);

        ctx.font = "11px Verdana";

        for ( let day = 1; day<= maxWeeks; day++ ) {
          this.drawVerticalLine(ctx, 60 + day*this.widthUnit, 60, .2, this.$refs['canvas'].height - 40 - day)
        }

        for (let i=0; i<this.datesForScale.length; i+=2) {
            if (this.datesForScale[i+1]) {
              ctx.fillText(this.datesForScale[i] + " - " + this.datesForScale[i + 1] , 70 + i/2*this.widthUnit, 80)
            } else {
             ctx.fillText(this.datesForScale[i], 70 + i/2*this.widthUnit, 80)}
           
        }
      }
      if ((this.selected === '3 месяца')) {
        this.getThreeMonthesForScale();
        this.widthUnit = 1200;

        let maxWeeks = (widthMax / this.widthUnit).toFixed(0);

        ctx.font = "11px Verdana";

        for ( let day = 1; day<= maxWeeks; day++ ) {
          this.drawVerticalLine(ctx, 60 + day*this.widthUnit, 60, .2, this.$refs['canvas'].height - 40 - day)
        }

        for (let i=0; i<this.datesForScale.length; i+=2) {
            if (this.datesForScale[i+1]) {
              ctx.fillText(this.datesForScale[i] + " - " + this.datesForScale[i + 1] , 70 + i/2*this.widthUnit, 80)
            } else {
             ctx.fillText(this.datesForScale[i], 70 + i/2*this.widthUnit, 80)}
           
        }
      }

      this.horizontalLineY = 114;

      for (let user of calendar) {
        this.drawBlocks(ctx, user);
        this.horizontalLineY+= 40;
      }
    },
    drawBlocks(ctx, user) {
      let datesForUser = user.calendar[Object.keys(user.calendar)[0]].sort((a,b)=> a.startAt - b.startAt);
      // eslint-disable-next-line no-console
      //console.log(datesForUser)

      //let lengthScale = this.gMaxX - this.gMinX;
      // eslint-disable-next-line no-console
      //console.log(lengthScale)
     // let start = 60;

      for ( let date of datesForUser) {
        // eslint-disable-next-line no-console
        //let nowDate = new Date()
        // eslint-disable-next-line no-console
      console.log( ((date.startAt) / (this.gMaxX) ) * 350 * this.datesForScale.length  )
       // ctx.fillRect(60 + (date.startAt - this.gMinX)/, this.horizontalLineY, 100,25);
        // eslint-disable-next-line no-console
        //console.log(this.gMaxX - date.startAt)
      
        //let start = 350 * this.datesForScale.length * (date.startAt + (86400000 - this.gMinX)) / ( lengthScale * 86400000^2);
        //start = ((nowDate - this.gMaxX) / (nowDate - date.startAt) ) * 350 * this.datesForScale.length * 350 
        //ctx.fillRect(60 + start , this.horizontalLineY, 1,25);

      }
      return ctx + user

    }

   

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
span {
  font-family: 'Verdana';
  font-size: 12px;
  padding-right: 2px;
}
.scaleSelection {
  position: absolute;
  margin-top: -35px;
  margin-left: 350px;
}
</style>