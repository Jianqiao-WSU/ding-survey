<template>
  <el-container class="survey">
    <el-header class="banner flex-block">
      <div class="flex-inline">
        {{ `${$utils.digitDX(activeIndex + 1)}、` }}
        <span class="flex-item ">{{ activeTopic }}</span>
      </div>
    </el-header>
    <el-main>
      <ProgressHint :data="activeSum"/>
      <div
        ref="swiper"
        class="swiper-container swiper-no-swiping">
        <div class="swiper-wrapper">
          <div
            class="swiper-slide"
            v-for="(value, index) in topicData"
            :key="index">
            <div
              class="item"
              v-for="(v, i) in value.options"
              :key="i">
              <p class="topic">
                <span class="prefix">{{ `${$utils.toLetter(i + 1)} . ` }}</span>
                <span class="elastic">{{ v.optionName }}</span>
              </p>
              <p class="option">
                <span class="elastic">
                  <el-slider
                    v-model="answerData[index].options[i].score"
                    :class="{'selected': answerData[index].options[i].score > 0}"
                    show-stops
                    :max="10"
                    :step="1"
                    :disabled="disabled"
                    :show-tooltip="false">
                  </el-slider>
                </span>
                <span class="focus-box">{{ answerData[index].options[i].score }}</span>
              </p>
            </div>
          </div>
        </div>
      </div>
    </el-main>
    <el-footer>
      <el-button
        :class="{'disabled': activeIndex === 0}"
        @touchend.native.prevent="prev">上一题
      </el-button>
      <span class="progress">{{ `${activeIndex + 1} / ${len}` }}</span>
      <el-popover
        placement="top-end"
        width="200"
        v-model="hint"
        trigger="manual">
        <p>每道题答完后不允许修改,</p>
        <p>请知悉</p>
        <div style="text-align: right; margin: 0">
          <el-button
            size="mini"
            type="text"
            @touchend.native.prevent="hint = false">等等</el-button>
          <el-button
            type="primary"
            size="mini"
            @touchend.native.prevent="activeIndex++">知道了</el-button>
        </div>
        <el-button
          v-if="activeIndex !== len - 1"
          slot="reference"
          @touchend.native.prevent="next">下一题
        </el-button>
        <el-button
          v-if="activeIndex === len - 1"
          slot="reference"
          :class="{'success': activeSum === 10}"
          @touchend.native.prevent="submitSurvey">提交
        </el-button>
      </el-popover>
    </el-footer>
  </el-container>
</template>

<script>
import Swiper from 'swiper';
import ProgressHint from '@/components/ProgressHint';

