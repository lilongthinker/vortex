<template>
  <toolbar></toolbar>
  <textarea id='editor'></textarea>
</template>

<script>
  import CodeMirror from '../../static/codemirror/lib/codemirror.js'
  import '../../static/codemirror/mode/markdown/markdown.js'
  import '../../static/codemirror/addon/search/search.js'
  import '../../static/codemirror/addon/search/searchcursor.js'
  import '../../static/codemirror/addon/search/jump-to-line.js'
  import '../../static/codemirror/addon/search/matchesonscrollbar.js'
  import '../../static/codemirror/addon/dialog/dialog.js'
  import toolbar from './ToolBar'
  import { ipcRenderer } from 'electron'

  export default {
    watch: {
      'editor': {
        handler: function (val, oldVal) {
          const value = this.editor.getValue()
          const index = this.curIndex(value)
          this.$dispatch('update', value, index)
        },
        deep: true
      }
    },
    components: {
      toolbar
    },
    data () {
      return {
        editor: null,
        isScolling: false,
        winId: null,
        isUpdating: false
      }
    },
    ready () {
      const textarea = document.getElementById('editor')
      const editor = CodeMirror.fromTextArea(textarea, {
        lineWrapping: true,
        mode: 'markdown',
        theme: 'twilight',
        matchBrackets: true
      })
      window.editor = editor
      this.editor = editor
      const imageType = ['.jpg', '.jpeg', '.png', '.bmp', '.gif']
      editor.on('drop', function (editor, e) {
        const file = e.dataTransfer.files[0]
        const extension = file.name.substring(file.name.lastIndexOf('.'))
        if (imageType.indexOf(extension) >= 0) {
          editor.replaceSelection('![](' + file.path + ')')
          e.preventDefault()
        }
      })
      editor.on('focus', () => {
        window.isEditorOnFocus = true
      })
      editor.on('blur', () => {
        window.isEditorOnFocus = false
      })
      const self = this
      editor.on('change', () => {
        if (!this.isUpdating) {
          ipcRenderer.send('content-changed', self.winId)
        } else {
          this.isUpdating = false
        }
      })
      editor.on('scroll', function () {
        self.isScolling = true
        const scrollInfo = editor.getScrollInfo()
        self.$dispatch('transferTo', 'editorScrollChanged', scrollInfo)
      })
      ipcRenderer.on('set-window-id', (event, id) => {
        this.winId = id
      })
      ipcRenderer.on('open-file', (event, data) => {
        this.editor.setValue(data)
        ipcRenderer.send('content-saved', self.winId)
      })
      ipcRenderer.on('trigger-save-file', (event, filename) => {
        const content = this.editor.getValue()
        ipcRenderer.send('save-file', this.winId, filename, content)
        ipcRenderer.send('content-saved', self.winId)
      })
      this.$dispatch('transferTo', 'focusEditor')
    },
    methods: {
      curIndex (value) {
        const lines = value.split('\n')
        const curLine = this.editor.getCursor().line
        let index = 0
        let preLine = null
        let isCodeBlock = false
        for (let i in lines) {
          if (curLine < i) {
            break
          } if (lines[i].startsWith('```')) {
            isCodeBlock = !isCodeBlock
          } else if (lines[i].startsWith('---') && !preLine && !isCodeBlock) {
            index++
          }
          preLine = lines[i]
        }
        return index
      }
    },
    events: {
      previewScrollChanged: function (scrollInfo) {
        if (this.isScolling) {
          this.isScolling = false
          return
        }
        const height = this.editor.getScrollInfo().height
        const scrollTop = (scrollInfo.top / scrollInfo.height) * height
        this.editor.scrollTo(null, scrollTop)
      },
      focusEditor: function () {
        this.editor.focus()
      },
      updateEditorValue: function () {
        this.isUpdating = true
        this.editor.setValue(this.editor.getValue())
      }
    }
  }
</script>

<style>
.editor {
  margin-bottom: 16px;
}
.CodeMirror {
  font-family: Helvetica, Tahoma, Arial, "Hiragino Sans GB", "Microsoft YaHei", "WenQuanYi Micro Hei", sans-serif;
  height: calc(100% - 24px);
  line-height: 1.6em;
  color: #757575;
  font-size: 15px;
  padding: 10px 20px 0 20px;
  background-color: white !important;
}
.cm-link {
  text-decoration: none;
}
</style>
