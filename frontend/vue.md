# 前端 · Vue（面试题库）

本题库聚焦 Vue 2/3 的工程实践，重点考察响应式原理与真实坑点、组合式 API 的工程化应用、组件设计与通信方案取舍、长列表与渲染性能优化，以及 SSR、微前端、TypeScript 迁移等大规模工程实践。全部题目均为场景化提问，不考概念背诵，目的是通过真实业务问题快速判断候选人的实践深度、方案取舍能力与排障思路。

### Q1. 动态添加属性不更新，Vue 2 与 Vue 3 的响应式差异

**问题：** Vue 2 项目里给对象动态添加属性（如 `this.obj.newField = 1`）页面不更新，怎么解决？Vue 3 还会这样吗？为什么？

**期望加分项：**
- 能准确说出 Vue 2 用 `Object.defineProperty` 在初始化时递归劫持已有属性，新增属性未经劫持所以不响应
- 给出至少两种修复方案（`Vue.set`/`this.$set`、`Object.assign` 整体替换），并说明各自适用场景
- 能补充 Vue 2 对数组索引直接赋值、修改 `length` 同样不响应，及对应处理方式（`splice`）
- 能说明 Vue 3 用 `Proxy` 后新增属性天然响应，但仍存在 `Object.freeze`、`Map/Set` 等边界
- 联系线上实践：接口返回动态字段导致表格列/表单控件渲染不出来的真实排障过程

**减分项：**
- 只会背「用 $set」但讲不清原理
- 答不出 Vue 2 数组方法的特殊处理
- 误以为 Vue 3 完全没有响应式限制
- 说不出 Proxy 懒代理相比 defineProperty 递归劫持在性能上的差异

**解答：**
先定位本质：Vue 2 的响应式建立在 `Object.defineProperty` 上，初始化时对 data 深度递归劫持已有属性的 getter/setter，之后新增的键没有被劫持，赋值自然不触发更新；数组则只重写了 `push/pop/splice` 等 7 个变更方法，`arr[2] = x` 或修改 `length` 也不响应。修复首选 `this.$set(obj, 'key', val)`，或整体替换 `this.obj = { ...this.obj, key: val }`（简单直观但会重建对象）；数组用 `splice(index, 1, val)` 替换指定项。线上最常见的场景是接口动态返回字段，表格或表单渲染不出来——排查先看 Devtools 里数据是否已变化：数据变了但视图没动，基本就是新增属性或数组索引问题。Vue 3 改用 `Proxy` 重写响应式，get 时惰性收集依赖、set/delete 统一拦截，新增属性天然响应；但仍要注意边界：`Object.freeze` 冻结对象不响应、`Map/Set` 需整体替换引用、`reactive` 对象不能整体重新赋值（要用 `Object.assign` 合并而不是直接 `state = newObj`）。另外 Proxy 是懒代理，大对象初始化远快于 Vue 2 的深度递归，这也是 Vue 3 首屏性能提升的来源之一。

**延伸考点：** 如果一个对象在初始化时就被 `Object.freeze`，Vue 3 里修改它会怎样？为什么 `reactive` 对象不能整体重新赋值？

### Q2. ref 与 reactive 的选型，解构后失去响应式

**问题：** Vue 3 里把 `reactive` 对象解构后页面不更新，是怎么回事？项目里什么时候用 `ref`、什么时候用 `reactive`，你们是怎么约定的？

**期望加分项：**
- 解释解构丢失响应式的原因：`reactive` 返回的是 Proxy 代理对象，解构拿到的是普通值/普通属性引用，脱离了代理的拦截
- 给出 `toRefs`/`toRef` 解法，并说明两者的区别
- 能说清 `ref` 适合基础类型、`reactive` 适合对象，但给出团队统一约定及理由
- 提到 `.value` 在模板中自动解包、在 `reactive` 对象中自动解包等细节
- 结合线上实践：从选项式 API 迁移组合式时遇到的解构坑

**减分项：**
- 只知道「别解构」但说不清原因
- 分不清 `toRef` 与 `toRefs` 的区别
- 说不清 ref 在模板/响应式对象中的自动解包行为
- 没有自己的选型约定，只会说「看情况」

**解答：**
先讲原因：`reactive` 返回的是 Proxy 代理对象，`const { name, age } = state` 解构时拿到的只是普通变量，脱离了代理的 getter/setter 拦截，修改解构出来的变量不会触发视图更新。解法是 `toRefs`：把 reactive 对象的每个属性转换为 ref 再解构，解构后的变量通过 `.value` 操作，仍指向原对象的响应式引用；只想转换某一个属性用 `toRef`。工程上的选型：`ref` 可以容纳任意类型（包括对象），且 `.value` 在模板里自动解包、放进 reactive 对象里也会自动解包，心智模型统一；`reactive` 更适合局部、深层嵌套的纯对象数据。很多团队为降低心智负担直接约定「全用 ref」，reactive 只在确实需要保持对象引用或深层结构时使用。实践中最常见的坑出现在迁移场景：把整个 data 换成 reactive 后解构赋值，页面静默不更新，调试半天；另外要注意 ref 放进**普通**对象不会自动解包，放进 reactive 对象才会。props 本身是 reactive 的，直接解构也会丢响应，Vue 3.5 起 defineProps 的解构默认保持响应式，老版本需要 `toRefs(props)` 或在 setup 内谨慎处理。

**延伸考点：** `toRefs` 转换出来的 ref 和原对象属性之间是单向还是双向同步？`reactive` 包 `ref` 时 `.value` 会被自动解开，这带来什么坑？

### Q3. computed 与 watch 的选型及各自的坑

**问题：** 一个搜索页，用户输入关键词后要防抖发起搜索，同时输入长度决定「搜索」按钮是否可用，你会用 computed 还是 watch？为什么？watch 用起来有哪些坑？

**期望加分项：**
- 说清判断标准：computed 是「派生值、有缓存」适合纯计算；watch 是「副作用」适合请求、埋点、批量操作
- 该场景的组合写法：computed 算按钮可用性，watch 做防抖搜索
- 能讲 watch 的 `deep`、`immediate`、监听路径字符串（`'form.name'`）的用法与代价
- 能处理异步竞态：watch 回调里用 `onCleanup`/`AbortController` 取消过期请求
- 主动点出「能用 computed 派生的关系不要用 watch 手工同步」的工程原则

**减分项：**
- 说不清两者本质区别，把 watch 当 computed 用
- 不知道 `deep: true` 时新旧值是同一引用、拿不到旧值
- 深监听大对象毫无性能意识
- 没有处理过「watch 触发连锁接口请求」的问题

**解答：**
判断标准很简单：computed 是根据已有状态**派生新值**，有缓存、依赖变化才重算，适合「输入长度决定按钮可用」「筛选后的列表」这类纯计算；watch 是**监听状态变化去执行副作用**，适合搜索请求、埋点上报、同步外部系统。所以该场景应组合使用：computed 算按钮状态，watch 里做防抖 + 取消上一次请求。watch 的坑要展开：一是 `deep: true` 监听对象时新旧值都是同一个引用，无法对比，且深监听性能开销大，能监听具体路径就写字符串路径（`'form.name'`）；二是 `immediate: true` 可以初始化时立即执行一次，适合「进页面按初始条件拉数据」；三是异步竞态——快速输入时旧请求后返回会覆盖新数据，Vue 3 的 watch 回调里可以用 `onCleanup` 注册清理函数，配合 `AbortController` 取消过期请求。线上还常见「watch 了不该 watch 的东西导致接口连环触发」，排查原则：依赖方向要想清楚，能用 computed 表达的派生关系永远不要用 watch 手工同步，watch 只做「有副作用的响应」。

**延伸考点：** watch 和 watchEffect 的区别？组件卸载时 watch 会自动停止吗，事件总线上的订阅会吗？

### Q4. 组件通信方案怎么选

**问题：** 一个三级嵌套的组件树，孙组件要修改根组件的一个状态；另一个场景是多个页面都要读写登录信息。你会分别怎么通信？props/emit、v-model、provide/inject、事件总线、Pinia 各自的适用边界是什么？

