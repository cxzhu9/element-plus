<template>
  <ul v-show="visible" ref="groupRef" :class="ns.be('group', 'wrap')">
    <li :class="ns.be('group', 'title')">{{ label }}</li>
    <li>
      <ul :class="ns.b('group')">
        <slot />
      </ul>
    </li>
  </ul>
</template>

<script lang="ts">
import {
  computed,
  defineComponent,
  getCurrentInstance,
  onMounted,
  provide,
  reactive,
  ref,
  toRefs,
} from 'vue'
import { useMutationObserver } from '@vueuse/core'
import { ensureArray } from '@element-plus/utils'
import { useNamespace } from '@element-plus/hooks'
import { selectGroupKey } from './token'
import type { VNodeChildAtom } from '@element-plus/utils'
import type { VNode, VNodeArrayChildren } from 'vue'

import type { OptionInternalInstance, OptionPublicInstance } from './type'

export default defineComponent({
  name: 'ElOptionGroup',
  componentName: 'ElOptionGroup',

  props: {
    /**
     * @description name of the group
     */
    label: String,
    /**
     * @description whether to disable all options in this group
     */
    disabled: Boolean,
  },
  setup(props) {
    const ns = useNamespace('select')
    const groupRef = ref(null)
    const instance = getCurrentInstance()!
    const children = ref<OptionPublicInstance[]>([])

    provide(
      selectGroupKey,
      reactive({
        ...toRefs(props),
      })
    )

    const visible = computed(() =>
      children.value.some((option) => option.visible === true)
    )

    const isOption = (
      node: VNodeArrayChildren | VNodeChildAtom
    ): node is VNode & { component: OptionInternalInstance } =>
      (node as any).type?.name === 'ElOption' &&
      !!(node as any).component?.proxy

    // get all instances of options
    const flattedChildren = (node: VNode | VNodeArrayChildren | string) => {
      const nodes = ensureArray(node) as VNode[] | VNodeArrayChildren
      const children: OptionPublicInstance[] = []

      nodes.forEach((child) => {
        if (isOption(child)) {
          children.push(child.component.proxy)
        } else if ((child as any).children?.length) {
          children.push(...flattedChildren((child as any).children))
        } else if ((child as any).component?.subTree) {
          children.push(...flattedChildren((child as any).component.subTree))
        }
      })

      return children
    }

    const updateChildren = () => {
      children.value = flattedChildren(instance.subTree)
    }

    onMounted(() => {
      updateChildren()
    })

    useMutationObserver(groupRef, updateChildren, {
      attributes: true,
      subtree: true,
      childList: true,
    })

    return {
      groupRef,
      visible,
      ns,
    }
  },
})
</script>
