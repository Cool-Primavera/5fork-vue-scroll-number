<template>
  <div class="scroll-number">
    <template v-for="(val, index) in numbers">
      <!-- 1.分隔符 -->
      <div
        class="digit"
        :style="itemStyle"
        v-if="!isNumber(val)"
        :key="'char-' + index"
      >
        <span>{{ val }}</span>
      </div>

      <!-- 2.数字 -->
      <ScrollNumberItem
        :key="getIndex(numbers, index)"
        v-else
        ref="scrollItem"
        :direction="direction"
        :transitionTime="transitionTime"
        :itemStyle="itemStyle"
      />
    </template>
  </div>
</template>

<script>
/**
 * ScrollNumber
 * @prop {number|string} value input number value
 * @prop {Boolean} numberOnly whether allow number input only
 * @prop {number} transitionTime animation transition time
 * @prop {object} itemStyle css style of each number item
 *
 * @member {Promise<number>} process a promise instance of animation process
 * @method changeTo(value):Promise trigger to change the value
 *
 * @event change(value) emit when animation done
 */
import ScrollNumberItem, { DIRECTIONS } from "./scroll-number-item";
import { getOptions } from "./options";

const isNumber = (val) => !Number.isNaN(+val);

export default {
  name: "ScrollNumber",
  components: {
    ScrollNumberItem,
  },
  props: {
    value: {
      type: [Number, String],
      default: 0,
    },
    numberOnly: {
      type: Boolean,
      default: () => getOptions().numberOnly,
    },
    transitionTime: {
      type: Number,
      default: () => getOptions().transitionTime, // default value 800
    },
    itemStyle: Object,
  },
  data() {
    return {
      innerValue: this.value,
      isNumber,
      direction: DIRECTIONS.FORWARD,

      process: Promise.resolve(),
    };
  },
  computed: {
    // 将 目标数字 转成 数组
    // - 注意: 当 innerValue 是 string 时，有分隔符的情况
    numbers() {
      return this.getNumbers(this.innerValue);
    },
  },
  watch: {
    value(newV) {
      this.changeTo(newV);
    },
    innerValue(newV, oldV) {
      let compareFn;
      if (isNumber(this.value)) {
        compareFn = (a, b) => Math.abs(newV) > Math.abs(oldV);
      } else {
        compareFn = (a, b) => newV > oldV;
      }

      if (compareFn(newV, oldV)) {
        this.direction = DIRECTIONS.FORWARD;
      } else {
        this.direction = DIRECTIONS.BACKWARD;
      }
    },
  },
  mounted() {
    this.changeTo(this.innerValue);
  },
  methods: {
    changeTo(value) {
      if (this.numberOnly && !isNumber(value)) {
        console.warn(
          "[vue-scroll-number]: You can only change value to a number"
        );
        return Promise.reject();
      }
      this.process = this.process.then(() => {
        return new Promise((resolve) => {
          this.innerValue = value;
          setTimeout(() => {
            const promises = this.getNumbers(value)
              .filter(isNumber) // 当value是string，可能有分隔符，过滤掉分隔符
              .map((item, index) => {
                const scrollItem = this.$refs.scrollItem[index];
                if (scrollItem) {
                  return scrollItem.changeTo(item); // changeTo
                } else {
                  return Promise.resolve();
                }
              });
            resolve(
              Promise.all(promises).then(
                () => (this.$emit("change", value), value)
              )
            );
          });
        });
      });
      return this.process;
    },

    getNumbers(value) {
      // const isNumber = (val) => !Number.isNaN(+val);
      return String(value)
        .split("")
        .map((it) => (isNumber(it) ? Number(it) : it));
    },

    // get index
    // except the non-number chars
    getIndex(numbers, index) {
      let nonNumCount = 0;
      for (let i = 0; i < index; i++) {
        if (!isNumber(numbers[i])) {
          nonNumCount++; // 符号的个数
        }
      }
      return index - nonNumCount; // 得到出去符号后的 index
    },
  },
};

export { DIRECTIONS };
</script>
