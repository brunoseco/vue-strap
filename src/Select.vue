<template>
  <div ref="select" :class="classes" v-click-outside="close">
    <div ref="btn" class="form-control dropdown-toggle" tabindex="1" :disabled="disabled || !hasParent" :readonly="readonly"
      @blur="canSearch ? null : close()"
      @click="toggle()"
      @keydown.esc.stop.prevent="close"
      @keydown.space.stop.prevent="toggle"
      @keydown.enter.stop.prevent="toggle"
    >
      <span class="btn-content" v-html="loading ? text.loading : showPlaceholder || (multiple && showCount ? selectedText : selected)"></span>
      <span v-if="clearButton&&values.length" class="close" @click="clear()">&times;</span>
    </div>
    <select ref="sel" v-model="val" :name="name" class="secret" :multiple="multiple" :required="required" :readonly="readonly" :disabled="disabled">
      <option v-if="required" value=""></option>
      <option v-for="option in list" :value="option[optionsValue]">{{ option[optionsLabel] }}</option>
    </select>
    <ul class="dropdown-menu">
      <template v-if="list.length">
        <li v-if="canSearch" class="input-group padding-5 dropdown-item">
          <input type="text" :placeholder="searchText||text.search" class="form-control" autocomplete="off" ref="search"
            v-model="searchValue"
            @keyup.esc="close"
          />
        </li>
        <li class="dropdown-item" v-if="required&&!clearButton"><a @mousedown.prevent="clear() && close()">{{ placeholder || text.notSelected }}</a></li>
        <li class="dropdown-item" v-for="option in filteredOptions" :id="option[optionsValue]" @mousedown.prevent="select(option[optionsValue])">
          <a @mousedown.prevent="">
            <span v-html="option[optionsLabel]"></span>
            <span class="glyphicon glyphicon-ok check-mark" v-show="isSelected(option[optionsValue])"></span>
          </a>
        </li>
      </template>
      <slot></slot>
      <transition v-if="notify && !closeOnSelect" name="fadein"><div class="notify in">{{limitText}}</div></transition>
    </ul>
    <transition v-if="notify && closeOnSelect" name="fadein"><div class="notify out"><div>{{limitText}}</div></div></transition>
    <!-- <pre>Options: {{list}}</pre> -->
  </div>
</template>

<script>
import {translations} from './utils/utils.js'
import ClickOutside from './directives/ClickOutside.js'

