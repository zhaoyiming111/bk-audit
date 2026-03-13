<!--
  TencentBlueKing is pleased to support the open source community by making
  蓝鲸智云 - 审计中心 (BlueKing - Audit Center) available.
  Copyright (C) 2023 THL A29 Limited,
  a Tencent company. All rights reserved.
  Licensed under the MIT License (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at http://opensource.org/licenses/MIT
  Unless required by applicable law or agreed to in writing,
  software distributed under the License is distributed on
  an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND,
  either express or implied. See the License for the
  specific language governing permissions and limitations under the License.
  We undertake not to change the open source license (MIT license) applicable
  to the current version of the project delivered to anyone in the future.
-->
<template>
  <div
    ref="wrapperRef"
    class="condition-tag-item condition-tag-event-item"
    :class="{ 'is-editing': isEditingValue }">
    <span class="tag-label">{{ item.display_name }}：</span>
    <!-- 操作符区域 -->
    <bk-popover
      ref="operatorPopoverRef"
      :arrow="false"
      :is-show="isOperatorShow"
      placement="bottom-start"
      theme="light nl-tag-popover"
      trigger="manual"
      @after-hidden="isOperatorShow = false">
      <span
        class="tag-operator-badge"
        :class="{ 'is-active': isOperatorShow }"
        @click.stop="handleToggleOperator">
        {{ item.operator }}
      </span>
      <template #content>
        <div
          class="nl-tag-operator-popover"
          @click.stop
          @mousedown.stop>
          <div
            v-for="op in operatorList"
            :key="op.id"
            class="nl-tag-operator-item"
            :class="{ 'is-selected': item.operator === op.id }"
            @click="handleSelectOperator(op.id)">
            <span>{{ op.name }}</span>
          </div>
        </div>
      </template>
    </bk-popover>
    <!-- 值区域 -->
    <template v-if="!isEditingValue">
      <span
        v-bk-tooltips="{
          content: fullDisplayValue,
          disabled: !isOverflow,
          extCls: 'nl-tag-tooltip-wrap',
        }"
        class="tag-value-wrapper"
        @click.stop="handleStartEditValue">
        <span class="tag-value">{{ displayValue }}</span>
      </span>
    </template>
    <template v-else>
      <div class="nl-tag-event-edit-zone">
        <!-- IN/NOT IN 模式：简单的 tag-input 风格 -->
        <template v-if="isMultiValueMode">
          <input
            ref="multiInputRef"
            v-model="multiInputText"
            class="nl-tag-event-inline-input"
            :placeholder="t('输入后按回车')"
            @keydown.enter.prevent="handleAddMultiValue">
          <div
            v-if="localMultiValue.length > 0"
            class="nl-tag-event-multi-tags">
            <span
              v-for="(val, idx) in localMultiValue"
              :key="idx"
              class="nl-tag-event-multi-tag">
              {{ val }}
              <audit-icon
                class="multi-tag-close"
                type="close"
                @click.stop="handleRemoveMultiValue(idx)" />
            </span>
          </div>
        </template>
        <!-- 单值模式 -->
        <template v-else>
          <input
            ref="singleInputRef"
            v-model="localSingleValue"
            class="nl-tag-event-inline-input"
            :placeholder="t('请输入')"
            @keydown.enter.prevent="handleConfirmValue">
        </template>
      </div>
    </template>
    <audit-icon
      class="tag-remove-btn"
      type="close"
      @click.stop="$emit('remove', item.id)" />
  </div>