**期望加分项：**
- 先给排序原则：能用最局部的通信就别上全局状态，优先单向数据流
- 能说 props + emit、v-model 适用于父子；provide/inject 适用于跨多层但数据「下发为主」；事件总线只适合低频点对点；跨页面共享状态直接 Pinia
- 主动指出 provide/inject 的问题：隐式耦合、无法追踪来源
- 指出事件总线的坑：破坏数据流可追踪性、需要手动 off 防泄漏，多人协作易失控
- 给出自己项目的分层实践：表单页内部 provide/inject 共享 form 实例，跨页面用 Pinia

**减分项：**
- 答「一律用 Pinia/vuex」或「一律用事件总线」这种单一方案
- 说不出 provide/inject 和事件总线各自的缺点
- 没有实际项目的分层经验

**解答：**
先给排序原则：能用最局部的通信方式就别上全局状态。父子之间用 props 传参、emit 回传，必要时用 v-model 简化双向绑定，这是默认形态；跨多层、但本质是「祖先向下下发」的场景（主题、表单上下文、页面级配置、区域权限）用 provide/inject，它的优点是省去层层透传，代价是组件与祖先隐式耦合、来源不可追踪，误用会变成「魔法数据」；事件总线（mitt 等）只适合低频、点对点、跨多层且不想引入全局状态的临时方案，它破坏了数据流可追踪性，组件销毁时必须手动 off，多人协作下极易变成「面条代码」，新代码不建议引入；一旦状态被多个页面/路由共享、需要持久化或调试回溯，直接上 Pinia，不要在 props 上硬钻、也不要用事件总线模拟全局状态。加分点是能给出自己的分层模型，比如：页面容器内多区块共享用 provide/inject（表单注册表、校验上下文），跨页面共享用 Pinia（用户、权限、购物车），DOM/组件事件用 emit，第三方库回调（如地图 marker 点击）才用全局事件。减分点是答「都行」或只有一种方案，说明没有经历过取舍。

**延伸考点：** provide 的响应式有什么坑（传值还是传 ref）？事件总线与 Pinia 在「跨组件通知」上的本质区别？

### Q5. v-for 的 key 用 index 会出什么问题

**问题：** 一个可勾选、可删除的列表，key 用了数组下标。删除中间一项后，后面的勾选状态和输入框内容错乱了，为什么？key 应该怎么生成？

**期望加分项：**
- 说清 key 是 diff 时判断「是否同一节点」的依据，index 在增删后会错位，Vue 误判节点复用
- 说明复用带来的具体后果：组件实例复用导致内部状态（勾选、输入、动画）错位
- 正确做法：稳定且唯一的 id（后端主键、uuid），没有 id 时用内容联合生成
- 能给出判断边界：纯展示、顺序稳定的列表用 index 问题不大，但一旦增删排序或有内部状态就不能用
- 联系线上案例：表格行拖拽排序、分页后勾选错乱、列表动画错乱

**减分项：**
- 只会背「不能用 index」，说不清 diff 复用机制
- 认为所有列表都必须用 id，说不出「纯展示列表」的例外
- 不知道 key 还会影响组件复用与性能

**解答：**
先给结论：key 用 index 在「顺序稳定、纯展示、无内部状态、无子组件 props」时基本没问题，但只要涉及增删排序或元素有内部状态（输入框、勾选、动画、懒加载），就会出 bug。原因：key 是 diff 时判断「两个节点是否同一节点」的依据，删除中间项后 index 整体前移，Vue 会误判「节点没变，只是位置变了」，于是复用组件实例，勾选状态、输入内容、动画全部跟着错位；同时「移动节点」被误判成「删除+新建」，性能也差。正确做法：用稳定且唯一的 id（后端主键、uuid、时间戳+随机），没有 id 时用「内容字段联合生成」；要避免用不唯一的字段（如 name）当 key，否则相同内容之间同样错乱。加分点是能结合 diff 的双端比较算法说明「为什么移动元素比删除重建更优」——用 id 时 Vue 能识别出元素只是换了位置，触发移动而非重建；以及主动说明在「只读日志流」「静态配置列表」这类场景下用 index 是合理的，体现边界意识。线上案例：表格行拖拽排序后样式错位、可勾选列表删除一行后勾选全部偏移，都是 index key 的典型症状。

**延伸考点：** 同一列表在「新增/删除」和「排序」两种操作下，key 的 diff 表现有什么不同？为什么说 key 相同但内容变化会触发「就地更新」？

### Q6. 自定义组件上实现 v-model，Vue 2 与 Vue 3 的差异

**问题：** 要封装一个「带单位的价格输入框」或「二次确认弹窗」，想通过 v-model 实现双向绑定，组件内部怎么写？Vue 2 和 Vue 3 有什么不同？Vue 3 支持多个 v-model 吗？

**期望加分项：**
- 说清 v-model 本质是语法糖：Vue 2 是 `value + input`，Vue 3 是 `modelValue + update:modelValue`
- 写出 Vue 3 组件内 `defineProps` + `emit('update:modelValue')` 的完整实现
- 知道 Vue 3 支持带参数的多 v-model（`v-model:start` / `v-model:end`），以及 `defineModel`（3.4+）简化写法
- 知道 Vue 2 用 `model` 选项自定义 prop/event 名（如 `value → checked`）
- 联系实践：封装金额格式化输入框、可编辑表格单元格时的选型

**减分项：**
- 只会在模板里用 v-model，写不出组件内的实现
- 不知道 Vue 3 改名为 `modelValue` 的原因
- 不知道多 v-model 的用法

**解答：**
v-model 本质是语法糖：Vue 2 中是 `value` prop + `input` 事件，所以封装组件时 props 写 `value`、内部 `this.$emit('input', v)`；Vue 3 改名为 `modelValue` + `update:modelValue`，原因是为了避免与原生 `value` attribute 冲突，并顺带支持了多个 v-model。Vue 3 组件内实现：`const props = defineProps(['modelValue'])`，`const emit = defineEmits(['update:modelValue'])`，内部 `<input :value="modelValue" @input="emit('update:modelValue', $event.target.value)">`；3.4+ 可以直接 `const model = defineModel()`，省去 props/emit 样板代码。带参数的多 v-model 在封装复杂控件时很实用，比如时间范围组件 `v-model:start` + `v-model:end`。坑有两个：一是 props 是只读的，直接给 `modelValue` 赋值会报错，必须通过 emit 更新，否则打破单向数据流；二是「内部先暂存、失焦再提交」的场景（如金额输入实时格式化）需要设计好「内部值 vs 提交值」两套状态，避免与 v-model 直接冲突。实践上，金额格式化输入框、远程搜索下拉、可编辑表格单元格都适合封装成 v-model 组件，把格式化、校验、空值处理统一收敛在一个地方，业务方只绑数据。

**延伸考点：** 为什么 Vue 3 把 v-model 的事件名改成 `update:modelValue` 而不是沿用 `input`？`defineModel` 相比手写 props/emit 有什么限制？

### Q7. 生命周期在真实场景中的应用：清理与竞态

**问题：** 页面里开了轮询定时器、绑了 window 的 resize 监听，还发了一个异步请求。切换路由后，旧页面还在发请求、监听还在触发，怎么处理？Vue 2 与 Vue 3 的写法差异是什么？

**期望加分项：**
- 能说出在 `onUnmounted`（Vue 3）/ `beforeDestroy`（Vue 2）里清理定时器、移除监听、取消请求
- 能说出 Vue 3 组合式 API 下把「创建 + 清理」封装成 `useInterval` 这类可复用 hook 的实践
- 能处理异步竞态：组件卸载后返回的请求不应再更新状态，用 `AbortController` 或请求序号
- 知道组件内 watch 卸载时自动停止，但事件总线订阅、防抖 setTimeout 闭包不会自动清理
- 联系线上案例：图表组件销毁后报 ResizeObserver 相关错误、切换路由后旧请求覆盖新数据

