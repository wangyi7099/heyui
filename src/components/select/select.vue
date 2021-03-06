<template>
  <div :class="selectCls">
    <div :class="showCls">
      <template v-if="multiple">
        <div class="h-select-multiple-tags">
          <span v-for="obj of objects" :key="obj[key]">
            <span>{{obj[title]}}</span><i class="h-icon-close" @click.stop="setvalue(obj)" v-if="!disabled"></i>
          </span>
          <input v-if="filterable"
                type="text"
                class="h-select-search-input" v-model="searchInput"
               @keyup="handle"
               @blur="blurHandle"
               @keypress.enter="enterHandle"
                :placeholder="showSearchPlaceHolder" />
        </div>
        <div v-if="codes.length==0&&!filterable" class="h-select-placeholder">{{showPlaceholder}}</div>
      </template>
      <template v-else-if="!multiple">
        <input v-if="filterable"
               type="text"
               @keyup="handle"
               @blur="blurHandle"
               @keypress.enter="enterHandle"
               class="h-select-search-input h-select-single-search-input" v-model="searchInput"
               :placeholder="codes&&objects?'':showSearchPlaceHolder" />
        <div class="h-select-value-single" v-if="codes&&objects" v-show="!searchInput">{{objects[title]}}</div>
        <div v-else-if="!filterable" class="h-select-placeholder">{{showPlaceholder}}</div>
      </template>
      <i class="h-icon-down"></i>
    </div>
    <div :class="groupCls">
      <div class="h-select-group-container" v-if="isShow">
        <!-- <Search v-if="filterable" class="h-select-search-input" :placeholder="showSearchPlaceHolder" trigger-type="input" @onsearch="search" position="front"></Search> -->
        <div class="h-select-list">
          <ul class="h-select-ul">
            <template v-for="(option, index) of filterOptions">
            <li v-if="!option.hidden"
                :key="option[key]"
                @click="setvalue(option)"
                :class="getLiCls(option, index)">
              <div v-if="!!render"
                  v-html="option[html]"></div>
              <template v-else-if="!$scopedSlots.item">{{option[title]}}</template>
              <slot v-else :item="option" name="item"></slot>
              <i v-if="multiple"
                class="h-icon-check"></i>
            </li>
            </template>
            <li v-if="filterOptions.length==0" class="h-select-ul-empty">{{emptyContent}}</li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import config from '../../utils/config';
import utils from '../../utils/utils';
import Dropdown from '../../plugins/dropdown';
import Locale from '../../mixins/locale';

const prefix = 'h-select';