var timeout = {}
export default {
  directives: {
    ClickOutside
  },
  props: {
    clearButton: {type: Boolean, default: false},
    closeOnSelect: {type: Boolean, default: false},
    disabled: {type: Boolean, default: false},
    lang: {type: String, default: navigator.language},
    limit: {type: Number, default: 1024},
    minSearch: {type: Number, default: 0},
    multiple: {type: Boolean, default: false},
    name: {type: String, default: null},
    options: {type: Array, default () { return [] }},
    optionsLabel: {type: String, default: 'label'},
    optionsValue: {type: String, default: 'value'},
    parent: {default: true},
    placeholder: {type: String, default: null},
    readonly: {type: Boolean, default: null},
    required: {type: Boolean, default: null},
    search: {type: Boolean, default: false},
    searchText: {type: String, default: null},
    countText: {type: String, default: null},
    showCount: {type: Boolean, default: false},
    url: {type: String, default: null},
    value: null
  },
  data () {
    return {
      list: [],
      loading: null,
      searchValue: null,
      show: false,
      notify: false,
      val: null,
      valid: null
    }
  },
  computed: {
    canSearch () { return this.minSearch ? this.list.length >= this.minSearch : this.search },
    classes() { return [{ open: this.show, disabled: this.disabled }, this.class, this.isLi ? 'dropdown' : this.inInput ? 'input-group-btn' : 'input-group'] },
    filteredOptions () {
      var search = (this.searchValue || '').toLowerCase()
      return !search ? this.list : this.list.filter(el => {
        return ~el[this.optionsLabel].toLowerCase().search(search)
      })
    },
    hasParent () { return this.parent instanceof Array ? this.parent.length : this.parent },
    inInput () { return this.$parent._input },
    isLi () { return this.$parent._navbar || this.$parent.menu || this.$parent._tabset },
    limitText () { return this.text.limit.replace('{{limit}}', this.limit) },
    selected () {
      if (this.list.length === 0) { return '' }
      var sel = this.values.map(val => (this.list.find(o => o[this.optionsValue] === val) || {})[this.optionsLabel]).filter(val => val !== undefined)
      this.$emit('selected', sel)
      return sel.join(', ')
    },
    selectedText () { return this.countText || this.text.selected.replace('{{count}}', this.values.length) },
    showPlaceholder () { return (this.values.length === 0 || !this.hasParent) ? (this.placeholder || this.text.notSelected) : null },
    text () { return translations(this.lang) },
    values () { return this.val instanceof Array ? this.val : ~[null, undefined].indexOf(this.val) ? [] : [this.val] },
    valOptions () { return this.list.map(el => el[this.optionsValue]) }
  },
  watch: {
    options (options) {
      if (options instanceof Array) this.setOptions(options)
    },
    show (val) {
      if (val) {
        this.$refs.search ? this.$refs.search.focus() : this.$refs.btn.focus()
        // onBlur(this.$refs.select, e => { this.show = false })
      } else {
        // offBlur(this.$refs.select)
      }
    },
    url () {
      this.urlChanged()
    },
    valid (val, old) {
      this.$emit('isvalid', val)
      this.$emit(!val ? 'invalid' : 'valid')
      if (val !== old && this._parent) this._parent.validate()
    },
    value (val, old) {
      if (val !== old) { this.val = val }
    },
    val (val, old) {
      if (val === undefined) { this.val = val = null }
      if (val !== old) {
        this.$emit('change', val)
        this.$emit('input', val)
      }
      if (val instanceof Array && val.length > this.limit) {
        this.val = val.slice(0, this.limit)
        this.notify = true
        if (timeout.limit) clearTimeout(timeout.limit)
        timeout.limit = setTimeout(() => {
          timeout.limit = false
          this.notify = false
        }, 1500)
      }
      this.valid = this.validate()
    }
  },
  methods: {
    close () {
      this.show = false
    },
    checkData () {
      if (this.multiple) {
        if (this.limit < 1) { this.limit = 1 }
        if (!(this.val instanceof Array)) {
          this.val = (this.val === null || this.val === undefined) ? [] : [this.val]
        }
        var values = this.valOptions
        this.val = this.val.filter(el => ~values.indexOf(el))
        if (this.values.length > this.limit) {
          this.val = this.val.slice(0, this.limit)
        }
      } else {
        if (!~this.valOptions.indexOf(this.val)) { this.val = null }
      }
    },
    clear () {
      if (this.disabled || this.readonly) { return }
      this.val = this.val instanceof Array ? [] : null
      this.toggle()
    },
    clearSearch () {
      this.searchValue = ''
      this.$refs.search.focus()
    },
    isSelected (v) {
      return this.values.indexOf(v) > -1
    },
    select (v) {
      if (this.val instanceof Array) {
        var newVal = this.val.slice(0)
        if (~newVal.indexOf(v)) {
          newVal.splice(newVal.indexOf(v), 1)
        } else {
          newVal.push(v)
        }
        this.val = newVal
        if (this.closeOnSelect) {
          this.toggle()
        }
      } else {
        this.val = v
        this.toggle()
      }
    },
    setOptions (options) {
      this.list = options.map(el => {
        if (el instanceof Object) { return el }
        let obj = {}
        obj[this.optionsLabel] = el
        obj[this.optionsValue] = el
        return obj
      })
      this.$emit('options', this.list)
    },
    toggle () {
      if (this.disabled && !this.show) return;
      this.show = !this.show
      if (!this.show) this.$refs.btn.focus()
    },
    urlChanged () {
      if (!this.url || !this.$http) { return }
      this.loading = true
      this.$http.get(this.url).then(response => {
        var data = response.data instanceof Array ? response.data : []
        try { data = JSON.parse(data) } catch (e) {}
        this.setOptions(data)
        this.loading = false
        this.checkData()
      }, response => {
        this.loading = false
      })
    },
    validate () {
      return !this.required ? true : this.val instanceof Array ? this.val.length > 0 : this.val !== null
    }
  },
  created () {
    this.setOptions(this.options)
    this.val = this.value
    this._select = true
    if (this.val === undefined || !this.parent) { this.val = null }
    if (!this.multiple && this.val instanceof Array) {
      this.val = this.val[0]
    }
    this.checkData()
    if (this.url) this.urlChanged()
    let parent = this.$parent
    while (parent && !parent._formValidator) { parent = parent.$parent }
    if (parent && parent._formValidator) {
      parent.children.push(this)
      this._parent = parent
    }
  },
  mounted () {
    if (this._parent) this._parent.children.push(this)
    this.setOptions(this.options)
    this.val = this.value
    this.checkData()
  },
  beforeDestroy () {
    if (this._parent) {
      var index = this._parent.children.indexOf(this)
      this._parent.children.splice(index, 1)
    }
  }
}
</script>