**减分项：**
- 不知道定时器/监听/第三方实例不会随组件销毁自动释放
- 没有「谁创建谁清理」的意识
- 说不出防抖闭包在组件卸载后仍会执行的问题
- 不会处理请求竞态

**解答：**
核心原则是「谁创建、谁清理」。定时器（setInterval）、全局事件监听（window.addEventListener）、第三方库实例（echarts 图表、地图）都不会随组件卸载自动销毁，必须在 `onUnmounted`（Vue 3）/ `beforeDestroy`（Vue 2）里显式清理，否则会内存泄漏、重复触发。Vue 3 组合式 API 的优势在于可以把「创建 + 清理」封装成可复用 hook：`useInterval(fn, ms)` 内部创建定时器并在 `onUnmounted` 里 `clearInterval`，任何组件直接复用，避免每个人漏写清理。异步请求是重灾区：组件卸载后请求才返回，Vue 3 里更新已卸载组件的响应式数据虽然不会报错（不同于 React），但会造成无效渲染与状态泄漏；更麻烦的是竞态——先发的 A 请求后返回，覆盖了后发的 B 请求的新数据。处理方案：`AbortController` 在卸载/重复触发时取消，或维护请求序号只采纳最新一次。还要注意细节：组件内 watch 会在卸载时自动停止，但事件总线（mitt）订阅必须手动 off；防抖/节流函数内部的 setTimeout 闭包在卸载后仍会执行，不要在回调里操作已销毁的 DOM 或更新组件状态。线上最常见的两类事故——「切换路由后旧页面还在轮询发请求」「图表销毁后报 ResizeObserver loop limit」——都是这个问题的变体。

**延伸考点：** vue-router 中同一个组件在不同参数下复用（如 `/user/1` 到 `/user/2`）时，生命周期不会重新执行，怎么处理？`onActivated`/`onDeactivated` 什么时候触发？

---

### Q8. 响应式深入：Proxy 的深层响应与性能开销

**问题：** 项目里有个几千条数据的大表格，用 `reactive` 包起来后初始化明显变慢，改成 `ref` 也一样慢。Vue 3 的 Proxy 响应式到底怎么做深层响应的？性能开销主要在哪儿？线上怎么降低这个开销？

**期望加分项：**
- 能说清 Proxy 的「懒代理」：get 时才对深层对象递归代理，set/delete 统一拦截，新增属性天然响应，初始化成本远低于 Vue 2 的深度递归劫持
- 能讲清开销来源：每次属性读写都经过 Proxy 拦截与 track/trigger 依赖收集，高频读路径（渲染循环、大数组遍历）会放大开销
- 知道 `reactive` 不适用于频繁读写超大对象，会用 `markRaw`/`shallowRef`/`shallowReactive` 跳过代理
- 知道 `ref` 对对象值内部也是走 `reactive`（`toReactive`）代理，不是「ref 更轻」
- 能先谈数据源头优化（分页、虚拟滚动），再谈代理层优化
- 能结合 Devtools/Performance 定位「响应式读写在渲染热路径被反复触发」

**减分项：**
- 以为 Vue 3 响应式没有任何性能开销
- 不知道 `shallowRef`/`markRaw` 的存在及适用场景
- 把 `ref` 与 `reactive` 的实现割裂看待
- 大数据量卡顿只甩锅给渲染，不排查响应式层

**解答：**
先理清 Vue 3 的实现：`reactive` 基于 Proxy，`set`、`deleteProperty`、`has` 等操作统一拦截，`get` 时若访问的属性值是对象，会惰性地把它也代理——访问到才代理，所以深层嵌套天然响应、新增属性天然响应，这是相比 Vue 2 在初始化时深度递归 defineProperty 的进步，初始化几乎零成本。代价是每次属性读写都要经过 Proxy 拦截与 track/trigger 依赖收集，性能开销集中在三处：一是高频读路径，比如渲染函数里遍历大数组，每个属性访问都触发 track；二是大对象被深度代理后，写入触发 trigger 级联更新；三是把不该响应的数据（echarts 实例、WebSocket、大图片二进制）包进响应式对象，白白被代理，还会拖慢每次更新的深度遍历。工程降耗三板斧：`shallowRef`/`shallowReactive` 只做浅层响应，配合 `triggerRef` 手动刷新；`markRaw` 标记第三方实例或一次性大数据；以及数据源头优化——大表格改后端分页或虚拟滚动，数据量降下来比任何代理层优化都有效。`ref` 的注意点：`ref` 存对象时内部同样通过 `toReactive` 走 `reactive` 代理，所以「ref 存大对象」的开销与 reactive 一致，`ref` 并不更轻。线上排查建议：先看 Devtools 响应式追踪或 Performance 面板，确认是不是响应式读写在渲染热路径被反复触发，再对症下药，避免「凭感觉优化」。

**延伸考点：** 为什么说 Vue 3 的响应式是「访问时惰性代理」？`shallowRef` 的值整体替换与内部属性修改，视图更新行为有什么不同？

---

### Q9. computed 的缓存机制与依赖收集

**问题：** 一个列表页里 `filterList` 依赖 keyword、tag、sort 三个状态，被模板和多个方法反复使用。为什么用 computed 而不是每次调用一个方法现场计算？computed 的缓存到底怎么实现的？依赖变化时它怎么知道要重算？

**期望加分项：**
- 能说出 computed 基于「依赖收集 + 脏标记」：getter 执行时收集读到的响应式依赖，任一依赖变化时标记 dirty，下次读取才重算
- 知道缓存失效是「惰性」的：依赖变了不会立即重算，而是下次被读取且 dirty 为 true 时才执行 getter
- 知道 computed 的依赖是属性级别的，而不是对象级别（对比 watch 的 deep 粗粒度）
- 知道 computed 可以依赖其他 computed/ref，依赖图逐层传播脏标记
- 主动说出「副作用放进 computed 是反模式」：发请求、改其他状态会让重算时机不可控
- 知道模板里 `{{ filterList }}` 与 JS 里 `filterList.value` 的缓存行为完全一致

**减分项：**
- 只会背「computed 有缓存」，讲不出脏标记机制
- 在 computed 里发请求、修改其他状态
- 以为 computed 依赖变化时会立即重算
- 不知道 computed 里读不到依赖（如 Date.now()、Math.random()）就永远不会重算

**解答：**
computed 的本质是「带缓存的响应式计算」，依赖两个机制：依赖收集（track）与脏标记（dirty）。第一次访问 `.value` 时执行 getter，getter 内部读到的每个响应式属性都被记录为该 computed 的依赖——注意依赖是属性级别的，读 `state.list` 只依赖 `list` 这一个 key，不会因为对象上其他键变化而失准，这点比 watch 的 deep 监听更精细；同时这个 computed 自身也会被收集进使用它的 effect（渲染函数、父 computed、watch）的依赖里。之后依赖属性被修改时，Vue 并不是立即重算，而是把 dirty 标记置为 true——重算是惰性的，只有再次被读取且 dirty 为 true 时才执行 getter 并缓存结果。这带来两个关键收益：一是无论模板、方法还是其他 computed 读多少次，只要依赖没变，getter 只执行一次；二是依赖变更后的第一次读取「补算」，其余读取命中缓存。工程上的坑：① 在 computed 里放副作用（发请求、改其他 ref）会破坏「纯计算」约定，重算时机不可控，严重时还会形成无限循环；② 依赖了非响应式的可变值（`Date.now()`、`Math.random()`、外部可变对象）永远不会触发重算，因为根本没进入响应式依赖，排查「computed 不更新」时先检查这里；③ computed 可以依赖其他 computed，脏标记会沿依赖图逐层传播，形成计算链。另一个常见疑问：模板中 `{{ filterList }}` 只是自动解包，与 JS 里读 `filterList.value` 的缓存行为完全一致。

**延伸考点：** computed 与 watchEffect 在依赖追踪方式上有何不同？computed 的 getter 抛出异常或依赖的 ref 为 undefined 时，组件渲染会怎样？

---

### Q10. 插槽：默认/具名/作用域插槽与编译时机

