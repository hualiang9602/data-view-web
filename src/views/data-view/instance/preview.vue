<template>
  <div :style="layoutStyle()">
    <div
      v-for="(item, index) in items"
      :key="index"
      :style="itemStyle(item)"
      class="data-view-item"
    >
      <chart
        :id="item.elId"
        :item="item"
        :theme="screenConfig.theme"
      />
    </div>
  </div>
</template>

<script>
import '@/components/DataView/index.js'
import defaultSettings from '@/config'
import Chart from '@/components/DataView/common/chart'
import { getDataView } from '@/api/data-view'

export default {
  name: 'Preview',
  components: {
    Chart
  },
  data() {
    return {
      imageBasicUrl: defaultSettings.imageBasicUrl,
      updateOptions: {
        notMerge: true,
        lazyUpdate: false
      },
      screenConfig: {
        title: '',
        width: 1920,
        height: 1080,
        theme: '',
        backgroundImg: ''
      },
      items: []
    }
  },
  created() {
    const instanceId = this.$route.params.instance_id
    if (instanceId) {
      this.getDataView(instanceId)
    }
  },
  methods: {
    layoutStyle() {
      return {
        width: `${this.screenConfig.width}px`,
        height: `${this.screenConfig.height}px`,
        backgroundRepeat: 'no-repeat',
        backgroundSize: '100% 100%',
        backgroundImage: `url(${this.imageBasicUrl}${this.screenConfig.backgroundImg})`
      }
    },
    itemStyle(item) {
      return {
        top: `${item.style.y}px`,
        left: `${item.style.x}px`,
        width: `${item.style.width}px`,
        height: `${item.style.height}px`
      }
    },
    async getDataView(instanceId) {
      try {
        const response = await getDataView(instanceId)
        this.screenConfig.title = response.data.instance_title
        this.screenConfig.width = response.data.instance_width
        this.screenConfig.height = response.data.instance_height
        this.screenConfig.backgroundImg = response.data.instance_background_img
        // 渲染图表数据
        const items = response.data.chart_items
        if (items !== null && items !== undefined && items.length > 0) {
          items.map((item) => {
            item.data = JSON.parse(item.data)
            item.chartData = JSON.parse(item.chartData)
            item.option = JSON.parse(item.option)
            return item
          })
          this.instanceVersion = response.data.instance_version
          this.items = items
        }
      } catch (e) {
        console.log('获取大屏信息或解析大屏信息失败', e)
      }
    }
  }
}
</script>
<style lang="less">
body {
  overflow: hidden;
}

.data-view-item {
  position: absolute;

  .data-view-chart {
    width: 100%;
    height: 100%;

    .chart {
      width: 100%;
      height: 100%;
    }
  }
}
</style>
