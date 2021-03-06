<script>
  // @ts-check
  /** @typedef {{component():*, path: string, isLayout: false, param: false}} Decorator */
  /** @typedef {ClientNode | Decorator} LayoutOrDecorator */
  /**
   * @typedef {Object} Context
   * @prop {ClientNode} layout
   * @prop {ClientNode} component
   * @prop {LayoutOrDecorator} child
   * @prop {SvelteComponent} ComponentFile
   * @prop {HTMLElement} parentNode
   * */

  import { suppressComponentWarnings } from './utils'
  import Noop from './decorators/Noop.svelte'
  import '../typedef.js'
  import { getContext, setContext, tick } from 'svelte'
  import { writable } from 'svelte/store'
  import { metatags, afterPageLoad } from './helpers.js'
  import { route, rootContext } from './store'
  import { handleScroll } from './utils'
  import { onPageLoaded } from './utils/onPageLoaded.js'

  /** @type {LayoutOrDecorator[]} */
  export let nodes = []
  export let scoped = {}
  export let isRoot = false
  export let decorator = undefined

  /** @type {LayoutOrDecorator} */
  let node = null
  let remainingNodes = null
  let scopedSync = {}
  let parentNode
  const context = writable(null)

  /** @type {import("svelte/store").Writable<Context>} */
  const parentContextStore = getContext('routify')
  $: parentContext = $parentContextStore

  const setparentNode = (el) => (parentNode = el.parentNode)

  setContext('routify', context)

  $: [node, ...remainingNodes] = nodes

  /**  @param {LayoutOrDecorator} node */
  function setComponent(node) {
    let PendingComponent = node.component()
    if (PendingComponent instanceof Promise)
      PendingComponent.then(onComponentLoaded)
    else onComponentLoaded(PendingComponent)
  }
  $: setComponent(node)

  /** @param {SvelteComponent} componentFile */
  function onComponentLoaded(componentFile) {
    scopedSync = { ...scoped }
    if (remainingNodes.length === 0) onLastComponentLoaded()

    const ctx = {
      decorator: ((decorator === false || !parentContext) && Noop) || decorator,
      layout:
        (node.isLayout && node) || (parentContext && parentContext.layout),
      component: node,
      route: $route,
      componentFile,
      parentNode: parentNode || (parentContext && parentContext.parentNode),
    }
    context.set(ctx)
    if (isRoot) rootContext.set(ctx)
  }

  async function onLastComponentLoaded() {
    handleScroll(parentNode)

    // let all synchronous code finish before we 
    setTimeout(() => {
      const isOnCurrentRoute = $context.component.path === $route.path //maybe we're getting redirected
      
      // Let everyone know the last child has rendered
      if (!window['routify'].stopAutoReady && isOnCurrentRoute) {
        onPageLoaded({ page: $context.component, metatags, afterPageLoad })
      }
    })
  }

  /**  @param {ClientNode} layout */
  function getID({ meta, path, param, params }) {
    return JSON.stringify({
      path,
      param: (meta['param-is-page'] || meta['slug-is-page']) && param,
      queryParams: meta['query-params-is-page'] && params,
    })
  }
  $: $context && suppressComponentWarnings($context, tick)
</script>

{#if $context}
  {#each [$context] as { component, componentFile, decorator } (getID(component))}
    <svelte:component this={decorator}>
      <svelte:component
        this={componentFile}
        let:scoped={scopeToChild}
        let:decorator
        {scoped}
        {scopedSync}
        {...node.param || {}}>
        {#if component && remainingNodes.length}
          <svelte:self
            {decorator}
            nodes={[...remainingNodes]}
            scoped={{ ...scoped, ...scopeToChild }} />
        {/if}
      </svelte:component>
    </svelte:component>
  {/each}
{/if}
<!-- get the parent element for scroll and transitions -->
<span use:setparentNode />