**问题：** 要封装一个通用表格组件，列头、单元格、空状态都允许外部定制；另一个场景是列表卡片想复用同一套布局但内容不同。插槽怎么设计？默认/具名/作用域插槽分别怎么用？插槽里的内容到底是什么时候编译的？

**期望加分项：**
- 说清三种插槽的适用边界：默认插槽做「整块内容替换」、具名插槽做「多区域定制」、作用域插槽做「子组件数据回传父组件定制渲染」
- 能写出作用域插槽的完整用法：子组件 `<slot :row="row" :index="index">`，父组件 `<template #default="{ row, index }">`
- 说清编译时机：插槽内容在**父组件作用域**编译，能访问父组件变量但不能直接访问子组件 data，子组件数据必须通过作用域插槽 props 显式传回
- 知道动态插槽名 `<template v-slot:[name]>` 的用法
- 能结合实践：表格列定制用「具名 + 作用域」组合、空状态/加载中用具名插槽保留扩展位
- 有「不要过度开插槽」的意识：能用 props 配置解决的不要开插槽

**减分项：**
- 分不清三种插槽的适用场景
- 不知道插槽内容属于父组件作用域，误以为能直接用子组件的数据
- 不知道 `v-slot` 只能用在 `<template>` 或组件标签上
- 把插槽当全局状态透传工具滥用

**解答：**
插槽是组件「内容定制」的三大手段，本质是把父组件的渲染内容「插进」子组件的布局位。默认插槽适合整块内容替换（弹窗内容、卡片 body）；具名插槽适合一个组件有多个定制区域（表格的 `#header`、`#footer`、`#empty`，用 `<template #header>` 写法）；作用域插槽解决「布局在子组件、数据也在子组件、但渲染细节由父组件决定」的场景——子组件通过 `<slot :row="row" :index="index">` 把数据传出，父组件用 `v-slot="{ row, index }"` 接收后决定怎么渲染，实现「子组件管数据、父组件管样式」的解耦。关键认知是编译时机：插槽内容在**父组件作用域**里编译，所以插槽里能直接访问父组件的变量，但不能直接访问子组件的 data——子组件的数据必须通过作用域插槽的 props 显式传回来，这是单向数据流在插槽上的体现；渲染时子组件通过 `renderSlot`（渲染函数中）把插槽内容挂到对应位置。工程实践：封装表格时用「具名 + 作用域插槽」组合（如 `#cell-{field}` 动态插槽名定制任意单元格）就能支持任意列的定制而不必暴露整行结构；空状态、加载中、分页用具名插槽保留扩展位。两个常见坑：一是过度设计——每个地方都开插槽会让组件 API 膨胀难维护，能用 props 配置解决的（如 `align: 'center'`）就不要开插槽；二是作用域插槽传出的字段命名要规范统一（`row`/`index`/`data`），否则使用者心智负担重。性能上注意：插槽内容会随父组件更新范围联动，若父组件数据频繁变化而子组件是重组件，可考虑把插槽内容抽成轻量组件隔离更新。

**延伸考点：** 动态插槽名 `<template v-slot:[dynamicName]>` 适合什么场景？父组件数据变化导致插槽内容更新时，会连子组件本身也重新渲染吗？

---

### Q11. 虚拟 DOM 与 diff 优化：patch 过程与静态节点提升

**问题：** 一个长列表页面渲染很慢，你怀疑是 diff 太慢。Vue 3 的 diff 到底优化了什么？静态节点提升、patch 过程是怎么回事？为什么这些优化只有 Vue 3 的模板编译器能做，Vue 2 做不了？

**期望加分项：**
- 先纠正误区：虚拟 DOM 的价值是跨平台与「用最小 diff 换取可维护性」，不是「一定比直接操作 DOM 快」
- 能讲 Vue 3 的编译时优化：静态节点提升（hoistStatic）、静态标记（patchFlag）、事件缓存（cacheHandlers）、Block 树动态节点收集
- 说清 patchFlag 的作用：编译期标记节点「只更新 text/只更新 props/只更新 class」，diff 时精确到属性，跳过整棵子树比较
- 能对比 Vue 2 的运行时全量 diff 与 Vue 3 的「动静分离」
- 能结合实践：模板 vs 手写 render 函数 vs JSX 在编译优化上的差异
- 提到 `v-memo`、`v-once` 等手动优化手段

**减分项：**
- 以为虚拟 DOM 一定比原生操作快
- 讲不出任何编译时优化细节，只会说「Vue 3 更快」
- 不知道 patchFlag/静态节点提升
- 分不清编译时优化与运行时 diff 的关系

**解答：**
先纠正常见误区：虚拟 DOM 的价值不是「比直接改 DOM 快」，而是跨平台（渲染器可替换）与「用可控的 diff 成本换取声明式开发的维护性」，真正省性能的是编译期把「该比的、不该比的」提前区分开。Vue 2 的 diff 是运行时全量：每次更新都创建整棵 vnode 树并逐节点对比，模板里大段静态内容也参与比较。Vue 3 的模板编译器引入了多项编译时优化：① 静态节点提升——模板中完全静态的节点（常量文本、固定结构）只创建一次 vnode 并缓存，重渲染直接复用，不再参与 diff；② 静态标记（patchFlag）——编译时分析每个动态节点「只有 class 变 / 只有 text 变 / 只有 props 变」，diff 时按 flag 精确更新对应属性，跳过子树的深度比较；③ 事件缓存——`@click` 这类 handler 用 `cacheHandlers` 缓存，避免每次渲染生成新函数导致依赖 props 的子组件被迫更新；④ Block 树——模板被编译成带动态节点索引的块（Block），更新时直接按索引定位动态节点，绕开整棵树遍历。这些是「模板编译器能做的、手写 render 函数做不到的」优化，这也是官方推荐模板优先的原因——手写 render/JSX 时这些优化会失效，只能靠手动 patchFlag 或 v-memo 兜底。工程建议：性能出问题时先用 Performance 定位是否真的卡在 diff，很多「慢」其实是数据量大、重渲染频繁或响应式读写在热路径，diff 只是背锅；确需优化时优先「减少不必要更新」（拆分组件、稳定 key、v-once、v-memo），再考虑更深层的渲染器级优化。

**延伸考点：** v-once 与静态节点提升有什么区别？为什么说「手写渲染函数越多，编译优化就越少」？

---

### Q12. 列表渲染性能优化：虚拟滚动与分片渲染

**问题：** 一个表格要展示 2 万行数据，每行还有图片和联动逻辑，直接 v-for 渲染会卡死。你有哪些手段？虚拟滚动和分片渲染的原理分别是什么？各自适用什么场景？

**期望加分项：**
- 给分层方案：先数据源头优化（后端分页/筛选/懒加载），再渲染层优化，这是成本最低的路径
- 说清虚拟滚动原理：只渲染可视区 + 缓冲区的行，用空白撑起总高度，滚动时按 scrollTop 换算起始索引更新渲染区间
- 说清分片渲染（time slicing）原理：用 requestIdleCallback/分帧把大批量渲染拆成小批次，避免长时间占用主线程
- 知道两者边界：虚拟滚动适合「行高可估算、数量极大、可预测窗口」；分片渲染适合「一次性灌入但总量可控、无滚动交互」的初始化场景
- 知道用库 vs 自研：成熟库（vue-virtual-scroller）处理了变高、滚动方向、首屏定位等边界，自研要踩的坑很多
- 有性能观测意识：用 Performance 面板确认瓶颈是渲染/重排而不是网络或响应式，再决定手段

**减分项：**
- 只会答「用虚拟滚动」但讲不清为什么
- 把分片渲染和虚拟滚动混为一谈
- 不考虑行高变化（图片未加载导致高度抖动）对虚拟滚动的破坏
- 直接上库而不理解其原理，出问题无法排查
- 不做数据源头优化，直接在最底层渲染层硬扛

