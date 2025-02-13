<!--suppress JSUnresolvedVariable, JSUnusedLocalSymbols -->
<template>
  <a-layout class="data-view-container">
    <a-layout-header class="data-view-header">
      <a-icon class="back" type="left-circle" @click="handleBack()" />
      <a-menu class="data-view-menu" mode="horizontal" theme="dark">
        <a-sub-menu v-for="menu in menus" :key="menu.key">
          <span slot="title">{{ menu.title }}<a-icon type="caret-down" style="margin-left: 5px" /></span>
          <template v-for="child in menu.children">
            <a-menu-item v-if="child.children === undefined" :key="child.key" :draggable="true" @dragstart="handleDragStart($event, child.key, child.version)">
              <icon class="chart-type-icon" :type="`icon-${child.icon}`" />
              <span>{{ child.title }}</span>
            </a-menu-item>
            <a-sub-menu v-else :key="child.key" :title="child.title">
              <a-menu-item v-for="grandson in child.children" :key="grandson.key" :draggable="true" @dragstart="handleDragStart($event, grandson.key, grandson.version)">
                <icon class="chart-type-icon" :type="`icon-${grandson.icon}`" />
                <span>{{ grandson.title }}</span>
              </a-menu-item>
            </a-sub-menu>
          </template>
        </a-sub-menu>
      </a-menu>
      <div class="handler">
        <a-icon type="save" @click="handleSave()" />
        <a-icon type="eye" @click="handlePreview()" />
        <a-icon type="bug" @click="handleDebug()" />
      </div>
    </a-layout-header>
    <a-layout class="data-view-main">
      <a-layout-sider v-model="layerCollapsed" collapsed-width="0" :collapsible="true">
        <layer />
      </a-layout-sider>
      <a-layout class="data-view-screen">
        <a-layout-content
          id="screenWrapper"
          class="data-view-screen-wrapper"
          @mousedown="handleItemUnClick"
          @mouseup="handleItemUnChoose"
        >
          <layout @dragover.native="handleDragOver" @drop.native="handleDrop">
            <item
              v-for="(item, index) in items"
              :key="index"
              :item="item"
              :index="index"
              :active="item === currentItem"
            >
              <chart
                :id="item.elId"
                :item="item"
                :theme="screenConfig.theme"
              />
            </item>
          </layout>
        </a-layout-content>
        <a-layout-footer class="data-view-screen-footer toolbox">
          <a-space>
            <div class="tool-item">
              <span class="label">参考线</span>
              <a-switch v-model="refLine" />
            </div>
            <div class="tool-item">
              <span class="label">指示线</span>
              <a-switch v-model="indicatorLine" />
            </div>
            <div class="tool-item">
              <span class="label" style="margin-right: 15px">缩放比例</span>
              <a-slider
                v-model="scale"
                class="data-view-scale"
                :min="20"
                :max="200"
                :marks="{ 100:{} }"
              />
              <span class="tool-btn" @click="$store.commit('autoCanvasScale')">自适应</span>
              <span class="tool-btn" @click="scale = 100">实际大小</span>
            </div>
          </a-space>
        </a-layout-footer>
      </a-layout>
      <a-layout-sider
        v-model="optionCollapsed"
        class="data-view-option-panel"
        width="400"
        collapsed-width="0"
        :reverse-arrow="true"
        :collapsible="true"
      >
        <div v-if="currentItem === null" class="data-view-screen-option dark-theme">
          <div class="screen-option-header">
            <span>页面设置</span>
          </div>
          <div class="screen-option-body">
            <a-form class="dark" layout="horizontal" :label-col="{span: 6}" :wrapper-col="{span: 14, offset: 1}">
              <a-form-item label="大屏标题">
                <a-input v-model="screenConfig.title" />
              </a-form-item>
              <a-form-item label="画板大小">
                <a-row :gutter="20">
                  <a-col :span="12">
                    <a-input-number v-model="screenConfig.width" :min="1" :step="10" />
                  </a-col>
                  <a-col :span="12">
                    <a-input-number v-model="screenConfig.height" :min="1" :step="10" />
                  </a-col>
                </a-row>
              </a-form-item>
              <a-form-item label="背景图">
                <a-select v-model="screenConfig.backgroundImg">
                  <a-select-option
                    v-for="image in backgroundImgList"
                    :key="image.image_path"
                    :value="image.image_path"
                  >
                    {{ image.image_name }}
                  </a-select-option>
                </a-select>
              </a-form-item>
              <a-form-item label="上传背景图">
                <upload-image
                  height="120px"
                  :image-type="'screen_background'"
                  :image-url="screenConfig.backgroundImg"
                  @uploaded="handleImageUploaded"
                />
              </a-form-item>
              <a-form-item label="吸附距离">
                <a-input-number v-model="screenConfig.diff" :min="1" :precision="0" />
              </a-form-item>
            </a-form>
          </div>
        </div>
        <chart-option v-else class="data-view-chart-option dark-theme" :data-source-list="dataSourceList" />
      </a-layout-sider>
    </a-layout>
  </a-layout>