</template>
<script setup lang="ts">
  import {
    computed,
    nextTick,
    onBeforeUnmount,
    ref,
    watch,
  } from 'vue';
  import { useI18n } from 'vue-i18n';

  interface Props {
    item: Record<string, any>;
    conditionList: Array<{ id: string; name: string }>;
  }
  interface Emits {
    (e: 'remove', id: string): void;
    (e: 'updateOperator', id: string, operator: string): void;
    (e: 'updateValue', id: string, value: any): void;
  }

  const props = defineProps<Props>();
  const emit = defineEmits<Emits>();
  const { t } = useI18n();

  const MAX_DISPLAY_LEN = 15;

  const wrapperRef = ref<HTMLElement>();
  const operatorPopoverRef = ref();
  const isOperatorShow = ref(false);
  const isEditingValue = ref(false);
  const multiInputRef = ref<HTMLInputElement>();
  const singleInputRef = ref<HTMLInputElement>();

  // 单值模式的本地值
  const localSingleValue = ref('');
  // 多值模式的本地值
  const localMultiValue = ref<string[]>([]);
  const multiInputText = ref('');

  // 操作符列表（与接口 event_filters 操作符枚举保持一致）
  const operatorList = computed(() => props.conditionList || [
    { id: '=', name: '=' },
    { id: '!=', name: '!=' },
    { id: 'CONTAINS', name: '包含' },
    { id: 'NOT CONTAINS', name: '不包含' },
    { id: 'IN', name: 'IN' },
    { id: 'NOT IN', name: 'NOT IN' },
    { id: '>=', name: '>=' },
    { id: '<=', name: '<=' },
    { id: '>', name: '>' },
    { id: '<', name: '<' },
  ]);

  // 是否为多值模式（IN / NOT IN）
  const isMultiValueMode = computed(() => props.item.operator === 'IN' || props.item.operator === 'NOT IN');

  // 完整显示值
  const fullDisplayValue = computed(() => {
    const { value } = props.item;
    if (!value && value !== 0) return '--';
    if (Array.isArray(value)) {
      return value.length > 0 ? value.join('，') : '--';
    }
    return String(value) || '--';
  });

  // 截断显示值
  const displayValue = computed(() => {
    const text = fullDisplayValue.value;
    if (text.length > MAX_DISPLAY_LEN) {
      return `${text.slice(0, MAX_DISPLAY_LEN)}...`;
    }
    return text;
  });

  const isOverflow = computed(() => fullDisplayValue.value.length > MAX_DISPLAY_LEN);

  // 操作符下拉切换
  const handleToggleOperator = () => {
    isOperatorShow.value = !isOperatorShow.value;
  };

  // 选择操作符
  const handleSelectOperator = (operator: string) => {
    emit('updateOperator', props.item.id, operator);
    isOperatorShow.value = false;
  };

  // 开始编辑值
  const handleStartEditValue = () => {
    if (isMultiValueMode.value) {
      localMultiValue.value = Array.isArray(props.item.value) ? [...props.item.value] : [];
      multiInputText.value = '';
    } else {
      localSingleValue.value = props.item.value || '';
    }
    isEditingValue.value = true;
    nextTick(() => {
      if (isMultiValueMode.value) {
        multiInputRef.value?.focus();
      } else {
        singleInputRef.value?.focus();
      }
    });
  };

  // 单值模式确认
  const handleConfirmValue = () => {
    emit('updateValue', props.item.id, localSingleValue.value);
    isEditingValue.value = false;
  };

  // 多值模式添加
  const handleAddMultiValue = () => {
    const text = multiInputText.value.trim();
    if (!text) return;
    // 支持逗号分隔批量输入
    const values = text.split(',').map(s => s.trim())
      .filter(s => s);
    const newValues = [...new Set([...localMultiValue.value, ...values])];
    localMultiValue.value = newValues;
    multiInputText.value = '';
    emit('updateValue', props.item.id, newValues);
  };

  // 多值模式删除
  const handleRemoveMultiValue = (index: number) => {
    localMultiValue.value.splice(index, 1);
    emit('updateValue', props.item.id, [...localMultiValue.value]);
  };

  // 点击外部关闭编辑态
  const handleDocumentClick = (e: MouseEvent) => {
    const target = e.target as HTMLElement;
    if (wrapperRef.value?.contains(target)) return;
    // 关闭操作符下拉
    if (isOperatorShow.value) {
      const closestTippy = (target as Element)?.closest?.('.tippy-box[data-theme~="nl-tag-popover"]');
      if (!closestTippy) {
        isOperatorShow.value = false;
      }
    }
    // 关闭值编辑态
    if (isEditingValue.value) {
      if (isMultiValueMode.value) {
        // 多值模式：如有未提交的文本，先添加
        if (multiInputText.value.trim()) {
          handleAddMultiValue();
        }
      } else {
        handleConfirmValue();
      }
      isEditingValue.value = false;
    }
  };

  watch(isEditingValue, (val) => {
    if (val) {
      setTimeout(() => {
        document.addEventListener('click', handleDocumentClick);
      });
    } else {
      document.removeEventListener('click', handleDocumentClick);
    }
  });

  watch(isOperatorShow, (val) => {
    if (val && !isEditingValue.value) {
      setTimeout(() => {
        document.addEventListener('click', handleDocumentClick);
      });
    } else if (!val && !isEditingValue.value) {
      document.removeEventListener('click', handleDocumentClick);
    }
  });

  onBeforeUnmount(() => {
    document.removeEventListener('click', handleDocumentClick);
  });
</script>
<style lang="postcss" scoped>
  .condition-tag-event-item {
    height: auto;
    min-height: 26px;

    .tag-operator-badge {
      display: inline-flex;
      height: 18px;
      padding: 0 4px;
      margin-right: 4px;
      font-size: 12px;
      font-weight: 700;
      color: #979ba5;
      cursor: pointer;
      background: transparent;
      border-radius: 2px;
      transition: all .15s;
      align-items: center;

      &:hover,
      &.is-active {
        background: #fdeed8;
      }

      /* &.is-active {
        color: #3a84ff;
        background: #e1ecff;
      } */
    }
  }

  .nl-tag-event-edit-zone {
    display: inline-flex;
    flex-wrap: wrap;
    gap: 4px;
    align-items: center;
    max-width: 300px;
  }

  .nl-tag-event-inline-input {
    width: 120px;
    max-width: 200px;
    min-width: 80px;
    min-height: 20px;
    padding: 0 4px;
    font-family: inherit;
    font-size: 12px;
    line-height: 18px;
    color: #63656e;
    background: #fff;
    border: 1px solid #3a84ff;
    border-radius: 2px;
    outline: none;
    box-sizing: border-box;

    &::placeholder {
      color: #c4c6cc;
    }
  }

  .nl-tag-event-multi-tags {
    display: inline-flex;
    flex-wrap: wrap;
    gap: 2px;
    align-items: center;
  }

  .nl-tag-event-multi-tag {
    display: inline-flex;
    height: 20px;
    padding: 0 4px;
    font-size: 12px;
    color: #63656e;
    background: #f0f1f5;
    border-radius: 2px;
    align-items: center;
    gap: 2px;

    .multi-tag-close {
      font-size: 12px;
      color: #979ba5;
      cursor: pointer;

      &:hover {
        color: #ea3636;
      }
    }
  }
</style>
<style lang="postcss">
  /* 操作符下拉弹出层样式（非 scoped，因为 popover 内容挂在 body 上） */
  .nl-tag-operator-popover {
    min-width: 80px;
    padding: 4px 0;

    .nl-tag-operator-item {
      display: flex;
      height: 32px;
      padding: 0 12px;
      font-size: 12px;
      color: #63656e;
      cursor: pointer;
      align-items: center;
      transition: background .15s;

      &:hover {
        background: #f5f7fa;
      }

      &.is-selected {
        color: #3a84ff;
        background: #e1ecff;
      }
    }
  }
</style>