**解答：**
先分层想，别一上来就上虚拟滚动：2 万行里用户真正会看的前 100 行，**后端分页/筛选/懒加载把数据量降下来**是成本最低、收益最大的手段——数据只有 200 条时根本不需要任何渲染优化。确定数据量无法减少后再看渲染层。**虚拟滚动**解决的是「DOM 数量过多」：浏览器真实渲染 2 万行 DOM 本身就慢（布局、样式、合成全是 O(n)），虚拟滚动只渲染可视区加少量缓冲（如 20 行），用一个空白占位容器撑起总滚动高度，监听 scrollTop 换算起始索引、slice 出可视窗口的行渲染，行高固定时实现简单（`start = floor(scrollTop / rowHeight)`）；行高不固定（内容撑开、图片未加载）就麻烦——需要估算高度、渲染后再修正并补偿位移，这是自研最易出错的地方。**分片渲染（time slicing）**解决的是「一次性大任务阻塞主线程」：比如列表初始化时一次性 push 几千条数据进响应式数组触发整批重渲染，用 `requestIdleCallback` 或定时器把渲染拆成每帧几十条的小批次，让浏览器有空处理输入与绘制——但它不减少 DOM 总量，只摊平时间，适合「初始化灌数据」场景，不适合持续滚动。两者边界清晰：虚拟滚动管「数量极大且持续滚动浏览」，分片渲染管「一次性灌入但总量可控」。配套优化要主动提：大数据用 `shallowRef` 避免深层代理 + `Object.freeze` 冻结静态数据、避免对整表 deep watch、图片加固定尺寸占位。工程上优先用成熟库（vue-virtual-scroller / element-plus 虚拟表格），它们的边界处理（变高、滚动定位、首屏预载）远超自研。最后用 Performance 面板看长任务分布，确认瓶颈真在渲染层，避免误伤。

**延伸考点：** 虚拟滚动里「滚动到底部加载更多」与「快速跳转到第 5000 行」分别需要哪些额外处理？行高不固定时，估算高度与实际高度的偏差怎么补偿？

---

### Q13. 组件里要「监听状态变化做异步副作用」，watch / watchEffect / watchPostEffect 怎么选？

**问题：** 搜索框输入关键词，要防抖后请求接口；还要在 `userId` 变化时重新拉取用户详情并取消旧请求。这类「响应状态驱动的副作用」在 Vue 3 里用 watch 还是 watchEffect？它们和 computed 的分工边界在哪？

**期望加分项：**
- 能说清三者分工：computed 是「派生纯值」（缓存 + 依赖追踪）；watch 是「显式监听 + 副作用」；watchEffect 是「自动收集依赖的副作用」，立即执行一次
- 能讲 watch 的关键配置：`immediate`（立即执行）、`deep`（深层监听，注意性能）、`flush`（'pre'/'post'/'sync' 控制回调时机）
- 能讲 watchEffect 的适用：不关心具体哪个依赖变了、只要「任意依赖变化都重新执行」；以及 `onCleanup`/回调里的清理函数处理竞态
- 能给出防抖的落地写法：watch 内 setTimeout 包裹、或用 watch + 防抖函数
- 能指出 watch 监听「响应式对象属性」的写法差异：`watch(() => props.userId, ...)` 而非直接传 props.userId
- 知道 `flush: 'post'` 用于在 DOM 更新后再操作（watchPostEffect），避免读到旧 DOM

**减分项：**
- 把 watch 和 computed 混为一谈（在 computed 里写副作用）
- 不知道 `immediate` 配置，实现「初始化也执行」时用笨办法
- 监听 props 属性时传 `props.userId`（值）而不是 getter，导致不触发
- 防抖/节流实现里没考虑竞态（旧请求覆盖新请求）
- 不知道 watchEffect 与 watch 的依赖追踪差异

**解答：**
Vue 3 里三个 API 的分工：**computed** 是纯派生（输入依赖 → 输出值，带缓存），任何有「返回值、被模板/其他逻辑多次使用」的需求都该用它，副作用一律不该进 computed。**watch** 是「显式声明要监听什么，变化后执行副作用」，适合「关心具体数据 + 要做异步操作 + 需要精确控制（立即执行/深层/时机）」的场景，写法是 `watch(source, cb, options)`，source 可以是 ref、getter、数组；两个高频坑：一是监听 props 属性必须写 getter（`watch(() => props.userId, ...)`），直接传 `props.userId` 会把**当前值**传进去而不是「可响应来源」；二是初始化不执行，需要 `immediate: true` 才在挂载时立即跑一次。**watchEffect** 是「自动收集副作用里读取的所有响应式依赖，任一变化重新执行，且立即执行一次」——它不关心「哪个变了」，只要求「变了就重跑」，适合「依赖不固定、只是要做一件事」的场景（如同步一份 state 到 localStorage、渲染后测量）。防抖落地：watch 回调里 `clearTimeout + setTimeout`，或把防抖函数包一层；竞态处理用回调的第三参数 `onCleanup`（watchEffect 的 onCleanup 或 watch 回调里的 onCleanup 注册函数），旧请求返回时通过 `aborted` 标志丢弃，保证「后发的先返回」不会覆盖新数据。`flush` 时机：默认 'pre'（DOM 更新前回调）、'post'（DOM 更新后，读 DOM 尺寸用）、'sync'（同步，慎用）。工程经验：能用 computed 派生的值用 computed；副作用分两类——「数据变化驱动」（watch/watchEffect）+「用户交互驱动」（事件回调），前者需要清理与竞态控制，后两者是 watch 的主要战场。

**延伸考点：** watchEffect 里读取了 `props.userId` 和 `route.query.page`，两次变化只触发一次回调还是两次，为什么？`flush: 'sync'` 在什么场景下必须用、什么场景下用了会踩坑？

---

### Q14. Pinia 与 Vuex 的差异，以及状态「该放 store 还是该放组件」的边界。

**问题：** 团队要把 Vuex 4 迁移到 Pinia，或者新项目直接选型。候选人要讲清：Pinia 相比 Vuex 的差异（API、TS 支持、模块化、devtools）、以及一个更本质的问题——哪些状态应该全局共享、哪些应该留在组件内部？

**期望加分项：**
- 能说 Pinia 核心差异：没有 mutations（直接改 state）、store 即模块（无需 modules 嵌套）、完整 TS 推断、支持 setup store 与 options store、devtools 时间旅行
- 能讲 `defineStore` 的两种写法：options store（state/getters/actions）与 setup store（组合式），以及 setup store 适合复杂逻辑复用
- 能讲 store 之间互相引用（在 action 里用另一个 store）、`storeToRefs` 保持响应性
- 核心认知：全局 store 的边界——「跨组件共享、跨页面持久、多组件共同消费的数据」放 store（用户信息、权限、购物车）；「只服务单个组件/单页的局部状态」留在组件里（表单输入、列表过滤）
- 能讨论「过度集中」的坏处：所有状态都进 store 会让组件失去内聚、store 变成大杂烩
- 能结合实践：按领域拆 store（user store、cart store），避免单 store 无限膨胀；持久化用插件（pinia-plugin-persistedstate）或手动

**减分项：**
- 只会背「Pinia 比 Vuex 好」，说不出具体差异
- 不知道 setup store 与 options store 的区别与适用
- 把只属于单个组件的表单状态也放进全局 store
- 不会拆 store，一个 store 装下所有模块
- 忽略 Pinia 的 devtools 与调试能力

**解答：**
先讲技术差异：Pinia 相对 Vuex 的变革是「移除 mutations」——state 直接可写，actions 里直接 `state.x = 1`，少一层概念；模块化用「多 store 平铺」替代 Vuex 的 modules 嵌套（每个 store 就是独立的 `useXxxStore()`），不再有命名空间与 getter 前缀的心智负担；TypeScript 推断是完整的一等公民（state/actions 全类型推导）；支持 **options store**（state/getters/actions 选项式）与 **setup store**（组合式函数风格，逻辑更灵活，可复用跨 store 的工具函数）两种写法；devtools 支持时间旅行。常用细节：跨 store 引用——在 action 里 `const other = useOtherStore()` 直接调用；模板里为了保持响应性用 `storeToRefs(store)` 解构 state/getters（直接解构会丢失响应性，store 本身除外）。**更本质的边界问题是状态分层**：全局 store 只放「真正需要跨组件/跨页面共享」的状态——用户登录态与权限、购物车、全局配置、多页面复用的缓存数据；而「单个组件内、随组件生命周期销毁」的状态（表单输入、列表筛选条件、下拉展开状态）必须留在组件 `ref/reactive` 里。判断标准三条：是否被 2 个以上无直接父子关系的组件消费？是否跨路由需要保持？是否需要持久化？全否 → 留在组件。过度集中的坏处要说出来：store 变成「所有数据的垃圾桶」后，组件失去了局部状态的内聚性，store 修改时所有依赖它的组件都受影响，调试定位变难。落地建议：按业务域拆 store（user/cart/order...），每个 store 职责单一；持久化用成熟插件而非手写；action 里放异步与业务编排，getter 放派生。

