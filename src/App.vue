<script setup>
import {
  defineComponent,
  h,
  ref,
  reactive,
  computed,
  watch,
  onMounted,
  onBeforeUnmount,
  onErrorCaptured,
} from 'vue'

// Optional manual overrides for props whose possible values can't be detected
// automatically, e.g. { Vue3MdsButton: { type: ['primary', 'secondary', 'tiertary'] } }
const stateOverrides = {}

// Props that get a sample value so the preview isn't empty.
const SAMPLE_PROP = /label|title|text|content|placeholder/i

const modules = import.meta.glob('/src/components/vue-3/base/**/*.vue', {
  eager: true,
})

// ---------- reading a component's props definition ----------

function typeNames(types) {
  return types.map((t) => t?.name ?? String(t)).join(' | ') || 'any'
}

function normalizeProps(def) {
  const raw = def?.props
  if (!raw) return []
  if (Array.isArray(raw)) {
    return raw.map((name) => ({
      name,
      kind: 'text',
      typeLabel: 'any',
      required: false,
      default: undefined,
      validator: null,
    }))
  }
  return Object.entries(raw).map(([name, opt]) => {
    const o =
      typeof opt === 'function' || Array.isArray(opt) ? { type: opt } : opt || {}
    const types = [].concat(o.type ?? [])
    const kind = types.includes(Boolean)
      ? 'boolean'
      : types.includes(Number)
        ? 'number'
        : types.includes(Array) || types.includes(Object)
          ? 'json'
          : 'text'
    const fallback =
      typeof o.default === 'function' && !types.includes(Function)
        ? o.default()
        : o.default
    return {
      name,
      kind,
      typeLabel: typeNames(types),
      required: !!o.required,
      default: fallback,
      validator: typeof o.validator === 'function' ? o.validator : null,
    }
  })
}