export default {
  name: 'Survey',
  components: {
    ProgressHint,
  },
  data() {
    return {
      activeIndex: 0,
      maxIndex: 0,
      code: '',
      userData: [],
      topicData: [],
      answerData: [],
      hint: false,
      once: true,
    };
  },
  computed: {
    activeTopic() {
      if (this.len === 0) {
        return 'Topic Loading...';
      }
      return this.topicData[this.activeIndex].questionName;
    },
    activeSum() {
      if (this.answerData.length === 0) {
        return 0;
      }
      return this.$utils.arrSum(this.answerData[this.activeIndex].options);
    },
    disabled() {
      return this.activeIndex < this.maxIndex;
    },
    len() {
      return this.topicData.length;
    },
  },
  watch: {
    activeIndex() {
      this.hint = false;
      this.maxIndex = Math.max(this.maxIndex, this.activeIndex);
      this.swiper.slideTo(this.activeIndex, 500, false);
      this.$nextTick(() => window.scrollTo(0, 0));
    },
    maxIndex() {
      if (this.maxIndex === this.len - 1) {
        this.$dd.setRight('提交', () => {
          this.submitSurvey();
        });
      }
    },
  },
  beforeRouteEnter(to, from, next) {
    next((vm) => {
      vm.doubt = () => {
        vm.$dd.alert({
          title: '答题须知',
          buttonName: '我已阅读，开始答题',
          message: '        本问卷共有七个部分，每部分有十项陈述。每部分的总分是10分。\n' +
          '        请将10分分配给你认为最准确地描述你的行为或感觉的项目上。\n' +
          '        你可以自由分配这10分，你认为哪一项越能反映你的行为或感受，就给这一项一个较高的分数；' +
          '这10分既可以分别打给几项，也可以只打到一项上。\n' +
          '        注意：每一部分的总分必须是10分。',
        });
      };
      vm.doubt();
      vm.$dd.setRight('答题须知', () => {
        vm.doubt();
      });
    });
  },
  created() {
    // localStorage.removeItem('USER');
    // 检测读取缓存(用户、题目、记录)
    const caches = ['user', 'topic', 'answer'];
    let isAnswer = true;
    caches.map((val) => {
      if (this.$utils.hasCache(val)) {
        const date = this.$utils.getCache(val);
        if (val === 'answer') {
          [this.maxIndex, this[`${val}Data`]] = date;
        } else {
          this[`${val}Data`] = date;
        }
        return date;
      }
      if (val === 'topic' && !this.$utils.hasCache(val)) {
        isAnswer = false;
        this.getTopic(() => this.getAnswer());
      } else if (isAnswer) {
        this[`get${val.substr(0, 1).toUpperCase() + val.substr(1)}`]();
      }
      return false;
    });
  },
  mounted() {
    this.swiper = new Swiper(this.$refs.swiper, {
      noSwiping: true,
    });
  },
  methods: {
    // 获取用户信息
    getUser() {
      if (this.$dd.version) {
        this.$dd.getAuthCode((code) => {
          this.code = code;
          this.$http.get('getUserInfo', {
            params: { code },
          })
            .then((result) => {
              this.userData = result.data;
              this.$utils.setCache('user', this.userData);
              if (this.$utils.hasFields(this.userData, 'grades')) {
                this.$router.push({ name: 'verdict' });
              }
            })
            .catch((error) => {
              this.$dd.alert({ message: `dd error: ${error}` });
            });
        });
      }
    },
    // 获取问卷题目
    getTopic(callback) {
      this.$http.get('survey')
        .then((result) => {
          this.topicData = result.data.questions;
          this.$utils.setCache('topic', this.topicData);
          return callback && callback();
        })
        .then(() => {
          this.swiper = new Swiper(this.$refs.swiper, {
            noSwiping: true,
          });
        })
        .catch((error) => {
          console.log(error);
          this.$dd.alert({ message: '获取问卷数据失败' });
        });
    },
    // 获取答题记录
    getAnswer() {
      const record = [];
      this.topicData.forEach((element) => {
        const item = {};
        this.$set(item, 'id', element.id);
        this.$set(item, 'options', []);
        element.options.forEach((el, i) => {
          const opts = {};
          this.$set(opts, 'id', i + 1);
          this.$set(opts, 'codeId', el.codeId);
          this.$set(opts, 'score', 0);
          item.options.push(opts);
        });
        record.push(item);
      });
      this.answerData = record;
    },
    prev() {
      return this.activeIndex !== 0 && this.activeIndex--;
    },
    next() {
      if (this.activeSum === 10) {
        this.hint = this.maxIndex === 0;
        if (!this.hint && this.activeIndex !== this.len - 1) {
          this.activeIndex++;
        }
      } else {
        this.$dd.vibration(100);
      }
    },
    submitSurvey() {
      const end = this.$utils.arrSum(this.answerData[this.maxIndex].options);
      if (end === 10) {
        this.$dd.confirm({ message: '确定提交问卷？' }, index => index === 1 && this.sendSubmit());
      } else {
        this.$dd.vibration(100);
      }
    },
    sendSubmit() {
      const answer = {};
      this.$set(answer, 'staff', this.userData);
      this.$set(answer, 'questions', this.answerData);
      const loading = this.$loading({
        lock: true,
        text: '提交中...',
        spinner: 'el-icon-loading',
        background: 'rgba(0, 0, 0, 0.7)',
      });
      this.$http.post('survey', answer)
        .then((result) => {
          this.$utils.setCache('user', result.data);
          loading.close();
          this.$router.push({ name: 'verdict' });
        })
        .catch((error) => {
          loading.close();
          this.$dd.alert({ message: `error${JSON.stringify(error)}` });
        });
    },
  },
};
</script>

<style scoped lang="less">
  .survey {
    background-color: #2E353D;
  }
  .swiper-slide {
    user-select: none;
    padding: 0 20px;
    box-sizing: border-box;
  }
  .banner {
    background: rgba(46, 53, 61, .7) url('/img/banner.png') 50% 50% no-repeat;
    background-size: 100%100%;
  }
  .item {
    line-height: .5rem;
    > p {
      display: flex;
      box-sizing: border-box;
    }
    .elastic {
      flex: 1;
    }
    .topic {
      .prefix {
        width: .5rem;
        text-align: left;
      }
    }
    .option {
      .elastic {
        line-height: .64rem;
        height: .72rem;
      }
    }
  }
  .progress {
    margin: 0 .7rem;
    color: #fff;
  }
  .mt-popup {
    color: #fff;
    background-color: rgba(0, 0, 0, .5);
  }
</style>
