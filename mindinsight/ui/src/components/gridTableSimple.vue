<!--
Copyright 2020 Huawei Technologies Co., Ltd.All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
<template>
  <div class="cl-slickgrid-container">
    <div class="data-show-container">
      <div v-show="incorrectData"
           class="error-msg-container">
        {{$t('components.gridIncorrectDataError')}}
      </div>
      <div v-show="!incorrectData && requestError"
           class="error-msg-container">
        {{errorMsg}}
      </div>
      <div v-show="!fullData.length && updated && !incorrectData && !requestError"
           class="error-msg-container">
        {{$t('components.gridTableNoData')}}
      </div>
      <div :id="itemId"
           v-show="!!fullData.length && !incorrectData"
           class="grid-item"></div>
    </div>
    <div class="operate-container"
         v-if="showOperate && (fullData.length || requestError)">
      <div class="filter-container"
           @keyup.enter="filterChange">
        <div v-for="(item, itemIndex) in filterArr"
             :key="itemIndex">
          <el-input class="filter-input"
                    :class="item.showError ? 'error-border' : ''"
                    v-model="item.model"></el-input>
          <span class="input-behind"
                v-if="itemIndex === filterArr.length - 1">{{$t('symbols.slashes')}}</span>
          <span class="input-behind"
                v-else>{{$t('symbols.point')}}</span>
        </div>
        <el-button class="filter-check"
                   size="mini"
                   v-if="!!filterArr.length"
                   @click="filterChange">
          <i class="el-icon-check"></i></el-button>
        <span class="filter-incorrect-text"
              v-if="!filterCorrect">{{$t('components.inCorrectInput')}}</span>
      </div>
      <div class="accuracy-container">
        {{$t('components.gridAccuracy')}}<el-select v-model="accuracy"
                   class="select-item"
                   @change="accuracyChange">
          <el-option v-for="item in accuracyArr"
                     :key="item.label"
                     :label="item.label"
                     :value="item.value"></el-option>
        </el-select>
        <div class="full-screen-icon"
             :title="$t('scalar.fullScreen')"
             @click="toggleFullScreen"
             :class="fullScreen ? 'active-color' : ''">
          <span><i class="el-icon-full-screen"></i></span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import 'slickgrid/css/smoothness/jquery-ui-1.11.3.custom.css';