// Possible values: manual override > boolean > string literals accepted by the validator.
function possibleValues(compName, d) {
  const manual = stateOverrides[compName]?.[d.name]
  if (manual) return { values: manual, source: 'override' }
  if (d.kind === 'boolean') return { values: [true, false], source: 'boolean' }
  if (d.validator) {
    const found = new Set()
    const re = /(['"`])((?:\\.|(?!\1)[^\\])*)\1/g
    let m
    const src = d.validator.toString()
    while ((m = re.exec(src))) {
      try {
        if (d.validator(m[2]) === true) found.add(m[2])
      } catch {
        /* candidate rejected */
      }
    }
    if (found.size) return { values: [...found], source: 'validator' }
  }
  return { values: [], source: null }
}

function format(v) {
  return typeof v === 'string' ? `"${v}"` : JSON.stringify(v)
}

// ---------- initial editable values ----------

function baseProps(propDefs) {
  const out = {}
  for (const d of propDefs) {
    if (d.default !== undefined) out[d.name] = d.default
    else if (d.kind === 'text' && (d.required || SAMPLE_PROP.test(d.name)))
      out[d.name] = 'Sample'
  }
  return out
}

function initialValues(propDefs) {
  const base = baseProps(propDefs)
  return Object.fromEntries(
    propDefs.map((d) => {
      const v = base[d.name]
      if (d.kind === 'boolean') return [d.name, !!v]
      if (d.kind === 'json')
        return [d.name, v === undefined ? '' : JSON.stringify(v, null, 2)]
      return [d.name, v ?? '']
    })
  )
}

const components = Object.entries(modules)
  .map(([path, mod]) => {
    const name = path.split('/').pop().replace('.vue', '')
    const propDefs = normalizeProps(mod.default).map((d) => {
      const { values, source } = possibleValues(name, d)
      return { ...d, values, valuesSource: source }
    })
    return { path, name, component: mod.default, propDefs }
  })
  .sort((a, b) => a.name.localeCompare(b.name))

// Editable state per component: current prop values + default slot text.
const state = reactive(
  Object.fromEntries(
    components.map((c) => [c.path, { props: initialValues(c.propDefs), slot: '' }])
  )
)

function reset(c) {
  state[c.path].props = initialValues(c.propDefs)
  state[c.path].slot = ''
}

function isInvalidJson(text) {
  if (!text) return false
  try {
    JSON.parse(text)
    return false
  } catch {
    return true
  }
}

// What is actually passed to the component; empty fields are left unset so defaults apply.
function boundProps(c) {
  const out = {}
  for (const d of c.propDefs) {
    const v = state[c.path].props[d.name]
    if (d.kind === 'boolean') out[d.name] = v
    else if (v === '' || v == null) continue
    else if (d.kind === 'number') out[d.name] = Number(v)
    else if (d.kind === 'json') {
      try {
        out[d.name] = JSON.parse(v)
      } catch {
        /* invalid JSON while typing: ignore until it parses */
      }
    } else out[d.name] = v
  }
  return out
}

function usage(c) {
  const attrs = Object.entries(boundProps(c))
    .map(([k, v]) =>
      v === true ? k : typeof v === 'string' ? `${k}="${v}"` : `:${k}='${JSON.stringify(v)}'`
    )
    .join(' ')
  const open = `<${c.name}${attrs ? ' ' + attrs : ''}`
  const slot = state[c.path].slot
  return slot ? `${open}>${slot}</${c.name}>` : `${open} />`
}

// ---------- sticky header height (so the sticky preview sits just below it) ----------

const rootEl = ref(null)
const headerEl = ref(null)
let resizeObserver = null

function syncHeaderHeight() {
  if (rootEl.value && headerEl.value)
    rootEl.value.style.setProperty('--gallery-header-h', `${headerEl.value.offsetHeight}px`)
}
onMounted(() => {
  syncHeaderHeight()
  resizeObserver = new ResizeObserver(syncHeaderHeight)
  if (headerEl.value) resizeObserver.observe(headerEl.value)
})
onBeforeUnmount(() => resizeObserver?.disconnect())

// ---------- copy usage snippet ----------

const copiedPath = ref(null)
let copiedTimer = null

async function copyUsage(c) {
  const text = usage(c)
  try {
    await navigator.clipboard.writeText(text)
  } catch {
    // Fallback for insecure contexts or blocked clipboard access.
    const ta = document.createElement('textarea')
    ta.value = text
    ta.style.position = 'fixed'
    ta.style.opacity = '0'
    document.body.appendChild(ta)
    ta.select()
    document.execCommand('copy')
    ta.remove()
  }
  copiedPath.value = c.path
  clearTimeout(copiedTimer)
  copiedTimer = setTimeout(() => (copiedPath.value = null), 1500)
}

// ---------- page state ----------

const search = ref('')
const filtered = computed(() =>
  components.filter((c) => c.name.toLowerCase().includes(search.value.toLowerCase()))
)

// Collapsed by default; a component's panel only renders once opened.
const open = reactive(new Set())

function toggle(path) {
  if (open.has(path)) open.delete(path)
  else open.add(path)
}
function expandAll() {
  filtered.value.forEach((c) => open.add(c.path))
}
function collapseAll() {
  open.clear()
}

// Isolates the preview so a component that throws doesn't take down the page.
const SafeRender = defineComponent({
  props: {
    component: { type: Object, required: true },
    componentProps: { type: Object, default: () => ({}) },
    slotText: { type: String, default: '' },
  },
  setup(props) {
    const error = ref(null)
    onErrorCaptured((err) => {
      error.value = err.message
      return false
    })
    // Try again whenever the inputs change.
    watch(
      () => [props.componentProps, props.slotText],
      () => (error.value = null)
    )
    return () =>
      error.value
        ? h('span', { class: 'gallery__error' }, `Failed to render: ${error.value}`)
        : h(
            props.component,
            props.componentProps,
            props.slotText ? { default: () => props.slotText } : undefined
          )
  },
})
</script>

<template>
  <section ref="rootEl" class="gallery">
    <header ref="headerEl" class="gallery__header">
      <h1>Base components ({{ filtered.length }})</h1>
      <div class="gallery__controls">
        <input
          v-model="search"
          type="search"
          placeholder="Filter by name"
          aria-label="Filter components by name"
        />
        <button type="button" @click="expandAll">Expand all</button>
        <button type="button" @click="collapseAll">Collapse all</button>
      </div>
    </header>

    <p v-if="!filtered.length" class="gallery__empty">
      No components found in src/components/vue-3/base/.
    </p>

    <article v-for="c in filtered" :key="c.path" class="gallery__item">
      <h2 class="gallery__title">
        <button
          type="button"
          class="gallery__toggle"
          :aria-expanded="open.has(c.path)"
          :aria-controls="`panel-${c.name}`"
          @click="toggle(c.path)"
        >
          <span class="gallery__chevron" :class="{ 'is-open': open.has(c.path) }" aria-hidden="true">▸</span>
          <span class="gallery__name">{{ c.name }}</span>
          <span class="gallery__meta">
            {{ c.propDefs.length }} {{ c.propDefs.length === 1 ? 'prop' : 'props' }}
          </span>
        </button>
      </h2>

      <div v-if="open.has(c.path)" :id="`panel-${c.name}`" class="gallery__body">
        <code class="gallery__path">{{ c.path }}</code>

        <div class="gallery__cols">
          <!-- Left: editable props -->
          <section class="gallery__col" :aria-label="`${c.name} props`">
            <div class="gallery__colhead">
              <h3>Props</h3>
              <button type="button" class="gallery__reset" @click="reset(c)">Reset</button>
            </div>

            <div class="gallery__scroll">
              <table>
                <thead>
                  <tr>
                    <th>Prop</th>
                    <th>Type</th>
                    <th>Default</th>
                    <th>Value</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="d in c.propDefs" :key="d.name">
                    <td>
                      <code>{{ d.name }}</code>
                      <span v-if="d.required" class="gallery__req" title="Required">*</span>
                    </td>
                    <td>{{ d.typeLabel }}</td>
                    <td>
                      <code v-if="d.default !== undefined">{{ format(d.default) }}</code>
                      <span v-else class="gallery__muted">none</span>
                    </td>
                    <td>
                      <input
                        v-if="d.kind === 'boolean'"
                        v-model="state[c.path].props[d.name]"
                        type="checkbox"
                        :aria-label="d.name"
                      />
                      <select
                        v-else-if="d.values.length"
                        v-model="state[c.path].props[d.name]"
                        :aria-label="d.name"
                        :title="d.valuesSource === 'validator' ? 'Values detected from validator' : ''"
                      >
                        <option value="">(default)</option>
                        <option v-for="v in d.values" :key="String(v)" :value="v">{{ v }}</option>
                      </select>
                      <input
                        v-else-if="d.kind === 'number'"
                        v-model="state[c.path].props[d.name]"
                        type="number"
                        :aria-label="d.name"
                      />
                      <template v-else-if="d.kind === 'json'">
                        <textarea
                          v-model="state[c.path].props[d.name]"
                          rows="3"
                          :aria-label="d.name"
                          :class="{ 'is-invalid': isInvalidJson(state[c.path].props[d.name]) }"
                        />
                        <span v-if="isInvalidJson(state[c.path].props[d.name])" class="gallery__error">
                          Invalid JSON
                        </span>
                      </template>
                      <input
                        v-else
                        v-model="state[c.path].props[d.name]"
                        type="text"
                        :aria-label="d.name"
                      />
                    </td>
                  </tr>
                  <tr>
                    <td><code>default slot</code></td>
                    <td>text</td>
                    <td><span class="gallery__muted">none</span></td>
                    <td>
                      <input v-model="state[c.path].slot" type="text" aria-label="default slot" />
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
            <p v-if="!c.propDefs.length" class="gallery__muted">No props declared.</p>
          </section>

          <!-- Right: live preview -->
          <section class="gallery__col gallery__col--preview" :aria-label="`${c.name} preview`">
            <div class="sticky">
              <div class="gallery__colhead">
                <h3>Preview</h3>
              </div>
              <div class="gallery__stage">
                <SafeRender
                  :component="c.component"
                  :component-props="boundProps(c)"
                  :slot-text="state[c.path].slot"
                />
              </div>
              <div class="gallery__usagewrap">
                <pre class="gallery__usage">{{ usage(c) }}</pre>
                <button type="button" class="gallery__copy" @click="copyUsage(c)">
                  {{ copiedPath === c.path ? 'Copied!' : 'Copy' }}
                </button>
                <span class="gallery__sr" role="status">
                  {{ copiedPath === c.path ? 'Usage copied to clipboard' : '' }}
                </span>
              </div>
            </div>
          </section>
        </div>
      </div>
    </article>
  </section>
</template>

<style scoped>
.gallery {
  padding: 24px;
  font-family: system-ui, sans-serif;
}
.sticky {
  position: sticky;
  top: 90px;
}
.gallery__header {
  position: sticky;
  top: 0;
  z-index: 10;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  /* cancel the section padding so the bar spans the full width */
  margin: -24px -24px 24px;
  padding: 16px 24px;
  border-bottom: 1px solid #dde0e3;
  background: #fff;
}
.gallery__header h1 {
  margin: 0;
  font-size: 1.25rem;
}
.gallery__header input {
  padding: 8px 12px;
  border: 1px solid #c8ccd0;
  border-radius: 6px;
}
.gallery__controls {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
.gallery__controls button,
.gallery__reset {
  padding: 8px 12px;
  border: 1px solid #c8ccd0;
  border-radius: 6px;
  background: #fff;
  cursor: pointer;
}
.gallery__reset {
  padding: 4px 10px;
  font-size: 0.8rem;
}
.gallery__controls button:hover,
.gallery__reset:hover {
  background: #eceff1;
}
.gallery__item {
  margin-bottom: 12px;
  border: 1px solid #dde0e3;
  border-radius: 8px;
}
.gallery__title {
  margin: 0;
  font-size: 1.1rem;
}
.gallery__toggle {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  padding: 14px 20px;
  border: 0;
  border-radius: 8px;
  background: transparent;
  font: inherit;
  text-align: left;
  cursor: pointer;
}
.gallery__toggle:hover {
  background: #f6f7f8;
}
.gallery__toggle:focus-visible,
.gallery__controls button:focus-visible,
.gallery__reset:focus-visible {
  outline: 2px solid #3d5afe;
}
.gallery__chevron {
  display: inline-block;
  transition: transform 0.15s;
}
.gallery__chevron.is-open {
  transform: rotate(90deg);
}
.gallery__name {
  font-weight: 600;
}
.gallery__meta {
  margin-left: auto;
  color: #777;
  font-size: 0.8rem;
  font-weight: 400;
}
.gallery__body {
  padding: 0 20px 20px;
}
.gallery__path {
  font-size: 0.75rem;
  color: #666;
  word-break: break-all;
}

/* two columns: props on the left, preview on the right */
.gallery__cols {
  display: grid;
  grid-template-columns: minmax(0, 1.2fr) minmax(0, 1fr);
  gap: 20px;
  margin-top: 12px;
}
@media (max-width: 900px) {
  .gallery__cols {
    grid-template-columns: minmax(0, 1fr);
  }
}
.gallery__colhead {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}
.gallery__colhead h3 {
  margin: 0;
  font-size: 0.95rem;
}
.gallery__scroll {
  overflow-x: auto;
}
table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.85rem;
}
th,
td {
  padding: 8px 10px;
  text-align: left;
  vertical-align: middle;
  border-bottom: 1px solid #eceff1;
}
th {
  background: #f6f7f8;
  font-weight: 600;
  white-space: nowrap;
}
td input[type='text'],
td input[type='number'],
td select,
td textarea {
  width: 100%;
  min-width: 100px;
  padding: 5px 8px;
  border: 1px solid #c8ccd0;
  border-radius: 4px;
  background: #fff;
  font: inherit;
}
td textarea.is-invalid {
  border-color: #b00020;
}
code {
  font-size: 0.8rem;
}
.gallery__req {
  margin-left: 2px;
  color: #b00020;
}
.gallery__stage {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 160px;
  padding: 24px;
  border: 1px dashed #b8bec4;
  border-radius: 8px;
  background: #fafbfb;
}
.gallery__usage {
  margin: 12px 0 0;
  padding: 12px;
  overflow-x: auto;
  border-radius: 6px;
  background: #1f2328;
  color: #e6edf3;
  font-size: 0.8rem;
}
.gallery__muted {
  color: #777;
  font-size: 0.8rem;
}
.gallery__error {
  color: #b00020;
  font-size: 0.8rem;
}
@media (prefers-reduced-motion: reduce) {
  .gallery__chevron {
    transition: none;
  }
}
</style>