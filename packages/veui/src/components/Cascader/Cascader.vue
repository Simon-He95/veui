<template>
<div
  ref="root"
  :class="$c('cascader')"
  v-bind="rootAttrs"
  @keydown="handleCascaderKeydown"
>
  <slot name="trigger" v-bind="slotProps">
    <veui-select-trigger
      ref="trigger"
      :class="$c('cascader-trigger')"
      :selected="backfillOption"
      :expanded="isExpanded"
      :placeholder="realPlaceholder"
      :input-value.sync="keyword"
      :clearable="clearable"
      :searchable="searchable"
      :multiple="multiple"
      :max="max"
      :disabled="realDisabled"
      :readonly="realReadonly"
      :invalid="realInvalid"
      :icons="icons"
      :ui-parts="uiParts"
      @input="$emit('input', $event)"
      @blur="handleTriggerBlur"
      @toggle="handleTriggerToggle"
      @remove="handleTriggerRemove"
      @clear="handleTriggerClear"
    >
      <!-- v-for 来透传 slots 比 v-if 干净 -->
      <template
        v-for="name in getTriggerSlots()"
        :slot="name"
        slot-scope="props"
      >
        <slot :name="name" v-bind="props">
          {{
            realCompleteDisplay && name === 'selected'
              ? backfillOption.chains.map((i) => i.label).join(' > ')
              : null
          }}
        </slot>
      </template>
    </veui-select-trigger>
  </slot>
  <veui-overlay
    ref="overlay"
    :open="isExpanded"
    target="trigger"
    :overlay-class="realOverlayClass"
    :overlay-style="overlayStyle"
    :options="realOverlayOptions"
    :match-width="inline || !!keyword"
    :autofocus="!searchable"
    modal
    @afterclose="handleAfterClose"
    @afteropen="handleAfterOpen"
  >
    <div
      :id="dropdownId"
      ref="box"
      v-outside="{
        delay: 100,
        refs: outsideRefs,
        handler: () => updateExpanded(false)
      }"
      :class="{
        [$c('cascader-pane-wrap')]: true,
        [$c('cascader-select-all')]: realShowSelectAll
      }"
      tabindex="-1"
      :role="searchable ? 'listbox' : 'menu'"
      :aria-expanded="isExpanded"
      :ui="realUi"
      @mouseleave="handleMouseLeave"
    >
      <div
        v-if="$scopedSlots.before && !keyword"
        :class="$c('cascader-before')"
      >
        <slot name="before"/>
      </div>
      <div
        v-if="keyword && !filteredOptions.length"
        :class="$c('cascader-pane-wrap-no-data')"
      >
        <slot name="search-no-data">{{ t('noData') }}</slot>
      </div>
      <slot v-else name="pane" v-bind="slotProps">
        <veui-cascader-pane
          ref="pane"
          :class="{
            [$c('cascader-search-result')]: !!keyword,
            [$c('cascader-pane-multiple')]: multiple
          }"
          :inline="inline"
          :multiple="multiple"
          :options="keyword ? filteredOptions : realOptions"
          :column-trigger="columnTrigger"
          :value="realValue"
          :select-mode="selectMode"
          :column-width="keyword ? null : columnWidth"
          :expanded="realExpanded"
          :load="load && !keyword ? loadAndSaveData : null"
          @update:expanded="handlePaneUpdateExpanded"
          @select="handlePaneSelect"
          @keydown.native="!searchable && handleCascaderKeydown($event)"
        >
          <template slot="option-label" slot-scope="{ option }">
            <veui-search-result
              v-if="keyword"
              :matches="option.matches"
              :separator="icons.separator"
              :separator-class="$c('cascader-search-result-item-separator')"
            />
          </template>
          <template
            v-for="name in getPaneSlots()"
            :slot="name"
            slot-scope="props"
          >
            <slot :name="name" v-bind="props"/>
          </template>
        </veui-cascader-pane>
      </slot>
      <div
        v-if="$scopedSlots.after && !keyword"
        :class="$c('cascader-after')"
      >
        <slot name="after"/>
      </div>
    </div>
  </veui-overlay>
</div>
</template>

<script>
import Overlay from '../Overlay'
import CascaderPane from './CascaderPane'
import config from '../../managers/config'
import useConfig from '../../mixins/config'
import dropdown from '../../mixins/dropdown'
import useControllable from '../../mixins/controllable'
import useSearchable, { getDefaultFilter } from '../../mixins/searchable'
import prefix from '../../mixins/prefix'
import i18n from '../../mixins/i18n'
import useTree from '../../mixins/tree'
import input from '../../mixins/input'
import { useKeySelect } from '../../mixins/key-select'
import {
  mapDatasource,
  find,
  findParents,
  hasChildren
} from '../../utils/datasource'
import ui from '../../mixins/ui'
import SelectTrigger from '../_SelectTrigger'
import SearchResult from '../_SearchResult'
import { contains, scrollIntoView } from '../../utils/dom'
import '../../common/global'