**延伸考点：** Pinia 的 setup store 与「组合式函数 + 局部状态」的本质区别是什么，什么情况下该用 store 而不是 composable？`storeToRefs` 与直接解构 store 的响应性差异是什么原因？

---

### Q15. 要给「按钮防重复点击」「输入框自动聚焦」这类横切需求做统一封装，自定义指令怎么做？

**问题：** 项目里多个页面都遇到：按钮要防连点（提交中禁用）、弹窗打开后输入框要自动聚焦、图片加载失败要替换默认图。这类「横切 DOM 行为」在 Vue 里用什么封装最合适？自定义指令的钩子函数、以及和「组件封装」的分工怎么定？

**期望加分项：**
- 能说自定义指令适合「纯 DOM 行为增强」（focus、防抖、点击外部关闭、权限隐藏），与组件封装（有模板/有内部状态/有交互逻辑）分工明确
- 能写出指令钩子：`created`/`mounted`/`updated`/`unmounted`（Vue 3）与 Vue 2 的 bind/inserted/update/unbind 对应
- 能实现典型指令：v-focus（mounted 时 el.focus）、v-throttle/v-debounce（包装 click）、v-click-outside（document 监听 + unmounted 清理）、v-permission（无权限 vnode 移除）
- 能说指令的清理义务：mounted 里注册的全局监听必须在 unmounted 里移除，否则内存泄漏与重复触发
- 知道指令修改行为（`el.addEventListener`）与修饰符、binding 参数的取值
- 能讲「指令 vs 组件」的边界：需要模板/插槽/响应式复杂状态 → 组件；只操作 DOM 属性与事件 → 指令

**减分项：**
- 所有横切需求都用指令（把复杂组件逻辑塞进指令）
- 忘了在 unmounted 清理全局监听
- 不知道 Vue 3 钩子名与 Vue 2 的对应关系
- 指令里直接改 vnode 或依赖内部状态（指令应该无状态、操作 el）
- 不会用 binding 参数（value/modifiers/arg）传递配置

**解答：**
自定义指令的本质是「对 DOM 元素的横切行为增强」，Vue 3 的指令钩子与组件生命周期对应：`created`（元素创建）、`beforeMount`/`mounted`（挂载，此时可操作 el）、`beforeUpdate`/`updated`、`beforeUnmount`/`unmounted`（卸载，**必须在这里清理监听**）。注册方式：局部 `directives: { focus: {...} }` 或全局 `app.directive('focus', {...})`。典型实现：v-focus 在 `mounted` 里 `el.focus()`；v-throttle 包装 click——`mounted` 时 `el._original = el.onclick`，用节流函数替换 `el.onclick`，`unmounted` 还原；v-click-outside——`mounted` 里 `document.addEventListener('click', handler)` 判断点击目标是否在 el 外，`unmounted` 移除监听（**漏掉这一步 = 内存泄漏 + 其他页面误触发**）；v-permission——`mounted` 里根据 binding.value 检查权限，无权限用 `el.parentNode.removeChild(el)` 移除。参数通过 `binding` 拿到：`binding.value`（指令值）、`binding.arg`（`v-foo:bar` 中的 bar）、`binding.modifiers`（`.stop` 等）。**分工边界是最关键的**：指令只做「无状态的 DOM 行为」——聚焦、防抖、节流、点击外部、权限控制、图片兜底；凡是「需要模板、插槽、props、复杂响应式状态、内部交互逻辑」的都应该是**组件**（如防重复提交按钮可以做成 `Button` 组件，用 props/loading 控制，而不是指令）；指令代码必须无状态——不要依赖组件 data、不要在指令里管理复杂业务数据，否则难以调试与复用。工程落地：指令按需注册、集中管理（directives/ 目录），配套单元测试（用 jsdom 挂载 el 测行为），避免指令内写死业务逻辑（通过 binding.value 传入配置保持通用）。

**延伸考点：** v-click-outside 里「点击了指令元素内部但不是某个子元素」怎么判断（composedPath）？指令的 `mounted` 里操作 `el` 和操作 `vnode.el` 的区别，以及为什么 Vue 3 推荐操作 el 而非 vnode？

---

### Q16. 组合式函数（composables）怎么设计才能既复用又不变乱？无渲染组件 vs composable 怎么选？

**问题：** 团队引入组合式 API 后，出现了「useUser」「useTable」「useRequest」等一堆 composable，有些相互调用、有些和 store 纠缠。候选人要讲：设计组合式函数的原则（单一职责、参数化、返回 ref 响应性）、以及「无渲染组件（renderless）」与 composable 的取舍。

**期望加分项：**
- 能说 composable 的本质：复用「逻辑」而非「模板」，返回 ref/computed/函数，调用方决定怎么渲染
- 设计原则：单一职责（一个 useXxx 只做一件事）、参数化（接收外部状态而非硬编码）、命名约定（use 前缀）、返回解构出的响应式值
- 能讲 composable 里怎么安全使用生命周期：只能在 setup 顶层调用、不能条件调用
- 能讲 composable 与 store 的分工：跨组件共享的全局状态 → store；只服务一个组件族的可复用逻辑 → composable
- 能讲无渲染组件：通过作用域插槽暴露逻辑与数据（`<DataProvider v-slot="{ data }">`），适合「模板结构需要外部完全定制」的场景，但心智负担比 composable 重
- 会做取舍：新代码优先 composable；无渲染组件留给「既复用逻辑又要复用 UI 骨架」的旧场景

**减分项：**
- 把 composable 当普通函数用（在 setup 外调用、或条件调用）
- composable 里管理全局状态（应放 store）
- 无渲染组件与 composable 分不清，选型随意
- composable 过大、职责混乱（一个 useAll 干所有事）
- 返回大量 ref 却不组织（返回对象被解构后失去响应性？——返回 ref 是安全的）

**解答：**
组合式函数（composable）是 Vue 3 复用「逻辑」的标准形态：**它返回响应式值/函数，调用方决定渲染结构**。设计原则：一是单一职责——一个 useXxx 只负责一个领域（useDebounce 只管防抖、useTable 只管表格数据流），绝不做一个「useAll」；二是参数化——接收外部 ref/props 作为输入（如 `useTable(fetchFn, { pageSize })`），不硬编码内部状态，输入变化时 composable 内部用 watch 响应；三是命名与调用约束——必须以 `use` 开头、**只能在组件的 setup 顶层同步调用**（不能放 if/循环里、不能异步后再调），因为内部要注册生命周期钩子，这个约束是响应式依赖收集正确性的前提；四是返回组织——返回 `{ data, loading, error, refresh }` 这类对象，解构后仍是 ref 保持响应性。与 store 的分工：composable 是「无状态/实例级」的可复用逻辑（每个组件调用一次就有自己的一份状态），跨组件共享的全局状态（用户信息、权限、购物车）应该进 store；「useRequest/useTable」这类完全服务单个组件的逻辑用 composable。**无渲染组件（renderless）**是另一种复用形态：组件自身不渲染任何 DOM，只通过作用域插槽把逻辑与数据暴露给父组件（`<DataProvider v-slot="{ data, loading }">...</DataProvider>`），它复用「逻辑 + 插槽结构」，但父组件每次用都要写一遍插槽模板，心智负担和样板代码都更重。取舍结论：新代码、逻辑复用优先 composable（更轻、TS 友好、组合自由）；无渲染组件适合「必须通过模板位置编排逻辑」或需要兼容旧模式的场景。工程实践：composable 目录按域组织、内部用 `onUnmounted` 清理资源、对第三方副作用（事件监听/定时器）保持「谁创建谁清理」。

