<template>
  <div class="configuration">
    <section
      v-for="(item, index) in specList"
      :key="index"
      class="spec-config-area"
    >
      <div
        v-if="item.specValueVOList.length"
        class="spec-mainImg-area flex-column-center"
      >
        <div
          v-if="selectSpec && selectSpec[item.name].mainImgUrls"
          class="main-img-wrap main-img-gray flex-column-center"
        >
          <img
            :src="selectSpec[item.name].mainImgUrls[0]"
            fit="fill"
            width="100%"
            height="100%"
            alt=""
          />
        </div>
        <div v-else class="main-img-wrap main-img-gray flex-column-center">
          <img class="gwm-car-default" src="../assets/logo.png" alt="" />
        </div>
      </div>
      <div
        v-if="item.specValueVOList.length"
        class="spec-title-area flex-column-center"
      >
        <div class="spec-title">
          {{ item.value }}
        </div>
        <div class="spec-value">
          {{ selectSpec[item.name].specValueName }}
        </div>
      </div>
      <!-- space-value-icons-scroll-view包含space-value-icons 此处针对某个规格项有多个规格值超出一行时，可以scroll -->
      <div
        v-if="item.specValueVOList.length"
        class="space-value-icons-scroll-view"
      >
        <div class="space-value-icons" :style="{ width: getLength(item) }">
          <template v-for="(specItem, specIndex) in item.specValueVOList">
            <div
              v-show="specItem.show"
              :key="`spec${specIndex}`"
              class="space-value-icon"
              :class="{
                'selected-spec':
                  selectSpec[item.name].specValueId === specItem.specValueId,
                'empty-spec': !specItem.full,
              }"
              @click="
                spaceValueSelect(
                  item.name,
                  specItem,
                  specItem.show,
                  index,
                  'click'
                )
              "
            >
              <img
                v-if="specItem.skuImgUrl"
                class="space-value-icon-image"
                :src="specItem.skuImgUrl"
                alt="img"
              />
              <img
                v-else
                class="space-value-icon-image"
                src="../assets/logo.png"
                alt="img"
              />
            </div>
          </template>
        </div>
      </div>
      <div
        v-if="
          item.specValueVOList.length &&
          !clickSpecItem.full &&
          clickIndex === index
        "
        class="spec-shortage-desc"
      >
        {{ book_out_stock }}
      </div>
    </section>
  </div>
</template>

