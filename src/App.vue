<template>
  <div class="calc">
    <div class="mb-1">
      <el-button type="primary" plain @click="fetchPairs"
        >Fetch data again</el-button
      >
    </div>
    <div class="d-flex mb-1">
      <el-select
        class="mr-2 calc__select"
        v-model="input1.index"
        @change="calculate(input1, input2)"
      >
        <el-option
          v-for="index in availableIndices"
          :key="index.label"
          :label="index.label"
          :value="index.label"
        />
      </el-select>
      <el-input-number
        class="calc__number"
        v-model="input1.value"
        @change="calculate(input1, input2)"
        controls-position="right"
      />
    </div>
    <div class="d-flex">
      <el-select
        class="mr-2 calc__select"
        v-model="input2.index"
        @change="calculate(input2, input1)"
      >
        <el-option
          v-for="index in availableIndices"
          :key="index.label"
          :label="index.label"
          :value="index.label"
        />
      </el-select>
      <el-input-number
        class="calc__number"
        v-model="input2.value"
        @change="calculate(input2, input1)"
        controls-position="right"
      />
    </div>
  </div>
</template>

<script>
import axios from "axios";
import { ElNotification } from "element-plus";

const API_URL = "https://api.kraken.com/0/public";
const httpInstance = axios.create({
  baseURL: API_URL,
});

export default {
  data: () => ({
    indices: [
      {
        label: "BTC",
      },
      {
        label: "ETH",
      },
      {
        label: "USD",
      },
      {
        label: "EUR",
      },
    ],
    pairs: [
      {
        name: "BTCUSD",
        rate: 0,
        leading: "BTC",
        aliases: ["XXBTZUSD"],
      },
      {
        name: "BTCEUR",
        rate: 0,
        leading: "BTC",
        aliases: ["XXBTZEUR"],
      },
      {
        name: "ETHUSD",
        rate: 0,
        leading: "ETH",
        aliases: ["XETHZUSD"],
      },
      {
        name: "ETHEUR",
        rate: 0,
        leading: "ETH",
        aliases: ["XETHZEUR"],
      },
      {
        name: "ETHBTC",
        rate: 0,
        leading: "ETH",
        aliases: ["XETHXXBT"],
      },
      {
        name: "EURUSD",
        rate: 0,
        leading: "EUR",
        aliases: ["ZEURZUSD"],
      },
    ],
    input1: {
      index: "EUR",
      value: 0,
    },
    input2: {
      index: "BTC",
      value: 0,
    },
  }),
  computed: {
    availableIndices() {
      return this.indices.filter(
        (i) => ![this.input1.index, this.input2.index].includes(i.label)
      );
    },
  },
  async mounted() {
    await this.fetchPairs();
  },
  methods: {
    async fetchPairs() {
      try {
        const response = await httpInstance.get("Ticker", {
          params: {
            pair: this.pairs.map((pair) => pair.name).join(","),
          },
        });
        const data = response.data;

        if (data.error.length > 0) {
          ElNotification({
            title: "Error!",
            message: data.error.join("\n"),
            type: "error",
          });

          return;
        }

        for (const pair in data.result) {
          this.updatePairRate(pair, data.result[pair]);
        }

        ElNotification({
          title: "Success!",
          message: "Pairs fetched successfully",
          type: "success",
        });
      } catch (e) {
        console.error(e);
        ElNotification({
          title: "Error!",
          message: "Error occurred while fetching data",
          type: "error",
        });
      }
    },
    updatePairRate(rawIndex, ticker) {
      /*
        index =  pair name
        a = ask array(<price>, <whole lot volume>, <lot volume>),
        b = bid array(<price>, <whole lot volume>, <lot volume>),
        c = last trade closed array(<price>, <lot volume>),
        v = volume array(<today>, <last 24 hours>),
        p = volume weighted average price array(<today>, <last 24 hours>),
        t = number of trades array(<today>, <last 24 hours>),
        l = low array(<today>, <last 24 hours>),
        h = high array(<today>, <last 24 hours>),
        o = today's opening price
      */
      const pair = this.getPairByRawIndex(rawIndex);
      if (pair) {
        pair.rate = this.NaNReplacer(pair.rate, ticker.a?.[0]);
      }
    },
    //
    getPairByRawIndex(rawIndex) {
      return this.pairs.find((p) => p.aliases.includes(rawIndex));
    },
    getPairsByIndex(index) {
      return this.pairs.filter((p) => p.name.includes(index));
    },
    getPairByIndices(index1, index2) {
      return this.pairs.find(
        (p) => p.name.includes(index1) && p.name.includes(index2)
      );
    },
    //
    calculate(input1, input2) {
      const { index: i1, value = 0 } = input1;
      const { index: i2 } = input2;
      const pair = this.getPairByIndices(i1, i2);
      if (pair) {
        const isLeading = pair.leading === i1;
        if (isLeading) {
          input2.value = this.NaNReplacer(input2.value, value * pair.rate);
        } else {
          input2.value = this.NaNReplacer(input2.value, value / pair.rate);
        }
      }
    },
    //
    NaNReplacer(oldVal, newVal) {
      return Number.isNaN(newVal) ? oldVal : newVal;
    },
  },
};
</script>

<style>
body {
  font-family: "Helvetica Neue", Helvetica, "PingFang SC", "Hiragino Sans GB",
    "Microsoft YaHei", "微软雅黑", Arial, sans-serif;
}

.d-flex {
  display: flex;
}

.mr-2 {
  margin-right: 2em;
}

.mb-1 {
  margin-bottom: 1em;
}

.calc__select {
  width: 90px;
}

.calc__number {
  width: 300px !important;
}
</style>