config.defaults(
  {
    placeholder: '@@cascader.placeholder'
  },
  'cascader'
)

const PANE_SLOTS = ['column-before', 'column-after', 'option', 'option-label']

const TRIGGER_SLOTS = ['label', 'selected']

export default {
  name: 'veui-cascader',
  components: {
    'veui-cascader-pane': CascaderPane,
    'veui-search-result': SearchResult,
    'veui-overlay': Overlay,
    'veui-select-trigger': SelectTrigger
  },
  mixins: [
    prefix,
    ui,
    i18n,
    dropdown(),
    input,
    useConfig('config', 'cascader'),
    useTree({ childrenKey: 'options', defaultMerge: 'downwards' }),
    useKeySelect({ expandedKey: 'realExpanded' }),
    useControllable({
      prop: 'value',
      event: 'select',
      get (val) {
        return val != null ? [].concat(val) : []
      }
    }),
    useSearchable({
      datasourceKey: 'searchableOptions',
      childrenKey: 'options',
      keywordKey: 'keyword',
      filterKey: 'filterParents',
      resultKey: 'filteredOptions',
      exposeProps: true
    })
  ],
  model: {
    event: 'select'
  },
  props: {
    /* eslint-disable vue/require-prop-types */
    value: {},
    /* eslint-ensable vue/require-prop-types */
    placeholder: String,
    clearable: Boolean,
    searchable: Boolean,
    options: Array,
    multiple: Boolean,
    expanded: {}, // 覆盖 dropdown 的定义，这里 expanded 不仅仅是 boolean
    columnTrigger: {
      type: String,
      default: 'click',
      validator (val) {
        return ['hover', 'click'].indexOf(val) >= 0
      }
    },
    selectMode: {
      type: String,
      default: 'any',
      validator (val) {
        return ['leaf-only', 'any'].indexOf(val) >= 0
      }
    },
    columnWidth: [Number, String],
    showSelectAll: Boolean,
    valueDisplay: {
      type: String,
      default: 'simple',
      validator (val) {
        return ['complete', 'simple'].indexOf(val) >= 0
      }
    },
    load: Function,
    inline: Boolean,
    max: Number
  },
  data () {
    return {
      outsideRefs: ['root'],
      keyword: '',
      loadedData: Object.create(null)
    }
  },
  computed: {
    isExpanded () {
      return this.realExpanded != null && this.realExpanded !== false
    },
    realOverlayClass () {
      return this.mergeOverlayClass(this.$c('cascader-overlay'))
    },
    realOptions () {
      let options = mapDatasource(
        this.options,
        {
          enter: (item, { replace }) => {
            const key = getKey(item)
            // keep walking on this.loadedData[key]
            if (item.lazy && !item.options && key in this.loadedData) {
              replace({ ...item, options: this.loadedData[key], _loaded: true })
            }
          },
          exit: (item, { parentIndices, index, parents, childrenResult }) => {
            const parent = parents[parents.length - 1]
            let key = getKey(item)
            const inheritDisabled = !item.disabled && parent && parent.disabled
            const containLazy =
              childrenResult &&
              childrenResult.some(
                ({ lazy, containLazy, options }) =>
                  (lazy && options == null) || containLazy
              )

            if (key == null || inheritDisabled || containLazy) {
              item = { ...item }

              // 没有 name/value 直接用 ${label}${depth} 来生成 name 更稳定一点，反正 datasource 变化 uniqueId 也是在变化
              if (key == null) {
                item.name = `veui-${item.label}-${parentIndices
                  .concat(index)
                  .join('-')}`
              }

              if (inheritDisabled) {
                item.disabled = true
              }

              // TODO 将更多功能放到 cascaderPane 上而非 cascader。
              if (containLazy) {
                item.containLazy = true
              }
            }
            return item
          }
        },
        'options'
      )
      options = this.realShowSelectAll
        ? [{ label: this.t('selectAll'), position: 'inline', options }]
        : options
      return this.markChecked(options, this.realValue)
    },
    rootAttrs () {
      return {
        ui: this.realUi,
        'aria-owns': this.dropdownId,
        'aria-readonly': this.realReadonly,
        'aria-expanded': this.realExpanded,
        'aria-disabled': this.realDisabled,
        'aria-haspopup': this.searchable ? 'listbox' : 'menu',
        tabindex:
          (this.searchable && this.multiple) || this.realDisabled ? null : '0'
      }
    },
    slotProps () {
      return {
        selected: this.backfillOption, // items
        value: this.realValue, // value
        options: this.realOptions,
        filteredOptions: this.keyword ? this.filteredOptions : [],
        expanded: this.isExpanded,
        placeholder: this.realPlaceholder,
        keyword: this.keyword,
        remove: this.handleTriggerRemove,
        clear: this.handleTriggerClear,
        toggle: this.handleTriggerToggle,
        updateKeyword: (val) => (this.keyword = val),
        select: this.handlePaneSelect
      }
    },
    searchableOptions () {
      return this.realShowSelectAll
        ? this.realOptions[0].options
        : this.realOptions
    },
    realShowSelectAll () {
      return this.multiple && this.showSelectAll
    },
    realSelectLeaves () {
      return !this.multiple && this.selectMode === 'leaf-only'
    },
    realPlaceholder () {
      return this.placeholder == null
        ? this.config['cascader.placeholder']
        : this.placeholder
    },
    selectedOptions () {
      let options = this.realValue
        .map((val) =>
          find(
            this.realOptions,
            ({ value }) => value !== null && value === val,
            'options'
          )
        )
        .filter(Boolean)
      return this.multiple ? options : options[0]
    },
    selectedWithParents () {
      if (!this.selectedOptions) {
        return
      }
      let chains = this.getOptionWithParents(
        this.selectedOptions.value,
        'value'
      )
      return chains
        ? {
          ...chains[chains.length - 1],
          chains
        }
        : null
    },
    expandedWithParents () {
      if (this.realExpanded != null && typeof this.realExpanded !== 'boolean') {
        let chains = this.getOptionWithParents(this.realExpanded)
        return chains
          ? {
            ...chains[chains.length - 1],
            chains
          }
          : null
      }
      return null
    },
    realCompleteDisplay () {
      return !this.multiple && this.valueDisplay === 'complete'
    },
    backfillOption () {
      if (this.multiple) {
        return this.selectedOptions
      }
      return this.realCompleteDisplay
        ? this.selectedWithParents
        : this.selectedOptions
    }
  },
  methods: {
    loadAndSaveData (context) {
      return Promise.resolve(this.load(context)).then((loaded) => {
        this.$set(this.loadedData, getKey(context.parent), loaded)
        return loaded
      })
    },
    handleCascaderKeydown (e) {
      let passive = true
      let elem = null
      switch (e.key) {
        case 'Up':
        case 'ArrowUp':
        case 'Down':
        case 'ArrowDown':
          if (!this.realExpanded) {
            this.expandSelected()
          }
          passive = false
          if (this.searchable) {
            this.focusTrigger()
          }
          this.$nextTick(() => {
            this.handleKeydown(e)
            let elem = this.getCurrentActiveElement()
            if (elem) {
              scrollIntoView(elem)
            }
          })
          break
        case 'Esc':
        case 'Escape':
          passive = false
          elem = this.getCurrentActiveElement()
          if (elem) {
            let level = Number(elem.dataset.kbdLevel)
            if (level > 1) {
              let parents = this.expandedWithParents.chains.slice(0, -1)
              let key = getKey(parents[parents.length - 1] || {})
              this.navigateToLevel(`[data-kbd-level="${level - 1}"]`, {
                lock: true,
                targetSelector: `[data-kbd-key="${this.realExpanded}-expandable"]`
              })
              this.commit('expanded', key || true)
              // expand 切换
            } else {
              this.commit('expanded', false)
            }
          }
          if (this.searchable) {
            this.focusTrigger()
          }
          break
        case 'Tab':
          if (this.realExpanded) {
            this.navigate()
          } else {
            this.commit('expanded', false)
          }
          break
        case 'Enter':
          if (!this.realExpanded) {
            if (this.searchable) {
              this.focusTrigger()
            }
            this.expandSelected()
            break
          }

          elem = this.getCurrentActiveElement()
          if (elem) {
            elem.click()
            let hasNext = elem.hasAttribute('data-kbd-next')
            if (hasNext) {
              let level = Number(elem.dataset.kbdLevel) + 1
              this.$nextTick(() =>
                this.navigateToLevel(`[data-kbd-level="${level}"]`, {
                  lock: true
                })
              )
            }
          }
          break
        case 'Backspace': {
          let deletable =
            this.multiple &&
            this.selectedOptions.length &&
            this.searchable &&
            !this.keyword
          if (deletable) {
            let selected = this.selectedOptions
            for (let i = selected.length - 1; i >= 0; i--) {
              let option = selected[i]
              if (!option.disabled) {
                this.toggleOption(option, 'uncheck')
                break
              }
            }
          }
          break
        }
        default:
          break
      }
      if (!passive) {
        e.preventDefault()
        e.stopPropagation()
      }
    },
    handleTriggerToggle (expand) {
      if (typeof expand === 'undefined') {
        expand = !this.realExpanded
      }

      if (expand) {
        this.expandSelected()
      } else {
        this.updateExpanded(false)
      }
    },
    handleTriggerBlur (e) {
      let keepFocus = this.searchable && !this.keyword
      // 这里会过多的将焦点抢到 trigger 上，如果box中操作会导致关闭行为，会在关闭时将焦点从 trigger 上 blur
      if (
        keepFocus &&
        this.realExpanded &&
        contains(this.$refs.box, e.relatedTarget, true)
      ) {
        this.focusTrigger()
      }
    },
    handleTriggerRemove (option) {
      if (this.multiple) {
        this.toggleOption(option, 'uncheck')
      }
    },
    handleTriggerClear () {
      this.keyword = ''
      if (!this.multiple) {
        this.commit('value', null)
      } else {
        let newValues = this.clearAll(this.realValue, this.realOptions)
        this.commit('value', newValues)
      }
    },
    handleMouseLeave () {
      if (this.columnTrigger === 'hover') {
        return this.updateExpanded(!!this.realExpanded)
      }
    },
    handlePaneSelect (option) {
      // 单选
      if (!this.multiple) {
        this.commit('value', option.value)
      } else {
        this.toggleOption(option)
        this.focusTrigger()
      }
    },
    handlePaneUpdateExpanded (expanded) {
      return this.updateExpanded(expanded)
    },
    // 在只选叶子的情况下，搜索出来的结果过滤掉非叶子节点
    filterParents (current, keyword, options) {
      if (this.realSelectLeaves && hasChildren(current, 'options')) {
        return false
      }
      const defaultFilter = getDefaultFilter(this, this.filter)
      return defaultFilter(current, keyword, options)
    },
    updateExpanded (expanded) {
      this.commit('expanded', expanded)
      // 关闭时将 trigger 上的焦点 blur
      if (!expanded && this.searchable && !this.keyword) {
        this.blurTrigger()
      }
      // afterclose 会清空搜索的，这里直接清空搜索会导致下拉重绘而闪动
    },
    getPopoutParents (value) {
      let parents =
        findParents(this.realOptions, (item) => getKey(item) === value, {
          alias: 'options'
        }) || []
      return parents.filter((i) => i.position !== 'inline')
    },
    expandSelected () {
      let expanded = true
      let len = this.realValue.length
      if (len) {
        let parents = this.getPopoutParents(this.realValue[len - 1])
        if (parents && parents.length) {
          expanded = getKey(parents[parents.length - 1]) || true
        }
      }
      this.updateExpanded(expanded)
    },
    handleAfterOpen (...args) {
      if (this.realValue && this.$refs.pane) {
        this.$refs.pane.scrollToCurrent()
      }
      this.$emit('afteropen', ...args)
    },
    handleAfterClose (...args) {
      // 关闭了之后清空搜索可以避免因退出搜索而下拉闪动
      if (this.keyword) {
        this.keyword = ''
      }
      this.$emit('afterclose', ...args)
    },
    focus () {
      return this.focusTrigger()
    },
    toggleOption (option, operation = 'toggle') {
      let parents = findParents(
        this.realOptions,
        (item) => getKey(item) === getKey(option),
        { alias: 'options', includeSelf: true }
      )
      if (!parents) {
        throw new Error(`[veui] Unknown option`)
      }
      option = parents[parents.length - 1]
      parents = parents.slice(0, -1)
      let newValues = this[`${operation}Item`](
        this.realValue,
        option,
        parents,
        this.realOptions
      )
      this.commit('value', newValues)
    },
    focusTrigger () {
      let { trigger } = this.$refs
      if (trigger && typeof trigger.focus === 'function') {
        trigger.focus()
      }
    },
    blurTrigger () {
      let { trigger } = this.$refs
      if (trigger && typeof trigger.blur === 'function') {
        trigger.blur()
      }
    },
    getOptionWithParents (value, field) {
      return findParents(
        this.realOptions,
        (item) => {
          let val = field ? item[field] : getKey(item)
          return val === value
        },
        { alias: 'options', includeSelf: true }
      )
    },
    getPaneSlots () {
      return PANE_SLOTS.filter(
        (name) => !!this.$scopedSlots[name] || !!this.$slots[name]
      )
    },
    getTriggerSlots () {
      let slots = TRIGGER_SLOTS.filter(
        (name) => !!this.$scopedSlots[name] || !!this.$slots[name]
      )
      if (
        this.realCompleteDisplay &&
        this.backfillOption &&
        slots.indexOf('selected') === -1
      ) {
        // 这里为 山东 > 济南 这种方式做了 fallback，所以补充上去
        slots.push('selected')
      }
      return slots
    }
  }
}

function getKey ({ name, value }) {
  return name != null ? name : value
}
</script>
