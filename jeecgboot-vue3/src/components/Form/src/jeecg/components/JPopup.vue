<!--popup组件-->
<template>
  <div class="JPopup components-input-demo-presuffix" v-if="avalid">
    <!--输入框-->
   <a-input @click="handleOpen" v-model:value="showText" :placeholder="placeholder" v-bind="attrs" allow-clear @change="handleEmpty">
      <template #prefix>
        <Icon icon="ant-design:cluster-outlined"></Icon>
      </template>
    </a-input>
    <!-- update-begin--author:liaozhiyang---date:20240515---for：【QQYUN-9260】必填模式下会影响到弹窗内antd组件的样式 -->
    <a-form-item>
      <!--popup弹窗-->
      <JPopupOnlReportModal
        @register="regModal"
        :code="code"
        :multi="multi"
        :sorter="sorter"
        :groupId="uniqGroupId"
        :param="param"
        :showAdvancedButton="showAdvancedButton"
        :getContainer="getContainer"
        :getFormValues="getFormValues"
        @ok="callBack"
      ></JPopupOnlReportModal>
    </a-form-item>
    <!-- update-end--author:liaozhiyang---date:20240515---for：【QQYUN-9260】必填模式下会影响到弹窗内antd组件的样式 -->
  </div>
</template>
<script lang="ts">
  import JPopupOnlReportModal from './modal/JPopupOnlReportModal.vue';
  import { defineComponent, ref, reactive, onMounted, watchEffect, watch, computed, unref } from 'vue';
  import { useModal } from '/@/components/Modal';
  import { propTypes } from '/@/utils/propTypes';
  import { useAttrs } from '/@/hooks/core/useAttrs';
  import { useMessage } from '/@/hooks/web/useMessage';

  export default defineComponent({
    name: 'JPopup',
    components: {
      JPopupOnlReportModal,
    },
    inheritAttrs: false,
    props: {
      code: propTypes.string.def(''),
      value: propTypes.string.def(''),
      sorter: propTypes.string.def(''),
      width: propTypes.number.def(1200),
      placeholder: propTypes.string.def('请选择'),
      multi: propTypes.bool.def(false),
      param: propTypes.object.def({}),
      spliter: propTypes.string.def(','),
      groupId: propTypes.string.def(''),
      formElRef: propTypes.object,
      setFieldsValue: propTypes.func,
      getFormValues: propTypes.func,
      getContainer: propTypes.func,
      fieldConfig: {
        type: Array,
        default: () => [],
      },
      showAdvancedButton: propTypes.bool.def(true),
      // 是否是在 筛选（search） 中使用
      inSearch: propTypes.bool.def(false),
    },
    emits: ['update:value', 'register', 'popUpChange', 'focus'],
    setup(props, { emit, refs }) {
      const { createMessage } = useMessage();
      const attrs = useAttrs();
      //pop是否展示
      const avalid = ref(true);
      const showText = ref('');
      const innerShowText = ref('')
      //注册model
      const [regModal, { openModal }] = useModal();
      //表单值
      let {code, fieldConfig } = props;
      // update-begin--author:liaozhiyang---date:20230811---for：【issues/675】子表字段Popup弹框数据不更新
      //唯一分组groupId
      const uniqGroupId = computed(() => (props.groupId ? `${props.groupId}_${code}_${fieldConfig[0]['source']}_${fieldConfig[0]['target']}` : ''));
      // update-begin--author:liaozhiyang---date:20230811---for：【issues/675】子表字段Popup弹框数据不更新
      /**
       * 判断popup配置项是否正确
       */
      onMounted(() => {
        if (props.fieldConfig.length == 0) {
          createMessage.error('popup参数未正确配置!');
          avalid.value = false;
        }
      });
      /**
       * 监听value数值
       */
      watch(
        () => props.value,
        (val) => {
          showText.value = val && val.length > 0 ? val.split(props.spliter).join(',') : '';
        },
        { immediate: true }
      );

      /**
       * 打开pop弹出框
       */
      function handleOpen() {
        emit('focus');
        // update-begin--author:liaozhiyang---date:20240528---for：【TV360X-317】禁用后JPopup和JPopupdic还可以点击出弹窗
        !attrs.value.disabled && openModal(true);
        // update-end--author:liaozhiyang---date:20240528---for：【TV360X-317】禁用后JPopup和JPopupdic还可以点击出弹窗
      }

      /**
       * TODO 清空
       */
      function handleEmpty(e) {
        // showText.value = '';
        // update-begin--author:huoshicang---date:20250716
        if (!e.target.value) {
            // 匹配popup设置的回调值
          let { fieldConfig } = props;
          // 初始化数据
          let values = {};
          for (let item of fieldConfig) {
            item.target.split(',').forEach((target) => {
              // 设置默认值为空字符串
              values[target] = '';
            });
          }
          props.formElRef && props.formElRef.setFieldsValue(values);
          props.setFieldsValue && props.setFieldsValue(values);
          emit('popUpChange', values);
        }
        // update-end--author:huoshicang---date:20250716
      }

      /**
       * 传值回调
       */
      function callBack(rows) {
        let { fieldConfig } = props;
        //匹配popup设置的回调值
        let values = {};
        let labels = []
        for (let item of fieldConfig) {
          let val = rows.map((row) => row[item.source]);
          // update-begin--author:liaozhiyang---date:20230831---for：【QQYUN-7535】数组只有一个且是number类型，join会改变值的类型为string
          val = val.length == 1 ? val[0] : val.join(',');
          // update-begin--author:liaozhiyang---date:20230831---for：【QQYUN-7535】数组只有一个且是number类型，join会改变值的类型为string
          item.target.split(',').forEach((target) => {
            values[target] = val;
          });

          if (props.inSearch) {
            // 处理显示值
            if (item.label) {
              let txt = rows.map((row) => row[item.label]);
              txt = txt.length == 1 ? txt[0] : txt.join(',');
              labels.push(txt);
            } else {
              labels.push(val);
            }
          }

        }
        innerShowText.value = labels.join(',');
        //传入表单示例方式赋值
        props.formElRef && props.formElRef.setFieldsValue(values);
        //传入赋值方法方式赋值
        props.setFieldsValue && props.setFieldsValue(values);
        // update-begin--author:liaozhiyang---date:20230831---for：【issues/5288】popup弹框，无法将选择的数据填充到自身
        // update-begin--author:liaozhiyang---date:20230811---for：【issues/5213】JPopup抛出change事件
        emit('popUpChange', values);
        // update-end--author:liaozhiyang---date:20230811---for：【issues/5213】JPopup抛出change事件
        // update-begin--author:liaozhiyang---date:20230831---for：【issues/5288】popup弹框，无法将选择的数据填充到自身
      }

      return {
        showText,
        innerShowText,
        avalid,
        uniqGroupId,
        attrs,
        regModal,
        handleOpen,
        handleEmpty,
        callBack,
      };
    },
  });
</script>
<style lang="less" scoped>
  // update-begin--author:liaozhiyang---date:20240515---for：【QQYUN-9260】必填模式下会影响到弹窗内antd组件的样式
  .JPopup {
    > .ant-form-item {
      display: none;
    }
  }
  // update-end--author:liaozhiyang---date:20240515---for：【QQYUN-9260】必填模式下会影响到弹窗内antd组件的样式
  .components-input-demo-presuffix .anticon-close-circle {
    cursor: pointer;
    color: #ccc;
    transition: color 0.3s;
    font-size: 12px;
  }

  .components-input-demo-presuffix .anticon-close-circle:hover {
    color: #f5222d;
  }

  .components-input-demo-presuffix .anticon-close-circle:active {
    color: #666;
  }
</style>