**延伸考点：** composable 内调用了另一个 composable（组合）和「一个 composable 复用另一个的状态」有什么区别？为什么 composable 必须在 setup 顶层调用——如果违背了会发生什么具体错误？

---

### Q17. keep-alive 缓存组件导致「数据不刷新」，候选人怎么理解它的机制并给出刷新方案？

**问题：** 列表页进详情页再返回，用户希望列表滚动位置和筛选条件保留；但另一个场景里，keep-alive 缓存的组件在每次进入时数据没刷新，业务要求每次进入都拉最新数据。候选人对 keep-alive 的缓存机制、include/exclude/max、以及「缓存 vs 刷新」的平衡分别怎么处理？

**期望加分项：**
- 能说 keep-alive 原理：缓存组件的 vnode 与实例，切走时不销毁、切回时不重建（渲染时直接从缓存取），配合 `onActivated`/`onDeactivated` 生命周期
- 能讲 include/exclude（按组件 name 匹配）与 `max`（最大缓存数，超出按 LRU 淘汰）
- 能讲「需要刷新」的场景：在 `onActivated` 里拉数据（每次重新激活都执行），而不是只放在 onMounted（只在首次执行）
- 能讲「需要保留状态」的场景：筛选条件/滚动位置天然保留，无需额外处理；配合 `activated` 恢复
- 能结合 vue-router 场景：`<router-view v-slot="{ Component }"><keep-alive :include="cacheList"><component :is="Component"/></keep-alive></router-view>`
- 能讲边界：动态修改 include 列表控制哪些页面被缓存；缓存过多导致内存压力用 max 限制

**减分项：**
- 不知道 keep-alive 是缓存 vnode/实例而非重新渲染
- 把 onActivated 与 onMounted 混淆（onMounted 在缓存复用时不会再次触发）
- include 匹配的 name 写错导致缓存不生效
- 所有页面都无脑缓存（内存/过期数据问题）
- 不知道 max 的 LRU 淘汰机制

**解答：**
keep-alive 的机制是**缓存「渲染过的组件实例与 vnode」**：被包裹的组件切走时不走卸载、切回时直接从缓存里复用实例（不重新 created/mounted，也不重新渲染模板——除非内部状态变化）。对应新增了两个生命周期：`onActivated`（每次进入缓存组件时调用）与 `onDeactivated`（切走时调用）。配置：`include`/`exclude` 用组件的 **name**（注意是组件的 name 选项，不是路由名）做匹配，支持数组/正则；`max` 限制最大缓存实例数，超出时按 LRU 淘汰最久未用的——防止长期浏览导致缓存膨胀。两个典型场景的处理：**保留状态**（列表页返回时筛选条件/滚动位置还在）——这是 keep-alive 的默认行为，无需额外代码，滚动位置如果丢失可以在 deactivated 时记录、activated 时恢复；**每次进入都刷新数据**——把拉数据逻辑从 `onMounted` 移到 `onActivated`：onMounted 只在首次创建时执行一次，之后切回缓存组件不会再触发；onActivated 每次激活都执行，配合一个「首次加载 vs 刷新」的判断（或始终刷新）。vue-router 集成写法：`<router-view v-slot="{ Component }"><keep-alive :include="keepAliveList"><component :is="Component"/></keep-alive></router-view>`，keepAliveList 动态维护（进入某页加 name、离开移除）即可实现「只缓存指定页面」。工程建议：不要全站无脑缓存——只对「高频返回且状态有意义」的页面缓存（列表页、Tab 页），表单页/数据强实时页面排除；缓存页面要警惕「数据过期」，在 activated 里决定刷新策略；排查「keep-alive 不生效」先检查 name 是否与 include 匹配、是否通过 `<component :is>` 正确包裹。

**延伸考点：** keep-alive 缓存的是实例，那缓存页面的组件内部 watch 还会触发吗（切走期间数据变化）？`onActivated` 里拉数据与 onMounted 里拉数据在「首次进入」时都会执行吗，有什么顺序差异？

---

### Q18. 要做一个 SSR 或 SSG 项目，Vue 的同构渲染（hydration）原理与坑有哪些？

**问题：** 团队要提升首屏速度与 SEO，考虑对核心页面做 SSR（服务端渲染）。候选人要讲：Vue SSR 的基本流程、hydration（水合）的原理、以及服务端渲染带来的典型问题（内存、事件绑定、第三方库、构建配置）怎么规避。

**期望加分项：**
- 能讲 SSR 流程：服务端把组件渲染成 HTML 字符串返回，客户端「水合」（复用已有 DOM 并挂载事件与响应式）而不是重新渲染整页
- 能讲 hydration 原理：客户端渲染函数渲染出 vnode 树，与已有 DOM 节点比对后 attach 事件、激活响应式，避免首屏白屏
- 能讲典型坑：服务端没有浏览器 API（window/document 不可用）——第三方库要判断环境或在 mounted 再初始化；请求数据要在服务端 pre-fetch（`onServerPrefetch`）；水合不一致（SSR HTML 与客户端首次渲染不同）会产生警告与错误
- 能讲构建：Vite + ViteSSR / Nuxt 3 生态，`client`/`server` 两个入口、路由与 store 要在服务端创建（避免跨请求状态共享）
- 能讲 SSG 与 SSR 的取舍：静态页面用 SSG（构建时生成），动态内容用 SSR
- 性能：注意服务端渲染是同步阻塞 IO 密集，要做好缓存与降级（CSR fallback）

**减分项：**
- 分不清 SSR / SSG / CSR 的适用边界
- 不知道 hydration 是「复用服务端 HTML」而不是重新渲染
- 在服务端代码里直接使用 window/document
- 不处理跨请求状态共享（store 被多用户复用污染）
- 不知道水合不一致的后果与排查方向

**解答：**
SSR 的核心是「首屏由服务端输出 HTML，客户端负责水合」。流程：服务端用 `renderToString` 把根组件渲染成 HTML 字符串返回（首屏内容直接可见，利于 SEO）；浏览器拿到 HTML 后，客户端用同一套组件树做 hydration——**不再重新渲染 DOM，而是把已有的静态 HTML 节点「激活」**：对比 vnode 树、绑定事件、建立响应式联系，所以首屏不白屏、不闪烁。关键坑与对策：一是**环境差异**——服务端没有 `window`/`document`，第三方库（图表、滚动库、需 DOM 的插件）不能直接 import 执行，要动态 import 或在 `onMounted` 后再初始化；二是**数据预取**——首屏数据必须在服务端拿到并注入（Vue 3 用 `onServerPrefetch`），否则客户端水合后才发现无数据、又要发请求，失去 SSR 意义；三是**水合不一致**——服务端渲染的 HTML 与客户端首次渲染结果必须一致，不一致会抛 hydration mismatch 警告（如 `Date.now()`、`Math.random()`、根据 UA 渲染的内容），排查方向是消除渲染路径上的非确定性；四是**状态隔离**——每次请求都要新建 store/router 实例（`createApp` 工厂函数），绝不能把单例 store 挂在模块顶层，否则多用户共享同一状态（严重的串号事故）；五是构建——ViteSSR 或 Nuxt 3（Nuxt 对 SSR/SSG 一体化封装，推荐），需要 client/server 双入口与对应构建产物。SSG 与 SSR 取舍：内容静态（文档、博客、营销页）用 SSG 构建期生成、CDN 缓存、成本最低；动态数据页面才用 SSR；纯工具型应用可保持 CSR。生产注意：SSR 服务是 CPU/内存密集型，要配缓存（页面级/接口级）、限流、以及 CSR fallback（服务端异常时退化为客户端渲染）保证可用性。

**延伸考点：** hydration 时「复用 DOM」和客户端重新渲染有何性能与体验差异（闪烁、事件丢失）？`onServerPrefetch` 与 `onMounted` 在 SSR 里的执行时机分别是什么？