<style scoped>
.padding-5 {
  padding: 5px;
}
.form-control.dropdown-toggle{
  height: auto !important;
  padding-right: 24px !important;
}
.form-control.dropdown-toggle:after{
  content: ' ' !important;
  position: absolute !important;
  right: 13px !important;
  top: 50% !important;
  margin: -1px 0 0 !important;
  border-top: 4px dashed !important;
  border-top: 4px solid \9 !important;
  border-right: 4px solid transparent !important;
  border-left: 4px solid transparent !important;
}
.bs-searchbox {
  position: relative !important;
  margin: 4px 8px !important;
}
.bs-searchbox .close {
  position: absolute !important;
  top: 0 !important;
  right: 0 !important;
  z-index: 2 !important;
  display: block !important;
  width: 34px !important;
  height: 34px !important;
  line-height: 34px !important;
  text-align: center !important;
}
.bs-searchbox input:focus,
.form-control.dropdown-toggle:focus {
  outline: 0 !important;
  border-color: #66afe9 !important;
  box-shadow: inset 0 1px 1px rgba(0,0,0,.075),0 0 8px rgba(102,175,233,.6) !important;
}
.secret {
  border: 0 !important;
  clip: rect(0 0 0 0) !important;
  height: 1px !important;
  margin: -1px !important;
  overflow: hidden !important;
  padding: 0 !important;
  position: absolute !important;
  width: 1px !important;
}
.form-control.dropdown-toggle>.close { margin-left: 5px !important;}
.notify.out { position: relative !important; }
.notify.in,
.notify>div {
  position: absolute !important;
  width: 96% !important;
  margin: 0 2% !important;
  min-height: 26px !important;
  padding: 3px 5px !important;
  background: #f5f5f5 !important;
  border: 1px solid #e3e3e3 !important;
  box-shadow: inset 0 1px 1px rgba(0,0,0,.05) !important;
  pointer-events: none !important;
}
.notify>div {
  top: 5px !important;
  z-index: 1 !important;
}
.notify.in {
  opacity: .9 !important;
  bottom: 5px !important;
}
.btn-group-justified .dropdown-toggle>span:not(.close) {
  width: calc(100% - 18px) !important;
  display: inline-block !important;
  overflow: hidden !important;
  white-space: nowrap !important;
  text-overflow: ellipsis !important;
  margin-bottom: -4px !important;
}
.btn-group-justified .dropdown-menu { width: 100% !important; }
</style>
