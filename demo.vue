<script lang="ts" name="FluentMathLiveEditor" setup>
import { throttle } from 'lodash-es'
import FluentEditor from '@opentiny/fluent-editor'
import type { IEditorModules } from '@opentiny/fluent-editor/types/config/types'
import '@opentiny/fluent-editor/style.css'
import 'mathlive'
import 'mathlive/static.css'
import 'mathlive/fonts.css'

interface IProps {
    content?: {
        type: 'html' | 'detail'
        value: string | Record<'ops' | string, Record<'insert' | string, any>[]>
    }
    modules?: IEditorModules
}

const props = withDefaults(defineProps<IProps>(), {
  modules: () => (
    {
      toolbar: {
        container: [
          [{ header: [false] }],
          ['bold', 'italic', 'underline'], // 加粗、斜体、下划线、删除线
          [{ color: [] }],
          ['formula'], // 公式按钮
          ['image'],
        ],
        handlers: {
        },
      },
      mathlive: true, // 启用 MathLive
      counter: {
        count: 1000, // 最大字符数
        enabled: true, // 启用计数器
      },
    }
  ),
},
)
const emits = defineEmits(['update:content'])
const model = defineModel({
  type: Object,
  default: () => ({}),
  required: false,
})
const editorInstance = ref<any>(null) // 保存编辑器实例

const uploadLoading = ref(false) // 上传文件时的 loading 状态

const { uploadFile } = useDirectOss()
const editorId = ref(`editor-${Math.random().toString(36).substring(2, 10)}`) // 生成唯一ID

const initEditor = () => {
  console.log('Initializing editor')

  editorInstance.value = new FluentEditor(`#${editorId.value}`, {
    theme: 'snow',
    modules: props.modules,
    uploadOption: {
      // @ts-ignore
      imageUpload: async ({ file, callback }) => {
        handleImageUpload(file, callback)
      },
    },
    mathliveOption: {
      virtualKeyboardMode: 'manual', // 启用虚拟键盘
      customVirtualKeyboardPolicy: () => ({
        attach: true,
        zIndex: 9999, // 设置虚拟键盘的 z-index
      }),
    },
  })
  if (editorInstance.value) {
    const throttledTextChangeHandler = throttle(() => {
      emits('update:content', getEditorContent())
      model.value = getEditorContent()
    }, 300) // 300ms 节流时间

    // 绑定节流回调到 'text-change' 事件
    editorInstance.value.on('text-change', throttledTextChangeHandler)

    // 设置初始默认内容
    setTimeout(() => {
      if (props.content && props.content.value) {
        if (props.content.type === 'html') {
          editorInstance.value.clipboard.dangerouslyPasteHTML(props.content.value)
        }
        else if (props.content.type === 'detail') {
          editorInstance.value.setContents(props.content.value)
        }
      }
    }, 1000)
  }
}

/**
 * 处理图片上传的函数
 *
 * @param file 要上传的文件
 * @param callback 上传完成后的回调函数
 */
async function handleImageUpload(file: File, callback: Function) {
  try {
    uploadLoading.value = true
    const { url: imageUrl } = await uploadFile(file)
    callback({
      code: 0,
      message: 'Upload success!',
      data: {
        imageUrl,
      },
    })
  }
  catch (error) {
    ElMessage.error(error.message || '上传失败，请稍后再试')
  }
  finally {
    uploadLoading.value = false
  }
}

/**
 * 获取编辑器内容
 *
 * @returns 包含编辑器内容的对象，包含三个属性：content（编辑器内容），plainText（纯文本内容），htmlContent（HTML内容）
 */
function getEditorContent() {
  if (!editorInstance.value)
    return {}

  const content = editorInstance.value.getContents()
  const plainText = editorInstance.value.getText()
  const htmlContent = editorInstance.value.root.innerHTML
  return {
    content,
    plainText,
    htmlContent,
  }
}

/**
 * 将 delta 对象转换为 HTML 字符串
 *
 * @param delta delta 对象，包含 ops 数组，ops 数组中的每个元素都是一个对象，包含 insert 和 attributes 属性
 * @returns 转换后的 HTML 字符串
 */
function renderDeltaToHTML(delta: Record<'ops' | string, Record<'insert' | string, any>[]>) {
  return delta.ops.reduce((html, op) => {
    const { insert, attributes } = op
    const styles = [
      attributes?.bold ? 'font-weight: bold;' : '',
      attributes?.color ? `color: ${attributes.color};` : '',
      attributes?.italic ? 'font-style: italic;' : '',
      attributes?.underline ? 'text-decoration: underline;' : '',
      attributes?.width ? `width: ${attributes.width};` : '',
      attributes?.height ? `height: ${attributes.height};` : '',
    ].join('')

    if (typeof insert === 'string') {
      return `${html}<span style="${styles}">${insert}</span>`
    }

    if (insert?.mathlive) {
      const { value } = insert.mathlive
      return `${html}<math-field class="ql-math-field view" mode="text" contenteditable="false" tabindex="0" read-only style="${styles}">${value}</math-field>`
    }

    if (insert?.image) {
      const { src, imageId } = insert.image
      return `${html}<img src="${src}" alt="${imageId}" style="${styles}" />`
    }

    return html
  }, '')
}

function getPreviewHTML(htmlContent: string) {
  return htmlContent.replace(/<math-field/g, '<math-field read-only')
}

defineExpose({
  getEditorContent,
  renderDeltaToHTML,
  getPreviewHTML,
})

onMounted(async () => {
  await nextTick()
  initEditor()
})

onUnmounted(() => {
  if (editorInstance.value) {
    editorInstance.value = null
  }
})
</script>

<template>
  <div v-loading="uploadLoading" class="w-full h-full flex flex-col overflow-hidden fluent-editor">
    <div :id="editorId" v-loading="uploadLoading" class="w-full flex-1" />
  </div>
</template>

<style lang="scss">
.fluent-editor {
  .ql-container {
    flex: 1;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
  }
}
</style>
