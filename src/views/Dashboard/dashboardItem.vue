<template>
  <div class="dashboard-wrapper">
    <div class="tool-bar">
      <div>
        <span class="db-name">{{ dashboard.name }}</span>
        <span>{{ dashboard.desc }}</span>
      </div>
      <div v-show="mode === 'edit'">
        <el-button type="primary" size="mini" @click="handleShare">
          分享
        </el-button>
        <el-button type="primary" size="mini" @click="handleLinkChart">
          添加图表
        </el-button>
      </div>
    </div>
    <grid-layout
      v-if="charts.length!==0"
      v-loading="loading"
      :layout="layout || []"
      :col-num="24"
      :row-height="30"
      :is-draggable="mode === 'edit'"
      :is-resizable="mode === 'edit'"
      :is-mirrored="false"
      :vertical-compact="true"
      :pane-container="false"
      :margin="[10, 10]"
      :use-css-transforms="true"
      style="min-height: 500px;"
      @layout-updated="handleLayoutChange"
    >
      <grid-item
        v-for="item in layout || []"
        :key="item.index"
        :x="item.x"
        :y="item.y"
        :w="item.w"
        :h="item.h"
        :i="item.i"
        @resized="handleResize"
      >
        <el-card v-loading="chartLoading[item.i]" class="visualize-card" body-style="padding: 10px;">
          <div slot="header" class="operation-bar">
            <div>
              <span>{{ getChartItem(item.i).chart_name }}</span>
            </div>
            <div>
              <i v-show="mode === 'edit'" class="el-icon-edit" @click="handleEdit(getChartItem(item.i))" />
              <i v-show="mode === 'edit'" class="el-icon-delete" @click="handleDelete(getChartItem(item.i))" />
              <el-tooltip :content="getChartItem(item.i).desc" class="item" effect="dark" placement="top-end">
                <i class="el-icon-info" style="color:#409eff;cursor:pointer;" />
              </el-tooltip>
            </div>
          </div>
          <visualize-panel :key="item.index" :ref="`chartInstance${item.i}`" :data="results[item.i]" :schema="getChartItem(item.i).content.allSelected" :chart-type.sync="getChartItem(item.i).content.chartType" :is-edit-mode="false" :chart-style="{height: `${item.h*30 + 10 * (item.h-1) - 60}px`}" />
        </el-card>
      </grid-item>
    </grid-layout>
    <div v-if="charts.length === 0 && mode === 'edit'" v-loading="loading" class="welcome-container">
      <el-button type="primary" size="mini" @click="handleLinkChart">
        添加图表
      </el-button>
      <div>
        <el-link type="info" :underline="false">
          <router-link to="/chartpanel/create">
            Dashboard Is Empty，Go Create Your First Chart!
          </router-link>
        </el-link>
      </div>
    </div>
    <el-dialog title="我的图表" :visible.sync="showChartList">
      <el-button type="primary" size="mini" @click="$router.push('/chartpanel/create')">
        Create New Chart
      </el-button>
      <el-table :data="myChartList">
        <el-table-column label="名称" width="200" prop="chart_name" />
        <el-table-column label="描述" prop="desc" />
        <el-table-column label="操作" align="center">
          <template slot-scope="scope">
            <el-button size="mini" type="primary" :disabled="isExisted(scope.row)" @click="linkChart(scope.row)">
              添加
            </el-button>
            <el-button size="mini" type="warning" @click="$router.push(`/chartpanel/${scope.row.chart_id}`)">
              编辑
            </el-button>
          </template>
        </el-table-column>
      </el-table>
      <span slot="footer" class="dialog-footer">
        <el-button type="primary" size="small" @click="showChartList = false">关 闭</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
import { GridLayout } from 'vue-grid-layout'
import { GridItem } from 'vue-grid-layout'
import visualizePanel from '../ChartPanel/components/visualizePanel'
import exeSql from '@/api/exeSql'
import { chartByDashboard, updateDashboard, addChartToDB, unMapChartDb } from '@/api/dashboard'
import { chartList } from '@/api/chart'
import { buildSqlSentence, buildFilterSentence } from '@/utils/buildSentence'