export default {
  name: 'hSelect',
  mixins: [ Locale ],
  props: {
    readonly: {
      type: Boolean,
      default: false
    },
    multiple: {
      type: Boolean,
      default: false
    },
    datas: [Array, Object],
    type: {
      type: [String],
      default: 'key'  //object
    },
    disabled: {
      type: Boolean,
      default: false
    },
    dict: String,
    limit: {
      type: Number
    },
    nullOption: {
      type: Boolean,
      default: true
    },
    nullOptionText: {
      type: String,
      // default: "请选择"
    },
    noBorder: {
      type: Boolean,
      default: false
    },
    placeholder: {
      type: String,
      // default: "请选择"
    },
    searchPlaceHolder: {
      type: String,
      // default: "请输入筛选文本"
    },
    emptyContent: {
      type: String,
      default: "未搜索到相关数据"
    },
    filterable: {
      type: Boolean,
      default: false
    },
    autosize: {
      type: Boolean,
      default: false
    },
    equalWidth: {
      type: Boolean,
      default: true
    },
    keyName: {
      type: String,
      default: () => config.getOption('dict', 'keyName')
    },
    titleName: {
      type: String,
      default: () => config.getOption('dict', 'titleName')
    },
    render: Function,
    value: [Number, String, Array, Object],
    className: String,
  },
  data() {
    return {
      key: this.keyName,
      title: this.titleName,
      html: "select_rander_html",
      codes: [],
      objects: {},
      hasNullOption: this.nullOption && !this.multiple,
      searchInput: '',
      nowSelected: -1,
      isShow: false,
      content: null
    };
  },
  watch: {
    datas() {
      this.parse();
    },
    value() {
      this.parse();
    },
    disabled() {
      if (this.disabled) {
        this.dropdown.disabled();
      } else {
        this.dropdown.enabled();
      }
    },
    searchInput() {
      this.nowSelected = -1;
    },
    nowSelected() {
      this.$nextTick(() => {
        if (this.content && this.nowSelected > -1) {
          let dom = this.content.querySelector('.h-select-item-picked')
          let uldom = this.content.querySelector('.h-select-list')
          if (dom && uldom) {
            if (
              dom.offsetTop + dom.offsetHeight - uldom.scrollTop >
              uldom.offsetHeight
            ) {
              uldom.scrollTop =
                dom.offsetTop + dom.offsetHeight - uldom.offsetHeight
            } else if (dom.offsetTop - uldom.scrollTop < 0) {
              uldom.scrollTop = dom.offsetTop
            }
          }
        }
      })
    }
  },
  beforeMount() {
    this.parse();
  },
  beforeDestroy() {
    let el = this.el;
    if(el) {
      el.style.display = 'none';
      this.$el.appendChild(el);
    }
    if(this.dropdown) {
      this.dropdown.destory();
    }
  },
  mounted() {
    this.$nextTick(() => {
      let el = this.el = this.$el.querySelector('.h-select-show');
      let content = this.content = this.$el.querySelector('.h-select-group');
      let that = this;
      this.dropdown = new Dropdown(el, {
        content,
        disabled: this.disabled,
        equalWidth: this.equalWidth,
        trigger: 'click foucs',
        triggerOnce: this.filterable,
        events: {
          show(){
            that.isShow = true;
          }
        }
      });
    });
  },
  methods: {
    handle(event) {
      let code = event.keyCode || event.which || event.charCode
      if (code == 38) {
        if (this.nowSelected > 0) {
          this.nowSelected -= 1
        }
      } else if (code == 40) {
        if (this.nowSelected < this.filterOptions.length - 1) {
          this.nowSelected += 1
        }
      }
    },
    enterHandle(event) {
      event.preventDefault()
      if (this.nowSelected >= 0) {
        this.setvalue(this.filterOptions[this.nowSelected], 'enter');
        if (!this.multiple) {
          event.target.blur();
        }
      }
    },
    blurHandle(event) {
      this.nowSelected = -1;
      setTimeout(() => {
        this.searchInput = '';
      }, 300);
    },
    search(value) {
      this.searchInput = value;
    },
    setObjects() {
      if (this.multiple) {
        let os = [];
        for (let code of this.codes) {
          if(this.optionsMap[code] == null) {
            continue;
          }
          os.push(this.optionsMap[code]);
        }
        this.objects = os;
      } else {
        this.objects = this.optionsMap[this.codes];
      }
    },
    parse() {
      if (this.multiple) {
        let values = this.value || [];
        this.codes = values.map((item) => {
          return this.type == 'key' ? this.getValue(item) : item[this.key];
        }).filter(item=>item!==null)
      } else {
        if (this.type == 'key') {
          this.codes = this.getValue(this.value);
        } else {
          if (utils.isObject(this.value)) {
            this.codes = this.value[this.key];
          } else {
            this.codes = null;
          }
        }
      }
      this.setObjects();
    },
    getValue(value) {
      return utils.isNull(value) ? null : value;
    },
    setvalue(option, trigger) {
      if (this.readonly) return;
      if (option.disabled || option.isLabel) return;
      let code = option[this.key];
      if (this.multiple) {
        if (!utils.isNull(this.limit) && !this.isIncludes(code) && this.codes.length >= this.limit) {
          this.$Message.error(`您最多可以选${this.limit}个选项`);
          return;
        }
        this.codes = utils.toggleValue(this.codes, code);
      } else {
        this.codes = code;
      }
      this.setObjects();
      let value = this.type == 'key' ? this.codes : this.objects;
      this.$emit('input', value);
      let event = document.createEvent("CustomEvent");
      event.initCustomEvent("setvalue", true, true, this.objects);
      this.$el.dispatchEvent(event);
      this.nowSelected = -1;
      if (this.multiple) {
        this.searchInput = '';
        this.$nextTick(()=>{
          this.dropdown.update();
        })
      } else {
        this.dropdown.hide();
        setTimeout(() => {
          this.searchInput = '';
        }, 100);
      }
    },
    isIncludes(code){
      return this.codes.some(item=>item == code);
    },
    getLiCls(option, index) {
      let code = option[this.key];
      if (option.isLabel){
        return {
          [`${prefix}-item-label`]: option.isLabel,
        };
      } else {
        return {
          [`${prefix}-item-disabled`]: option.disabled,
          [`${prefix}-item`]: true,
          [`${prefix}-item-selected`]: (this.multiple ? this.isIncludes(code) : this.codes == code),
          [`${prefix}-item-picked`]: (this.nowSelected == index),
        };
      }
    }
  },
  filters: {
    showText(key, value) {
      return value.includes(key);
    }
  },
  computed: {
    hasLabel() {
      return this.options.some(item => item.isLabel);
    },
    showNullOptionText() {
      return this.nullOptionText || this.t('h.select.nullOptionText');
    },
    showPlaceholder() {
      return this.placeholder || this.t('h.select.placeholder');
    },
    showSearchPlaceHolder() {
      return this.searchPlaceHolder || this.t('h.select.searchPlaceHolder');
    },
    selectCls() {
      let autosize = this.autosize || !!this.noBorder;
      return {
        [`${prefix}`]: true,
        [`${prefix}-input-border`]: !this.noBorder,
        [`${prefix}-input-no-border`]: this.noBorder,
        [`${prefix}-multiple`]: this.multiple,
        [`${prefix}-no-autosize`]: !autosize,
        [`${prefix}-disabled`]: this.disabled,
      }
    },
    showCls() {
      return {
        [`${prefix}-show`]: true,
        [`${this.className}-show`]: !!this.className,
      }
    },
    groupCls() {
      return {
        [`${prefix}-group`]: true,
        [`${prefix}-group-has-label`]: this.hasLabel,
        [`${prefix}-multiple`]: this.multiple,
        [`${prefix}-single`]: !this.multiple,
        [`${this.className}-dropdown`]: !!this.className
      }
    },
    optionsMap() {
      let optionsMap = utils.toObject(this.options, this.key);
      delete optionsMap.null;
      return optionsMap;
    },
    filterOptions() {
      if (this.searchInput) {
        if (this.dropdown) this.dropdown.update();
        let searchValue = this.searchInput.toLocaleLowerCase();
        return this.options.filter((item) => {
          return (item[this.html] || item[this.title]).toLocaleLowerCase().indexOf(searchValue) != -1;
        });
      }
      return this.options;
    },
    options() {
      if (!this.datas && !this.dict) {
        log.error('Select组件:datas或者dict参数最起码需要定义其中之一');
        return [];
      }
      let datas = this.datas;
      if (this.dict) {
        datas = config.getDict(this.dict);
      }
      datas = utils.initOptions(datas, this);
      if (!this.multiple && this.hasNullOption) {
        datas.unshift({
          [`${this.key}`]: null,
          [`${this.title}`]: this.showNullOptionText,
          [`${this.html}`]: this.showNullOptionText
        });
      }
      return datas;
    }
  }
};
</script>