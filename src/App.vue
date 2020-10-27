<template>
  <div id="app">
    <ul id="coin-list">
      <li
        @click="selectCoin(coin.Name)"
        v-for="coin in coinList"
        :key="coin.Name"
      >
        {{ coin.FullName }}
      </li>
    </ul>

    <hr />

    <div id="canvas-wrapper" class="container">
      <canvas width="1000" height="520" id="canvas"></canvas>
      <canvas id="tip" width="1000" height="520"></canvas>
    </div>
  </div>
</template>

<script>
import axios from "axios";
//import Chart from 'chart.js'

export default {
  name: "App",
  data() {
    return {
      coinList: [],
      selectedCoinValue: null,
      coinPrices: [],
      coinTimes: [],
      labels: [],
      hightestPrice: "",
      points: []
    };
  },
  computed: {},
  methods: {
    drawCanvas() {
      let canvas = document.getElementById("canvas");
      let ctx = canvas.getContext("2d");
      for (var x = 20; x < 1000; x += 40) {
        ctx.strokeStyle = "#e8e8e8";
        ctx.beginPath();
        ctx.moveTo(x, 20);
        ctx.lineTo(x, 500);
        ctx.stroke();
      }

      for (var y = 20; y < 500; y += 45) {
        ctx.strokeStyle = "#e8e8e8";
        ctx.beginPath();
        ctx.moveTo(20, y);
        ctx.lineTo(980, y);
        ctx.stroke();
      }

      ctx.strokeStyle = "#333";
      ctx.lineWidth = 2.0;
      ctx.beginPath();
      ctx.moveTo(20, 20);
      ctx.lineTo(20, 500);
      ctx.lineTo(980, 500);
      ctx.stroke();
    },
    drawLines() {
      let ctx = document.getElementById("canvas").getContext("2d");
      let startPointX = 20;
      let startPointY = this.coinPrices[0] / this.hightestPrice;
      let percent;
      ctx.beginPath();
      ctx.moveTo(startPointX, startPointY);
      for (let i = 0; i < this.coinPrices.length; i++) {
        percent = parseFloat(this.coinPrices[i] / this.hightestPrice).toFixed(
          2
        );
        let x = 20 + i * 40;
        let y = 20 + (1 - percent) * 480;
        ctx.strokeStyle = "#ff8f2e";
        ctx.lineWidth = 3;
        ctx.lineTo(x, y);
        ctx.moveTo(x, y);
        this.drawDot(x, y);
        this.points.push({
          x: x,
          y: Math.round(y),
          price: this.coinPrices[i],
          radius: 10
        });
      }
      console.log(this.points);
      ctx.stroke();
    },
    drawDot(x, y) {
      let ctx = document.getElementById("canvas").getContext("2d");
      let circle = new Path2D();
      console.log(x, y);
      circle.arc(x, y, 5, 0, 2 * Math.PI);
      circle.strokeStyle = "red";
      ctx.stroke(circle);
    },
    selectCoin(coin) {
      axios
        .get(
          "https://min-api.cryptocompare.com/data/v2/histohour?fsym=" +
            coin +
            "&tsym=USD&limit=24"
        )
        .then(res => {
          this.selectedCoinValue = res.data.Data.Data;
          //empty coinPrices and coinTimes arrays for another selected coin
          let ctx = document.getElementById("canvas").getContext("2d");
          this.selectedCoinByTime = [];
          this.coinPrices = [];
          this.coinTimes = [];
          this.labels = [];
          this.points = [];
          ctx.clearRect(0, 0, 1000, 1000);
          this.drawCanvas();

          //get Hourly Pair OHLCV prices and epoch time

          for (let i = 0; i < this.selectedCoinValue.length; i++) {
            this.coinPrices.push(this.selectedCoinValue[i].close);
          }
          for (let j = 0; j < this.selectedCoinValue.length; j++) {
            this.coinTimes.push(this.selectedCoinValue[j].time);
          }

          //get Hourly Pair OHLCV hours

          let coinDate = [];
          this.coinTimes.forEach(e => {
            let date = new Date(e * 1000);
            coinDate.push(date.getHours() + ":00");
          });
          coinDate.push(new Date().getHours() + ":00");

          //get 24h highest price

          this.hightestPrice = Math.max.apply(null, this.coinPrices);
          this.labels = coinDate;

          this.drawLines(this.hightestPrice, this.coinPrices);

          // vertical axis' values
          ctx.fillText(this.hightestPrice, 2, 20);
          ctx.fillText(this.hightestPrice / 2, 2, 260);
          ctx.fillText(0, 2, 500);

          // horizontal axis' values

          for (let i = 0; i <= 24; i++) {
            ctx.fillText(this.labels[i], 10 + i * 40, 515);
          }
        });
    }
  },
  mounted() {
    //get list of all coins

    axios
      .get("https://min-api.cryptocompare.com/data/all/coinlist")
      .then(res => {
        this.coinList = res.data.Data;
      });

    let tip = document.getElementById("tip");
    let tipCtx = tip.getContext("2d");

    let BB = tip.getBoundingClientRect(),
      offsetX = BB.left,
      offsetY = BB.top;

    //add tooltip on mouse hover

    tip.onmousemove = e => {
      let mouseX = parseInt(e.clientX - offsetX),
        mouseY = parseInt(e.clientY - offsetY);
      if (this.hightestPrice) {
        tipCtx.clearRect(0, 0, 1000, 520);
        for (let i = 0; i <= this.points.length; i++) {
          if (this.points[i]) {
            let p = this.points[i],
              dx = mouseX - p.x,
              dy = mouseY - p.y;
            if (dx * dx + dy * dy < p.radius * p.radius)
              p.x > 975
                ? tipCtx.fillText("$ " + p.price, p.x - 30, p.y - 15)
                : tipCtx.fillText("$ " + p.price, p.x, p.y - 15);
          }
        }
      }
    };

    this.drawCanvas();
  }
};
</script>

<style lang="scss">
hr {
  display: none;
  background: #a0a0a0;
  border: 1px #a0a0a0 solid;
  margin: 20px 0;
}
#app {
  margin-top: 30px;
  height: 100vh;
  overflow-y: hidden;
  display: flex;
}
#canvas {
  border: 1px #999 solid;
}
#coin-list {
  width: 20%;
  overflow-y: scroll;
  padding: 10px;
  li {
    list-style: none;
    border-bottom: 1px #e8e8e8 solid;
    font-size: 20px;
    padding: 5px 0;
  }
  li:hover {
    cursor: pointer;
    color: #888;
  }
  li:active {
    color: #666;
  }
}
#coin-list::-webkit-scrollbar {
  display: none;
}
#canvas-wrapper {
  width: 80%;
  position: relative;
}
#tip {
  position: absolute;
  left: 15px;
}

@media (orientation: portrait) {
  #app {
    display: inline-block;
  }
  #coin-list {
    height: 40vh;
    width: 100%;
  }
  #canvas-wrapper {
    display: flex;
    justify-content: center;
    width: 100%;
    height: 50vh;
  }
  hr {
    display: block;
  }
}
</style>