import 'slickgrid/slick.grid.css';
import 'slickgrid/lib/jquery-3.1.0';
import 'slickgrid/lib/jquery-ui-1.9.2';
import 'slickgrid/lib/jquery.event.drag-2.3.0.js';
import 'slickgrid/slick.core.js';
import 'slickgrid/slick.dataview.js';
import 'slickgrid/slick.grid.js';
export default {
  props: {
    // Table data
    fullData: {
      type: Array,
      default() {
        return [];
      },
    },
    // Display operation Bar
    showOperate: {
      type: Boolean,
      default: true,
    },
    // Display full screen
    fullScreen: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      itemId: '', // Dom id
      gridObj: null, // slickgrid object
      columnsData: [], // Column information
      columnsLength: 0, // Column length
      filterArr: [], // Dimension selection array
      formateData: [], // formatted data
      formateArr: [], // formatted Array
      statistics: {}, // Object contain maximun and minimun
      accuracy: 5, // accuracy value
      incorrectData: false, // Wheather the dimension is correctly selected
      updated: false, // Updated
      scrollTop: false, // Wheather scroll to the top
      filterCorrect: true, // Wheather the dimension input is correct
      requestError: false, // Exceeded the specification
      errorMsg: '', // Error message
      viewResizeFlag: false, // Size reset flag
      // Accuray options
      accuracyArr: [
        {label: 0, value: 0},
        {label: 1, value: 1},
        {label: 2, value: 2},
        {label: 3, value: 3},
        {label: 4, value: 4},
        {label: 5, value: 5},
      ],
      // Table configuration items
      optionObj: {
        enableColumnReorder: false,
        enableCellNavigation: true,
        frozenColumn: 0,
        frozenRow: 0,
      },
    };
  },
  computed: {},
  watch: {},
  mounted() {
    this.init();
  },
  methods: {
    /**
     * Initialize
     */
    init() {
      this.itemId =
        `${new Date().getTime()}` + `${this.$store.state.componentsCount}`;
      this.$store.commit('componentsNum');
    },
    /**
     * Initialize dimension selection
     * @param {Array} dimension Dimension array
     * @param {String} filterStr Dimension String
     */
    initializeFilterArr(dimension, filterStr) {
      if (!filterStr) {
        this.filterArr = [];
        return;
      }
      const tempFilterArr = filterStr.slice(1, filterStr.length - 1).split(',');
      const tempArr = [];
      for (let i = 0; i < tempFilterArr.length; i++) {
        tempArr.push({
          model: tempFilterArr[i],
          max: dimension[i] - 1,
          showError: false,
        });
      }
      this.filterArr = tempArr;
    },
    /**
     * Initialize column information
     */
    formateColumnsData() {
      this.columnsData = [
        {
          id: -1,
          name: ' ',
          field: -1,
          width: 100,
          headerCssClass: 'headerStyle',
        },
      ];
      const columnSample = this.formateData[0];
      if (columnSample) {
        columnSample.forEach((num, numIndex) => {
          const order = numIndex;
          this.columnsData.push({
            id: order,
            name: order,
            field: order,
            width: 100,
            headerCssClass: 'headerStyle',
            formatter: this.formateValueColor,
          });
        });
      } else {
        this.columnsData = [];
      }
    },
    /**
     * Setting the Background color of data
     * @param {Number} row
     * @param {Number} cell
     * @param {String} value,
     * @param {Object} columnDef
     * @param {Object} dataContext
     * @return {String}
     */
    formateValueColor(row, cell, value, columnDef, dataContext) {
      if (
        !cell ||
        !value ||
        isNaN(value) ||
        value === Infinity ||
        value === -Infinity
      ) {
        return value;
      } else if (value < 0) {
        return `<span class="table-item-span" style="background:rgba(94, 124, 224, ${value /
          this.statistics.min})">${value}</span>`;
      } else {
        return `<span class="table-item-span" style="background:rgba(246, 111, 106, ${value /
          this.statistics.max})">${value}</span>`;
      }
    },
    /**
     * Convetring raw data into table data
     */
    formateGridArray() {
      if (this.fullData instanceof Array) {
        if (this.fullData.length) {
          if (this.fullData[0] instanceof Array) {
            this.formateData = this.fullData;
          } else {
            this.formateData = [this.fullData];
          }
        } else {
          this.formateData = [[]];
          this.columnsData = [];
        }
      } else {
        this.formateData = [[this.fullData]];
      }
      const tempArr = [];
      this.formateData.forEach((outerData, outerIndex) => {
        const tempData = {
          '-1': outerIndex,
        };
        outerData.forEach((innerData, innerIndex) => {
          const innerOrder = innerIndex;
          if (isNaN(innerData)) {
            tempData[innerOrder] = innerData;
          } else {
            tempData[innerOrder] = innerData.toFixed(this.accuracy);
          }
        });
        tempArr.push(tempData);
      });
      this.formateArr = tempArr;
    },
    /**
     * Update the table
     */
    updateGrid() {
      this.$nextTick(() => {
        if (!this.gridObj) {
          this.gridObj = new Slick.Grid(
              `#${this.itemId}`,
              this.formateArr,
              this.columnsData,
              this.optionObj,
          );
          this.columnsLength = this.columnsData.length;
        }
        this.gridObj.setData(this.formateArr, this.scrollTop);
        this.scrollTop = false;
        const columnsLength = this.columnsData.length;
        if (this.columnsLength !== columnsLength || this.viewResizeFlag) {
          this.gridObj.setColumns(this.columnsData);
          this.columnsLength = columnsLength;
          this.viewResizeFlag = false;
        }
        this.gridObj.render();
      });
    },
    /**
     * accuracy changed
     * @param {Number} value The value after changed
     */
    accuracyChange(value) {
      this.formateGridArray();
      if (!this.requestError && !this.incorrectData) {
        this.updateGrid();
      }
    },
    /**
     * Dimension selection changed
     */
    filterChange() {
      // 校验检索条件
      let filterCorrect = true;
      let incorrectData = false;
      let limitCount = 2;
      const tempArr = [];
      this.filterArr.forEach((filter) => {
        let value = filter.model.trim();
        if (!isNaN(value)) {
          if (
            value < -(filter.max + 1) ||
            value > filter.max ||
            value === '' ||
            value % 1
          ) {
            filter.showError = true;
            filterCorrect = false;
          } else {
            filter.showError = false;
            value = Number(value);
          }
        } else if (value === ':') {
          filter.showError = false;
          if (!limitCount) {
            incorrectData = true;
          } else {
            limitCount--;
          }
        } else {
          filter.showError = true;
          filterCorrect = false;
        }
        tempArr.push(value);
      });
      this.filterCorrect = filterCorrect;
      if (incorrectData && filterCorrect) {
        this.incorrectData = true;
        return;
      } else {
        this.incorrectData = false;
      }
      if (filterCorrect) {
        this.$emit('martixFilterChange', tempArr);
      }
    },
    /**
     * Updating Table Data
     * @param {Boolean} newDataFlag Wheather data is updated
     * @param {Array} dimension Array of dimension
     * @param {Object} statistics Object contains maximun and minimun
     * @param {String} filterStr String of dimension selection
     */
    updateGridData(newDataFlag, dimension, statistics, filterStr) {
      this.updated = true;
      this.requestError = false;
      this.$nextTick(() => {
        if (!this.fullData || !this.fullData.length) {
          return;
        }
        if (newDataFlag) {
          this.initializeFilterArr(dimension, filterStr);
          this.scrollTop = true;
        }
        if (newDataFlag || this.statistics.max === undefined) {
          this.statistics = statistics;
        }
        this.formateGridArray();
        this.formateColumnsData();
        if (!this.incorrectData) {
          this.updateGrid();
        }
      });
    },
    /**
     * Update the view Size
     */
    resizeView() {
      if (this.gridObj) {
        if (this.incorrectData || this.requestError) {
          this.viewResizeFlag = true;
        } else {
          this.$nextTick(() => {
            this.gridObj.resizeCanvas();
            this.gridObj.render();
          });
        }
      }
    },
    /**
     * Expand/Collapse in full screen
     */
    toggleFullScreen() {
      this.$emit('toggleFullScreen');
    },
    /**
     * Show Error message
     * @param {String} errorMsg Error message
     * @param {Array} dimension Array of dimension
     * @param {String} filterStr String of dimension selection
     */
    showRequestErrorMessage(errorMsg, dimension, filterStr) {
      this.errorMsg = errorMsg;
      if (!this.filterArr.length && dimension && filterStr) {
        this.initializeFilterArr(dimension, filterStr);
      }
      this.requestError = true;
    },
  },
  destroyed() {},
};
</script>
<style lang="scss">
.cl-slickgrid-container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  .data-show-container {
    width: 100%;
    flex: 1;
    .grid-item {
      width: 100%;
      height: 100%;
    }
    .error-msg-container {
      width: 100%;
      height: 100%;
      display: flex;
      align-items: center;
      justify-content: center;
    }
  }
  .info-show-container {
    width: 100%;
  }
  .operate-container {
    width: 100%;
    text-overflow: ellipsis;
    white-space: nowrap;
    overflow: hidden;
    z-index: 99;
    flex-wrap: wrap;
    .full-screen-icon {
      float: right;
      margin-left: 15px;
      height: 100%;
      line-height: 34px;
      cursor: pointer;
      :hover {
        color: #3e98c5;
      }
    }
    .active-color {
      color: #3e98c5;
    }
    .filter-container {
      float: left;
      flex-wrap: wrap;
      display: flex;
      .error-border {
        input {
          border-color: red;
        }
      }
      .filter-input {
        width: 50px;
        text-align: center;
      }
      .input-behind {
        padding: 0 5px;
      }
      .filter-incorrect-text {
        margin-left: 10px;
        line-height: 32px;
        color: red;
      }
    }
    .accuracy-container {
      float: right;
      .select-item {
        width: 60px;
      }
    }
  }
}
.slick-cell,
.slick-headerrow-column,
.slick-footerrow-column {
  padding: 0;
  border-top: none;
  border-left: none;
  text-align: center;
}
.slick-viewport-left {
  overflow: hidden !important;
}
.slick-viewport-left .slick-row {
  background-color: white !important;
  ::-webkit-scrollbar {
    width: 0px;
  }
}
.ui-widget-content {
  background: none;
}
.headerStyle {
  vertical-align: middle;
  text-align: center;
}
.filter-check {
  font-size: 18px;
  color: #00a5a7;
  cursor: pointer;
}
.table-item-span {
  display: block;
  width: 100%;
  height: 100%;
  text-align: center;
}
</style>