</template>

<script>
import html2canvas from 'html2canvas'
import { message } from 'ant-design-vue'
import { mapState } from 'vuex'
import { menus } from './menu'
import defaultSettings from '@/config'

import '@/components/DataView/index.js'
import OptionConfigMap from '@/components/DataView/components/config'
import Layout from '@/components/DataView/layout/layout'
import Item from '@/components/DataView/layout/item'
import Chart from '@/components/DataView/common/chart'
import Layer from '@/components/DataView/common/layer'
import ChartOption from '@/components/DataView/common/option'
import UploadImage from '@/components/UploadImage'

import { getDataSourceList } from '@/api/data-source'
import { getImageList, saveThumbnail } from '@/api/image'
import { getDataView, saveDataView, updateDataView } from '@/api/data-view'

export default {
  name: 'Index',
  components: {
    Layout,
    Item,
    Chart,
    Layer,
    ChartOption,
    UploadImage
  },
  data() {
    return {
      menus,
      routerBase: defaultSettings.routerBase,
      layerCollapsed: false,
      optionCollapsed: false,
      loading: true,
      instanceId: null,
      isCopy: null,
      dataSourceList: [],
      backgroundImgList: [],
      instanceVersion: 1
    }
  },
  computed: {
    scale: {
      get() {
        return this.$store.state.canvasConfig.scale * 100
      },
      set(val) {
        this.$store.commit('setCanvasScale', {
          scale: val
        })
      }
    },
    refLine: {
      get() {
        return this.$store.state.canvasConfig.refLine
      },
      set(val) {
        this.$store.commit('setRefLine', val)
      }
    },
    indicatorLine: {
      get() {
        return this.$store.state.canvasConfig.indicatorLine
      },
      set(val) {
        this.$store.commit('setIndicatorLine', val)
      }
    },
    screenConfig: {
      get() {
        return this.$store.state.screenConfig
      },
      set(val) {
        this.$store.commit('setScreenConfig', val)
      }
    },
    ...mapState([
      'items',
      'canvasConfig',
      'clickItem',
      'currentItem'
    ])
  },
  watch: {},
  created() {
    // 初始化页面选项
    this.getDataSourceList()
    this.getImageList()
    const instanceId = this.$route.params.instance_id
    const isCopy = this.$route.params.is_copy
    if (instanceId) {
      this.instanceId = parseInt(instanceId)
      this.isCopy = parseInt(isCopy)
      this.getDataView(instanceId)
    } else {
      // 初始化默认缩放比例
      this.$store.commit('autoCanvasScale')
    }
  },
  methods: {
    handleDragStart(event, key, version) {
      event.dataTransfer.setData('type', `${key}_${version}`)
    },
    handleDragOver(event) {
      event.preventDefault()
    },
    handleDrop(event) {
      const type = event.dataTransfer.getData('type')
      if (!type) return
      const newItem = OptionConfigMap[type]()
      newItem.style.x = event.offsetX - newItem.style.width / 2
      newItem.style.y = event.offsetY - newItem.style.height / 2
      newItem.hover = false
      this.$store.commit('addItem', newItem)
    },
    handleItemUnClick() {
      this.$store.commit('setClickItem', false)
    },
    handleItemUnChoose(e) {
      if (!this.clickItem) {
        this.$store.commit('setCurrentItem', { item: null, index: -1 })
      }
      if (e.button !== 2) {
        this.$store.commit('hideContextmenu')
      }
    },
    handleImageUploaded() {
      this.getImageList()
    },
    async getDataSourceList() {
      const response = await getDataSourceList()
      this.dataSourceList = response.data
    },
    async getImageList() {
      const response = await getImageList({
        'imageType': 'screen_background'
      })
      this.backgroundImgList = response.data
    },
    async getDataView(instanceId) {
      try {
        const response = await getDataView(instanceId)
        // 渲染大屏数据
        this.screenConfig.title = response.data.instance_title
        this.screenConfig.width = response.data.instance_width
        this.screenConfig.height = response.data.instance_height
        this.screenConfig.backgroundImg = response.data.instance_background_img
        this.$store.commit('autoCanvasScale')
        // 渲染图表数据
        const items = response.data.chart_items
        if (items !== null && items !== undefined && items.length > 0) {
          items.map((item) => {
            item.style = {
              x: item.x,
              y: item.y,
              width: item.width,
              height: item.height,
              rotate: item.rotate
            }
            item.data = JSON.parse(item.data)
            item.chartData = JSON.parse(item.chartData)
            item.option = JSON.parse(item.option)
            return item
          })
          this.instanceVersion = response.data.instance_version
          await this.$store.dispatch('setCharts', items)
        }
      } catch (e) {
        console.log('获取大屏信息或解析大屏信息失败', e)
      }
    },
    handleBack() {
      this.$router.push('/data-view/index')
    },
    async handleSave() {
      const thumbnail = await this.screenshot()
      if (!thumbnail.success) {
        message.error('保存缩略图失败')
        return
      }

      const items = JSON.parse(JSON.stringify(this.items))
      items.map((item, index) => {
        item.index = index
        item.x = item.style.x
        item.y = item.style.y
        item.width = item.style.width
        item.height = item.style.height
        item.rotate = item.style.rotate
        item.data = JSON.stringify(item.data)
        item.chartData = JSON.stringify(item.chartData)
        item.option = JSON.stringify(item.option)
        return item
      })
      if (this.isCopy === 1) {
        // 证明是复制该模板，创建新的实例
        // 需要将ID置为null
        this.instanceId = null
      }
      const screenInstance = {
        instance_id: this.instanceId,
        instance_title: this.screenConfig.title,
        instance_width: this.screenConfig.width,
        instance_height: this.screenConfig.height,
        instance_background_img: this.screenConfig.backgroundImg,
        instance_view_thumbnail: thumbnail.data,
        instance_version: this.instanceVersion,
        chart_items: items
      }
      if (this.instanceId) {
        // 更新
        const response = await updateDataView(screenInstance)
        if (response.success) {
          message.success('更新成功')
          window.location.href = this.routerBase + 'data-view-instance/edit/' + this.instanceId + '/0'
        } else {
          message.success('更新失败, ' + response.data)
        }
      } else {
        const response = await saveDataView(screenInstance)
        if (response.success) {
          const instanceId = response.data
          message.success('保存成功')
          window.location.href = this.routerBase + 'data-view-instance/edit/' + instanceId + '/0'
        } else {
          message.success('保存失败, ' + response.data)
        }
      }
    },
    async screenshot() {
      // 清除选中和hover状态
      this.$store.commit('setCurrentItem', { item: null, index: -1 })

      const container = document.getElementById('data-view-layout')
      // 需要移除transform属性
      const { transform } = container.style
      container.style.transform = 'scale(1) translate(0px, 0px)'

      const params = {
        logger: false,
        allowTaint: true,
        useCORS: true,
        scrollX: 0,
        scrollY: 0,
        width: this.screenConfig.width,
        height: this.screenConfig.height
      }
      const canvas = await html2canvas(container, params)

      container.style.transform = transform

      const data = window.atob(canvas.toDataURL('image/png').split(',')[1])
      const ia = new Uint8Array(data.length)
      for (let i = 0; i < data.length; i++) {
        ia[i] = data.charCodeAt(i)
      }
      const formData = new FormData()
      formData.append('file', new Blob([ia], { type: 'image/png' }), `${new Date().getTime()}_thumbnail.png`)
      return saveThumbnail(formData)
    },
    handlePreview() {
      if (this.instanceId) {
        window.open(this.routerBase + 'data-view-instance/preview/' + this.instanceId)
      } else {
        message.info('请先保存图表后预览')
      }
    },
    async handleDebug() {
    }
  }
}
</script>
<style lang="less">
@import "index.less";
</style>