---

### Q19. 项目要从 Vue 2 迁移 Vue 3，候选人的迁移清单与风险控制怎么安排？

**问题：** 老项目用 Vue 2 + Vuex + vue-router 3 + Element UI，要迁到 Vue 3 + Pinia + Element Plus。候选人要给出：迁移的破坏性变更清单（API、响应式、生命周期）、渐进迁移还是全量重写的取舍、以及迁移过程中的兼容与验证策略。

**期望加分项：**
- 能列核心破坏性变更：全局 API 改为实例上（`Vue.use`→`app.use`）、`Vue.prototype`→`app.config.globalProperties`、`$on/$off/$once` 移除、`filter` 移除、`slot`/`slot-scope` → `v-slot`、`v-model` 行为变化、`keyCode` 修饰符移除、`filters` 移除
- 能讲响应式差异：Vue 2 defineProperty vs Vue 3 Proxy（新增属性、数组索引、删除属性都响应）
- 生命周期：`beforeDestroy`/`destroyed` → `beforeUnmount`/`unmounted`；`$listeners` 合并进 `$attrs`
- 工具链：vue-cli → Vite、composition API 可选渐进使用
- 迁移策略：用官方 `@vue/compat` 兼容构建在 Vue 3 上跑 Vue 2 代码，逐页迁移；或先抽公共逻辑/组件，按模块渐进替换
- 验证：自动化测试（单测/组件测试）在迁移前后保持一致，UI 走查 + 回归

**减分项：**
- 一上来就要「全量重写」（成本与风险没说清）
- 不知道兼容构建 @vue/compat 的存在
- 漏掉生命周期/全局 API 等破坏性变更导致运行报错
- 没有测试与回归策略
- 迁移后不检查性能（Vue 3 编译优化生效前提）

**解答：**
迁移要先分清「破坏性变更」与「可用兼容层缓解」。**核心破坏性变更清单**：全局 API 从 `Vue` 构造函数迁移到应用实例（`Vue.use()`→`app.use()`、`Vue.prototype.xxx`→`app.config.globalProperties.xxx`、`Vue.component`→`app.component`）；实例方法 `$on/$off/$once` 被移除（事件总线要改 Pinia/自定义事件或 mitt）；`filter` 语法移除（改用 computed/方法）；`slot`/`slot-scope` 改为 `v-slot`；`v-model` 在组件上默认绑定 `modelValue`（Vue 2 是 `value`）且 `sync` 修饰符合并为 `v-model`；`keyCode` 修饰符（`.13` 等）移除；生命周期 `beforeDestroy/destroyed` → `beforeUnmount/unmounted`、`$listeners` 并入 `$attrs`。**响应式差异**：Vue 2 的 defineProperty 不能侦测新增属性/数组下标赋值/删除属性；Vue 3 Proxy 全部覆盖——迁移后这些「历史补丁」（`Vue.set`、`$set`、数组 length 处理）可以删除。**迁移策略**：优先官方 `@vue/compat` 兼容构建——在 Vue 3 运行时提供 Vue 2 行为兼容（警告模式下逐项提示），让整个项目先在 Vue 3 上跑通、再逐页迁移到 Vue 3 写法，这是「渐进迁移」的官方路线；比「全量重写」成本低得多，风险也可控（每页迁移独立验证）。工具链建议同步到 Vite（构建快、与 Vue 3 编译优化配套）。**验证策略**：迁移前建立基线——关键路径的单元测试（`@vue/test-utils` v2）+ 组件测试 + E2E（Playwright）；每迁移一批页面跑一次全量回归，对比前后功能与性能（Vue 3 编译优化需要模板优先，若大量手写 render 要评估）；UI 走查重点看插槽、v-model、事件绑定这类语义变化的点。工程建议：先抽「无依赖的公共组件与 utils」小步迁移验证流程，再迁移业务页；迁移后注意检查 build 产物大小（Vue 3 更小）与运行时警告，清理兼容模式。

**延伸考点：** `@vue/compat` 迁移模式里「警告提示的破坏性变更」与「静默失效的变更」（如 $on）怎么区分优先级？Vue 3 移除 filters 后，`{{ text | capitalize }}` 这类模板最优雅的替代写法是什么？

---

### Q20. 大前端项目里 Vue 的代码组织、模块边界与「减少不必要渲染」的整体策略怎么设计？

**问题：** 一个中大型管理后台，几十个页面、几十个组件，已经出现：组件间隐式传值、页面越来越慢、改动一个公共组件引发多个页面回归。候选人要给出：Vue 项目的目录/模块组织规范、组件划分原则、以及「性能治理」的完整策略（怎么定位、怎么改、怎么防回归）。

**期望加分项：**
- 能讲目录组织：按业务域划分（features/modules + 共享层 common/shared），而不是按技术类型平铺（components/views/utils 大杂烩）
- 组件划分原则：原子组件（无业务状态）→ 业务组件（接 store/请求）→ 页面容器；容器组件与展示组件分离
- 状态分层：本地 state → composable → store 的渐进使用；禁止组件间「隔代传值」用 props 层层透传（用 provide/inject 或 store）
- 性能治理流程：先量化（Performance/Profiler 找长任务、慢组件）→ 再定位（是渲染次数、数据量还是响应式开销）→ 针对性改（拆分组件减少渲染范围、v-memo、shallowRef、key 稳定、异步组件）→ 回归
- 防回归：lint 规则（禁止在模板中写复杂表达式）、测试关键组件、Code Review 检查点
- 能谈构建与运行时的配合：路由级代码分割（动态 import）、按需引入 UI 库、CDN 与缓存

**减分项：**
- 目录按文件类型堆（views/components/utils 各一大坨）
- 组件划分没有边界（一个页面一个巨型组件）
- 性能治理没有量化，凭感觉「加缓存」「加 memo」
- 不防回归，改了不管
- 忽略构建层（按需加载/代码分割）对首屏的影响

**解答：**
中大型项目的组织核心是「边界」。**目录组织**：按业务域划分（`src/features/order`、`src/features/user`，各含自己的 components/composables/api/store），跨域复用放 `src/shared`（通用组件、工具、类型）——相比按技术类型平铺（所有 components 堆一起），业务域自治后「改一个域不影响其他域」、新人好定位。**组件划分**：分三层——原子组件（无业务依赖、纯 props 驱动，如 UiButton）、业务组件（接 store/接口，可复用跨页面）、页面容器（组合业务组件、负责路由数据）；配合「容器/展示分离」：容器管数据与逻辑，展示组件纯 props/emit，便于单测与复用。**状态分层原则**：能放组件本地（ref）不放 store，能放 composable（实例级复用）不放全局 store，只有跨组件/跨页面共享才进 Pinia；禁止「隔代 props 透传」，超过两层的共享用 `provide/inject` 或 store，props 只用于「就近传值」。**性能治理要流程化**：① 量化——用 Performance 面板/Profiler 找长任务与慢组件，确认瓶颈是「渲染次数多」「数据量大」还是「响应式开销大」，别凭感觉；② 针对性优化——渲染次数多：拆小组件缩小渲染边界、稳定 key、`v-memo` 缓存、避免把大对象传进 props（触发子组件更新）、异步组件（`defineAsyncComponent`）拆分非首屏；数据量大：虚拟滚动、分页；响应式开销：`shallowRef`/`markRaw`、避免 deep watch；③ 防回归——模板 lint 禁止复杂表达式、测试关键组件、Code Review 检查「props 变化是否引发不必要渲染」。**构建层配合**：路由级动态 import 做代码分割、UI 库按需引入（unplugin-vue-components）、静态资源上 CDN 加 contenthash 缓存。最后强调：组织规范与性能治理是「持续工程」，要沉淀为团队的 lint 规则、文档与 review check list，而不是某一次改造。

**延伸考点：** 「容器/展示组件分离」在 Vue 3 组合式 API 下的新形态（composable 承担了展示组件的数据来源）怎么理解？`defineAsyncComponent` 与路由懒加载在「首屏性能」上的分工是什么？
