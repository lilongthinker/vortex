<template>
  <div id='text-preview' class='text' @scroll='onscroll'>
    <div class="markdown-body" v-html="text"></div>
  </div>
</template>

<script>
  import marked from '../../node_modules/marked/lib/marked.js'
  import katex from '../../node_modules/katex/katex.js'
  import util from '../lib/util.js'

  const renderer = new marked.Renderer()
  util.customizeLink(renderer)
  util.customizeKatex(katex, renderer)
  util.customizeCode(renderer)
  marked.setOptions({renderer: renderer})
  marked.setOptions({
    highlight: (code, lang) => {
      return window.hljs.highlightAuto(code, [lang]).value
    }
  })

  export default {
    data () {
      return {
        text: '',
        isScolling: false
      }
    },
    methods: {
      onscroll () {
        const dom = document.getElementById('text-preview')
        const scrollTop = dom.scrollTop
        const height = dom.scrollHeight
        const scrollInfo = {top: scrollTop, height: height}
        this.$dispatch('transferTo', 'previewScrollChanged', scrollInfo)
        this.isScolling = true
      }
    },
    events: {
      updateMarkdown: function (value) {
        this.text = marked(value)
      },
      editorScrollChanged: function (scrollInfo) {
        if (this.isScolling) {
          this.isScolling = false
          return
        }
        const dom = document.getElementById('text-preview')
        const height = dom.scrollHeight
        dom.scrollTop = (scrollInfo.top / scrollInfo.height) * height
      }
    }
  }
</script>

<style lang="scss">
.text {
  height: 100%;
  margin-top: 20px;

  .markdown-body {
    font-size: 15px;
    padding: 24px 32px;
    overflow-y: hidden;

    ::-webkit-scrollbar {
      display: block;
    }

    ::-webkit-scrollbar-track {
      background-color: white;
    }

    ::-webkit-scrollbar {
      width: 3px;
      height: 6px;
      background-color: white;
    }

    ::-webkit-scrollbar-thumb {
      background-color: #616161;
      border-radius: 20px;
    }

    hr {
      height: 1px;
      color: #616161;
      border-bottom-color: #616161;
    }

    .katex-mathml {
      display: none;
    }
  }
}
</style>
