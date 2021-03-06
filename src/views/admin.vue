<template>
  <div class="dv-admin"
       @click.stop.prevent="hideContextMenu">
    <board>
      <!--头部嵌套可拖拽物品-->
      <template v-slot:headerBox>
        <drag-list :drag-list="navigate"></drag-list>
      </template>
      <template v-slot:coverage>
        <div class="list-item" :key="transform.id" v-for="transform in coverageMaps"
             :class="[{'hovered':hoverItem===transform.id},{'selected':currentSelected&&currentSelected.id===transform.id},]"
             :selected="currentSelected&&currentSelected.id===transform.id"
             @click.stop.prevent="handleSelected(transform)"
             @mouseenter="handleHover(transform)"
             @mouseleave="handleNoHover()">
          <b-icon v-if="transform.packageJson.icon" :name="transform.packageJson.icon"></b-icon>
          <!---->
          <span v-if="transform.packageJson.config&&transform.packageJson.config.title.content">
            {{ transform.packageJson.config.title.content}}
          </span>
          <span v-else> {{ transform.packageJson.title }}</span>
        </div>
      </template>
      <template v-slot:canvas>
        <!--动态组件-->
        <template v-for="transform in canvasMap">
          <drag-item :key="transform.id" :item="transform"
                     :com-hover="hoverItem===transform.id"
                     :selected="currentSelected&&currentSelected.id===transform.id"
                     @click.native.stop.prevent="handleSelected(transform)"
                     @contextmenu.native.stop.prevent="handleRightClickOnCanvas(transform,$event)"
                     @mouseenter.native="handleHover(transform)"
                     @mouseleave.native="handleNoHover()">
            <charts-factory :type-name="transform.packageJson.name"
                            :config="transform.packageJson.config"
                            :api-data="transform.packageJson.api_data"
                            :apis="transform.packageJson.apis"></charts-factory>
          </drag-item>
        </template>
      </template>
    </board>
    <b-modal v-model="deleteDialog" :styles="{top: '300px',width:'350px'}"
             class-name="delete-dialog" @on-ok="deleteOne">
      <div class="delete-dialog-inner">
        <div>
          <b-icon name="ios-warning" size="40"></b-icon>
        </div>
        <p>是否删除选中的1个组件</p>
      </div>
    </b-modal>
  </div>
</template>

<script>
  import Board from '../components/board/index'
  import navigateList from '../config/navigate'
  import DragList from '../components/drag/DragList'
  import DragItem from '../components/drag/DragItem'
  import { mapGetters } from 'vuex'
  import { on, off } from 'bin-ui/src/utils/dom'
  import { getCanvasMaps } from '../api/canvasMaps/canvas-maps-request'
  import { getPageSettings } from '../api/app/app-request'
  import ChartsFactory from '../components/charts/charts-factory'

  export default {
    name: 'Admin',
    data () {
      return {
        navigate: navigateList,
        hoverItem: null,
        deleteDialog: false
      }
    },
    computed: {
      ...mapGetters(['canvasMap', 'currentSelected']),
      coverageMaps () {
        let maps = [...this.canvasMap]
        return maps.reverse()
      }
    },
    created () {
      // 拉取页面配置信息
      getPageSettings().then(res => {
        this.$store.dispatch('SetPageSettings', res.data)
      })
      // 拉取页面canvasMaps
      getCanvasMaps().then(res => {
        this.$store.dispatch('InitCanvasMaps', res.data)
        this.$log.danger('========>canvasMaps')
        this.$print(res.data)
      })
    },
    mounted () {
      on(document, 'keyup', this.handleKeyup)
      this.$EventBus.$on('context/menu/delete', this.deleteDialogShow)
    },
    methods: {
      // 悬停事件
      handleHover (item) {
        this.hoverItem = item.id
      },
      handleNoHover () {
        this.hoverItem = null
      },
      // transform点击事件
      handleSelected (item) {
        this.$store.dispatch('SingleSelected', item)
        this.$store.dispatch('ToggleContextMenu')
      },
      // transform点击事件右键点击
      handleRightClickOnCanvas (item, event) {
        let info = { x: event.pageX + 10, y: event.pageY + 10 }
        this.$store.dispatch('ToggleContextMenu', info)
        this.$store.dispatch('SingleSelected', item)
      },
      // 外层区域关闭右键菜单
      hideContextMenu () {
        this.$store.dispatch('ToggleContextMenu')
      },
      handleKeyup (event) {
        let e = event || window.event
        let k = e.keyCode || e.which
        if (k === 46) {
          if (this.currentSelected) {
            this.deleteDialogShow()
          }
        }
      },
      deleteDialogShow () {
        this.deleteDialog = true
      },
      deleteOne () {
        this.$store.dispatch('ContextMenuCommand', 'remove')
      }
    },
    components: { ChartsFactory, DragItem, DragList, Board },
    beforeDestroy () {
      off(document, 'keyup', this.handleKeyup)
      this.$EventBus.$off('context/menu/delete', this.deleteDialogShow)
    }
  }
</script>
