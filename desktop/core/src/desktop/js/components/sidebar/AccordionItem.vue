<!--
  Licensed to Cloudera, Inc. under one
  or more contributor license agreements.  See the NOTICE file
  distributed with this work for additional information
  regarding copyright ownership.  Cloudera, Inc. licenses this file
  to you under the Apache License, Version 2.0 (the
  "License"); you may not use this file except in compliance
  with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

<!-- eslint-disable vue/no-v-html -->
<template>
  <div
    ref="containerRef"
    class="sidebar-accordion-item"
    :class="{ 'sidebar-active': isActive }"
    @mouseenter="onMouseEnter"
    @mouseleave="onMouseLeave"
    @focusout="onFocusOut"
  >
    <button
      class="sidebar-base-btn sidebar-sidebar-item"
      :aria-label="item.displayName"
      :class="{ 'sidebar-active': isActive, 'sidebar-accordion-item-btn-open': tooltip }"
      @click="toggleOpen"
      @keypress="onKeyPress"
    >
      <div
        class="sidebar-accordion-collapse-icon"
        :class="{ 'sidebar-accordion-collapse-icon-open': isOpen }"
      >
        <svg viewBox="0 0 24 24" width="1em" height="1em">
          <path
            d="M17 12l-8.678 8L7 18.515 14.069 12 7 5.484 8.322 4z"
            fill="currentColor"
            fill-rule="evenodd"
          />
        </svg>
      </div>

      <div v-if="item.iconHtml" class="sidebar-sidebar-item-icon" v-html="item.iconHtml" />

      <span>{{ item.displayName }}</span>

      <div style="flex-grow: 1;" />
    </button>

    <div
      v-if="!isCollapsed"
      ref="accordionItems"
      class="sidebar-accordion-items"
      :style="{ height: accordionItemsHeight }"
      :aria-hidden="!isOpen || isCollapsed"
    >
      <AccordionSubItem
        v-for="subItem in item.children"
        :key="subItem.name"
        :item="subItem"
        :active="activeItemName === subItem.name"
        :disabled="!isOpen || isCollapsed"
      />
    </div>

    <div
      v-if="tooltip && isCollapsed"
      :style="{ top: tooltip.top + 'px', left: tooltip.right - 28 + 'px' }"
      class="sidebar-sidebar-item-tooltip-extra-hover-area"
      @click="toggleOpen"
    />

    <BaseNavigationItemTooltip
      v-if="tooltip"
      :visible="isCollapsed"
      :style="{
        top: tooltip.top + 'px',
        left: tooltip.right + 'px',
        'max-height': tooltip.maxHeight + 'px'
      }"
      role="tooltip"
    >
      <div
        class="sidebar-sidebar-item-tooltip-primary"
        :class="{ 'sidebar-active': isActive }"
        :style="{ height: tooltip.height + 'px' }"
      >
        {{ item.displayName }}
      </div>
      <div
        class="sidebar-sidebar-item-tooltip-accordion-items"
        :class="{ 'sidebar-sidebar-item-tooltip-accordion-items-overflow-top': isTooltipScrolled }"
        @scroll="onTooltipAccordionItemsScroll"
      >
        <AccordionSubItem
          v-for="child in item.children"
          :key="child.name"
          :item="child"
          :active="activeItemName === child.name"
        />
      </div>
    </BaseNavigationItemTooltip>
  </div>
</template>
<!-- eslint-enable vue/no-v-html -->

<script lang="ts">
  import Vue from 'vue';
  import Component from 'vue-class-component';
  import { Inject, Prop } from 'vue-property-decorator';
  import AccordionSubItem from './AccordionSubItem.vue';
  import BaseNavigationItemTooltip from './BaseNavigationItemTooltip.vue';
  import { SidebarAccordionItem, SidebarNavigationItem } from './types';
  import SubscriptionTracker from 'components/utils/SubscriptionTracker';

  interface Tooltip {
    top: number;
    right: number;
    height: number;
    maxHeight: number;
    fromKeyboard?: boolean;
  }

  @Component({
    components: { BaseNavigationItemTooltip, AccordionSubItem }
  })
  export default class AccordionItem extends Vue {
    @Inject()
    selectedItemChanged?: (itemName: string) => void;

    @Prop()
    item!: SidebarAccordionItem;
    @Prop()
    isCollapsed!: boolean;
    @Prop()
    activeItemName?: string | null = null;

    isOpen = false;
    isTooltipScrolled = false;
    tooltip: Tooltip | null = null;

    subTracker = new SubscriptionTracker();

    disposeParentScroll?: () => void;

    get isActive(): boolean {
      return (
        this.item.name === this.activeItemName ||
        this.item.children.some(item => (<SidebarNavigationItem>item).name === this.activeItemName)
      );
    }

    get accordionItemsHeight(): string {
      const el = <HTMLElement>this.$refs.accordionItems;
      if (this.isOpen && el) {
        return `${el.scrollHeight}px`;
      }
      return '0';
    }

    mounted(): void {
      const containerEl = <HTMLElement>this.$refs.containerRef;
      if (containerEl) {
        const parent = containerEl.parentElement;
        if (parent) {
          this.subTracker.addEventListener(parent, 'scroll', () => {
            this.tooltip = null;
          });
        }
      }
    }

    destroyed(): void {
      this.subTracker.dispose();
    }

    onFocusOut(): void {
      if (!this.tooltip) {
        return;
      }
      window.setTimeout(() => {
        const el = this.$refs.containerRef;
        if (
          el &&
          this.tooltip &&
          this.tooltip.fromKeyboard &&
          !(<HTMLElement>el).contains(document.activeElement)
        ) {
          this.tooltip = null;
        }
      }, 10);
    }

    onKeyPress(event: KeyboardEvent): void {
      if ((event.key === 'Enter' || event.key === ' ') && this.isCollapsed) {
        if (!this.tooltip) {
          this.openTooltip(event.target as HTMLElement, true);
        } else {
          this.tooltip = null;
        }
      }
    }

    onMouseEnter(event: MouseEvent): void {
      if (this.isCollapsed) {
        this.openTooltip(event.target as HTMLElement);
      }
    }

    onMouseLeave(): void {
      this.tooltip = null;
    }

    onTooltipAccordionItemsScroll(event: Event): void {
      this.isTooltipScrolled = (event.target as HTMLElement).scrollTop > 0;
    }

    openTooltip(el: HTMLElement, fromKeyboard?: boolean): void {
      const rect = el.getBoundingClientRect();
      // maxHeight is needed for the edge-cases where the accordion tooltip content would
      // otherwise render outside the viewport.
      const maxHeight = window.innerHeight - rect.top;

      this.isTooltipScrolled = false;
      this.tooltip = {
        top: rect.top,
        right: rect.right,
        height: rect.height,
        maxHeight,
        fromKeyboard
      };
    }

    toggleOpen(event: MouseEvent): void {
      if (!this.isCollapsed) {
        this.isOpen = !this.isOpen;
      } else if (this.item.handler) {
        this.item.handler(event);
        if (this.selectedItemChanged) {
          const firstNavChild = (<SidebarNavigationItem[]>this.item.children).find(
            item => item.name
          );
          this.selectedItemChanged((firstNavChild && firstNavChild.name) || this.item.name);
        }
      }
    }
  }
</script>