function isLineOverLap(line1, line2) {
  const start1 = {
    x: line1[0][0],
    y: line1[0][1]
  }
  const end1 = {
    x: line1[1][0],
    y: line1[1][1]
  }
  const start2 = {
    x: line2[0][0],
    y: line2[0][1]
  }
  const end2 = {
    x: line2[1][0],
    y: line2[1][1]
  }
  if (start1.y === start2.y && end1.y === end2.y) {
    if (start1.x >= start2.x && start1.x <= end2.x) {
      return true
    } else {
      return false
    }
  } else {
    return false
  }
}
export default {
  components: { GridLayout, GridItem, visualizePanel },
  props: {
    dashboard: {
      required: false,
      type: Object,
      default: () => {
        return {}
      }
    },
    mode: {
      required: false,
      type: String,
      default: 'view'
    }
  },
  data() {
    return {
      charts: [],
      results: {},
      loading: false,
      layout: [],
      myChartList: [],
      showChartList: false,
      chartLoading: {}
    }
  },
  watch: {
    'dashboard.objectId': {
      immediate: true,
      handler(value) {
        if (!value) return
        this.getList(value)
      }
    }
  },
  methods: {
    getList() {
      this.charts = []
      this.layout = []
      this.loading = true
      chartByDashboard(this.dashboard.objectId).then(resp => {
        this.loading = false
        this.charts = resp.data
        let filterStrs = []
        const layout = (this.dashboard.content && this.dashboard.content.layout) || []
        this.charts.forEach((chart, index) => {
          this.$set(this.results, chart.objectId, [])
          this.$set(this.chartLoading, chart.objectId, false)
          chart.content.allSelected = []
          chart.content.allSelected = chart.content.allSelected.concat(chart.content.selectedCalcul).concat(chart.content.selectedDimension)
          if (chart.content.filters) {
            filterStrs = chart.content.filters.map(buildFilterSentence)
          }
          const sqlSentence = buildSqlSentence({
            dataSrc: chart.content.dataSrc,
            selectedCalcul: chart.content.selectedCalcul,
            selectedDimension: chart.content.selectedDimension,
            orderByStrs: chart.content.orderByStrs,
            filterStr: filterStrs.join(' and '),
            limit: chart.content.limit
          })
          this.exeSql(sqlSentence, chart, index)
          if (!layout.find(layoutItem => layoutItem.id === chart.objectId)) {
            this.generatePosition(chart, layout, index)
          }
        })
        this.layout = layout.filter(item => {
          return this.charts.find(chart => chart.objectId === item.id)
        })
        this.handleLayoutChange()
      })
    },
    getChartItem(id) {
      return this.charts.find(chart => chart.objectId === id)
    },
    handleCaculPos(layout) {
      // const layout = JSON.parse(JSON.stringify(layout))
      const bottomItems = []
      layout.forEach(i => {
        i.yOffSet = i.y + i.h
        i.xOffSet = i.x + i.w
        i.bottomLine = [[i.x, i.yOffSet], [i.xOffSet, i.yOffSet]]
        i.topLine = [[i.x, i.y], [i.xOffSet, i.y]]
      })
      layout.forEach(i => {
        const flag = layout.every(j => {
          return !isLineOverLap(i.bottomLine, j.topLine)
        })
        if (flag) {
          bottomItems.push(i)
        }
      })
      return bottomItems
    },
    generatePosition(chart, layout, index) {
      let posObj
      if (layout.length === 0) {
        posObj = {
          id: chart.objectId,
          x: 0,
          y: 0,
          w: 12,
          h: 9,
          i: chart.objectId
        }
      } else {
        const bottomItems = this.handleCaculPos(layout)
        const highestItem = bottomItems.reduce((result, item) => {
          if (result.bottomLine[0][1] > item.bottomLine[0][1]) {
            result = item
          }
          return result
        }, bottomItems[0])
        posObj = {
          id: chart.objectId,
          x: highestItem.x,
          y: highestItem.yOffSet,
          w: highestItem.w,
          h: 9,
          i: chart.objectId
        }
      }

      layout.push(posObj)
    },
    handleShare() {
      const h = this.$createElement
      const link = `https://islasher.com/vue-data-board#/fullscreendb/${this.dashboard.objectId}`
      this.$msgbox({
        title: '分享链接',
        message: h('p', null, [
          h('a', { style: 'color: #205cd8', attrs: { href: link, target: '_blank' }}, link)
        ])
      })
    },
    handleLinkChart() {
      chartList().then(resp => {
        this.myChartList = resp.data
        this.showChartList = true
      })
    },
    linkChart(chart) {
      const data = {
        chart_id: chart.chart_id,
        dashboard_id: this.dashboard.objectId
      }
      addChartToDB(data).then(resp => {
        this.showChartList = false
        this.getList()
        this.$message({
          type: 'success',
          message: '添加成功！'
        })
      })
    },
    isExisted(chart) {
      return this.charts.findIndex(item => item.objectId === chart.chart_id) >= 0
    },
    handleEdit(chart) {
      this.$router.push(`/chartpanel/${chart.objectId}`)
    },
    handleDelete(chart) {
      this.$confirm('该操作将从该 Dashboard 中删除该图表，并不会删除原图表，确认继续?', '提示', {
        confirmButtonText: 'Confirm',
        cancelButtonText: 'Cancel',
        type: 'warning'
      }).then(() => {
        // deleteChart(index)
        const deleteChartIndex = this.layout.findIndex(item => item.id === chart.objectId)
        const layout = JSON.parse(JSON.stringify(this.layout))
        layout.splice(deleteChartIndex, 1)
        this.dashboard.content.layout = layout
        const data = {
          chart_id: chart.objectId,
          dashboard_id: this.dashboard.objectId
        }
        Promise.all([updateDashboard(this.dashboard), unMapChartDb(data)]).then(resp => {
          this.getList()
          this.$message({
            type: 'success',
            message: '删除成功!'
          })
        })
      })
    },
    handleLayoutChange() {
      this.dashboard.content = this.dashboard.content || {}
      this.dashboard.content.layout = this.layout
      updateDashboard(this.dashboard).then(() => {
        console.log('layout saved')
      })
    },
    handleResize(i, newH, newW, newHPx, newWPx) {
      this.$refs[`chartInstance${i}`][0].$children[0].$emit('resized')
    },
    exeSql(sqlSentence, item, index) {
      this.$set(this.chartLoading, item.objectId, true)
      exeSql(sqlSentence).then(resp => {
        this.$set(this.chartLoading, item.objectId, false)
        this.$set(this.results, item.objectId, resp.data)
      }).catch(() => {
        this.$set(this.chartLoading, item.objectId, false)
      })
    }
  }
}
</script>
<style lang="scss" scoped>
.tool-bar {
  display: flex;
  justify-content: space-between;
  border-top: none;
  height: 45px;
  line-height: 45px;
  color:#303133;
  padding: 0 10px;
  position: relative;
  .db-name {
    font-size: 1.2em;
    font-weight: 600;
    color: #909399;
    margin-left: 0;
  }
  span {
    color: #C0C4CC;
    font-size: 0.8em;
    margin-left: 10px;
  }
}
.visualize-card {
  /deep/ .el-card__header {
    padding: 0;
    .operation-bar {
      font-size: 14px;
      display: flex;
      justify-content: space-between;
      height: 35px;
      padding: 0 10px;
      line-height: 35px;
      z-index: 9;
      i {
        margin-right: 10px;
        color: #409EFF;
        cursor: pointer;
      }
    }
  }
}
.welcome-container {
  text-align: center;
  height: 500px;
  color:#C0C4CC;
  /deep/ .el-button {
    margin-top: 200px;
    margin-bottom: 25px;
  }
}
</style>