<script>
// 没有返回的sku或者互斥的sku进行隐藏
// 库存不足的sku组合进行置灰
import { testData } from "../data";
export default {
  name: "specificationS",
  data() {
    return {
      specList: [],
      skuList: [],
      selectSpec: {}, // 已经选择的数据
      selectSku: {},
      fullStockSkuList: [], // 初始化时的 库存不为空的sku
      clickIndex: 0, // 默认点击第一个规格项
      clickSpecItem: null, // 点击的规格项值
      testData,
    };
  },

  beforeMount() {
    this.httpInit();
  },

  methods: {
    httpInit() {
      this.testData = testData;
      this.initData();
      this.initSelected();
      this.filterMuteList()
      this.initSpecList();
      this.initClick();
      this.getSelectSku();
      // ....
    },

    // 页面针对所有的规格值渲染后，就要初始化有效的数据，针对后台返回的有效sku组合，并且状态是SPEC_ON_SHELF
    initData() {
      this.specList =
        this.testData?.specDataTypeVOS.filter((item) => {
          return item.specValueVOList.some(
            (specItem) => specItem.onSaleStatus === "SPEC_ON_SHELF"
          );
        }) || [];
      this.skuList = this.testData?.availableGoodsSkuList || [];
    },
    // 初始化选择数据的对象，包含各个规格值组合
    initSelected() {
      this.specList.forEach((item) => {
        this.$set(this.selectSpec, item.name, "");
      });
    },
    // 互斥 初始化时 针对初始化时的sku 需要排除
    filterMuteList() {
      const availableSkuList = this.skuList;
      const mutexList = this.testData?.mutexListBos || [];
      this.skuList = availableSkuList.reduce((total, sku) => {
        const isMutex = mutexList.some((muItem) => {
          const lengthEqual =
            sku.specValueVOS.length === muItem.goodMutexBoList.length;
          const idEqual = sku.specValueVOS.every((v) => {
            return muItem.goodMutexBoList.find(
              (j) => v.specValueId === j.specValueId
            );
          });
          return lengthEqual && idEqual;
        });
        if (!isMutex) total.push(sku);
        return total;
      }, []);
    },

    // 处理规格数据成视图需要的规格类型并筛选上架的
    initSpecList() {
      this.specList = this.specList.map((item) => {
        return {
          name: item.name,
          value: item.value,
          specValueVOList: item.specValueVOList
            .filter((specItem) => specItem.onSaleStatus === "SPEC_ON_SHELF")
            .map((its) => {
              return {
                ...its,
                show: this.isShow(item.name, its), // 开始也要判断一次，防止规格项第一个被互斥了还可以选
                full: this.isFull(item.name, its),
              };
            }),
        };
      });
      console.log("初始化--initSpecList-this.specList", this.specList);
    },

    // 初始化默认选中，等于一个点击
    initClick() {
      this.specList.forEach((item) => {
        for (const specItem of item.specValueVOList) {
          if (specItem.show) {
            this.spaceValueSelect(
              item.name,
              specItem,
              specItem.show,
              0,
              "other"
            );
            return false;
          }
        }
      });
    },

    filterI18n(text) {
      if (!text) {
        return "";
      }
      const obj = JSON.parse(text);
      return obj["en_US"];
    },
    spaceValueSelect(key, value, show, index = 0, type = "") {
      if (!show) return;
      this.clickIndex = index;
      this.clickSpecItem = value;

      // key,value,show 规格项,值，是否可点击
      if (this.selectSpec[key].specValueId !== value.specValueId) {
        this.selectSpec[key] = value;
      }
      // 改变
      this.specList.forEach((item) => {
        item.specValueVOList.forEach((its) => {
          its.show = this.isShow(item.name, its);
          its.full = this.isFull(item.name, its);
        });
      });
      if (type === "click") {
        this.getSelectSku();
      }
    },
    // 判断规格是否可以选择，key是规格项名称，value是规格项值, 首先判断有skuList,而库存用其他函数判断
    isShow(key, value) {
      // console.log('isShow, key---', key) // EXTERIOR 或者 NTERIOR

      let copySelectSpec = JSON.parse(JSON.stringify(this.selectSpec));
      // 直接赋值当前验证项
      copySelectSpec[key] = value;
      let flag = this.skuList.some((item) => {
        let i = 0;
        for (let k in copySelectSpec) {
          if (
            copySelectSpec[k].specValueId !== "" &&
            item.specValueVOS
              .map((valItem) => valItem.specValueId)
              .includes(copySelectSpec[k].specValueId)
          ) {
            i++;
          } else if (copySelectSpec[k] === "") {
            i++;
          }
        }
        return i === this.specList.length;
      });
      return flag;
    },
    // 是否有库存 筛选库存置灰
    isFull(key, value) {
      let copySelectSpec = JSON.parse(JSON.stringify(this.selectSpec));
      let fullStockSkuList = this.skuList.filter(
        (item) =>
          !item.orderInventoryVerification ||
          (item.orderInventoryVerification && item.stock)
      );

      copySelectSpec[key] = value;
      let flag = fullStockSkuList.some((item) => {
        let i = 0;

        for (let k in copySelectSpec) {
          if (
            copySelectSpec[k].specValueId !== "" &&
            item.specValueVOS
              .map((valItem) => valItem.specValueId)
              .includes(copySelectSpec[k].specValueId)
          ) {
            i++;
          } else if (copySelectSpec[k].specValueId === "") {
            i++;
          }
        }
        return i === this.specList.length;
      });
      return flag;
    },
    getSelectSku() {
      const selectSpecIds = Object.values(this.selectSpec).map(
        (item) => item.specValueId
      );
      // console.log('selectSpecIds', selectSpecIds)
      const copySkuList = JSON.parse(JSON.stringify(this.skuList));
      copySkuList.forEach((item) => {
        item.specValueVOSIds = [];
        item.specValueVOS.map((its) => {
          return item.specValueVOSIds.push(its.specValueId);
        });
      });

      copySkuList.forEach((item) => {
        if (
          this.specList.length === selectSpecIds.length &&
          selectSpecIds.every((v) => item.specValueVOSIds.includes(v))
          // item.specValueVOSIds.every((v) => selectSpecIds.includes(v))
        ) {
          this.selectSku = item;
        }
      });
    },
    customBack() {
      this.$router.go(-1);
    },

    getLength(item) {
      let arr = item.specValueVOList.filter((specItem) => specItem.show);
      return `${arr.length * 2.5}rem`;
    },
  },
};
</script>

<style lang="less" scoped>
.configuration {
  width: 100%;
  &::-webkit-scrollbar {
    display: none;
  }
  section.spec-config-area {
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
    margin-top: 80px;
    .spec-mainImg-area {
      width: 750px;
      height: 422px;

      .main-img-wrap {
        width: 750px;
        height: 422px;
        img.main-img {
          width: 750px;
          height: 422px;
        }
        img.gwm-car-default {
          width: 210px;
          height: 48px;
        }
      }

      img.gesture-img {
        width: 574px;
        height: 40px;
      }
      .main-img-gray {
        background: #f5f5f5;
      }
    }
    .spec-title-area {
      margin-top: 40px;
      .spec-title {
        font-size: 40px;
        font-weight: 500;
        color: #16181a;
        line-height: 52px;
      }
      .spec-value {
        margin-top: 8px;
        font-size: 32px;
        font-weight: normal;
        color: #4d4f51;
        line-height: 44px;
      }
    }
    .space-value-icons-scroll-view {
      width: 100%;
      overflow-x: scroll;
      overflow-y: hidden;
      height: 120px;
      padding: 0 40px;
      text-align: center;
      position: relative;
      padding-bottom: 50px;
      padding-top: 50px;
      margin-top: 40px;
      &::-webkit-scrollbar {
        display: none;
      }
      .space-value-icons {
        width: auto;
        display: inline-block;

        & > div.space-value-icon {
          width: 120px;
          height: 120px;
          float: left;
          margin: 0 6px;
          padding: 24px;

          & > img {
            width: 72px;
            height: 72px;
            border-radius: 50%;
          }
        }
        & > .selected-spec {
          width: 120px;
          height: 120px;
          background: #f6f7f8;
          border-radius: 50%;
        }
        & > .empty-spec {
          opacity: 0.3;
          cursor: not-allowed;
        }
      }
    }
    .spec-shortage-desc {
      font-size: 32px;
      font-weight: normal;
      color: #909295;
      line-height: 44px;
      margin-top: 12px;
    }
  }

  .flex-column-center {
    display: flex;
    justify-content: center;
    align-items: center;
    flex-direction: column;
  }
}
</style>
