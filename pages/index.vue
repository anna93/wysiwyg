<template>
  <section>
    <div class="editor" 
      @mouseup="getSelection($event,'mouse')" 
      @keydown="getSelection($event)"
      @drop.prevent="imageDragOver"
      @paste="onPaste"
      ref="editor"
      contenteditable="true">
    </div>
    <div class="toolbar">
      <button @click="convertToHead('h2')">h2</button>
      <button @click="convertToHead('h3')">h3</button>
      <button @click="convertToHead('i')"><i>i</i></button>
      <button @click="convertToHead('b')"><b>b</b></button>
      <button @click="convertToHead('u')"><u>ul</u></button>
      <button @click="addLink">href</button>
      <button @click="convertToHead">img</button>
      <button @click="convertToHead">code</button>
    </div>
    {{ articleObject }}
  </section>
</template>

<script>
import Logo from '~/components/Logo.vue'

export default {
  components: {
    Logo
  },
  data () {
    return {
      selection: null,
      articleObject: [],
      lastKey: ''
    }
  },
  methods: {
    convertToHead (type) {
      let currentNode = window.getSelection().focusNode
      let data = currentNode.data
      // check which one to remove
      // if parentNode is not the editor div, remove parent
      // else remove the the current Node
      if (currentNode.parentNode.className.indexOf('editor') >= 0) {
        currentNode.remove()
      } else {
        currentNode.parentNode.remove()
      }
      document.execCommand('insertHTML', false, `<${type}>${data}</${type}>`)
    },
    getSelection (e, isMouse) {
      if (isMouse) {
        this.selection = window.getSelection()
      } else if (e.shiftKey === true && /Arrow/.test(e.code)) {
        this.selection = window.getSelection()
      } else if (e.keyCode === 13) {
        // TODO: the behavior of enter key should depend on enclosing tags of cursor
        // for example in list tags it should create new li and inside h1 it should split h1 and a new
        // line should be created from the rest
        // console.log(e)
        // document.execCommand('insertHTML', false, '<br><br>')
        // e.preventDefault()
      } else {
        if (this.lastKey === 189 && e.keyCode === 32) { // - followed by space should convert into unordered list
          document.execCommand('insertUnOrderedList', false)
        } else if (this.lastKey === 49 && e.keyCode === 190) { // 1 followd by . should convert into ordered list
          document.execCommand('insertOrderedList', false)
        }
        this.lastKey = e.keyCode
      }
      // console.log(e)
      if (!isMouse) {
        this.updateArticleObject()
      }
    },
    updateArticleObject () {
      this.articleObject = []
      let editor = this.$refs.editor
      for (let key = 0; key < editor.children.length; ++key) {
        let nodeObj = {}
        nodeObj.type = editor.children[key].tagName
        nodeObj.content = editor.children[key].innerText
        this.articleObject.push(nodeObj)
      }
      this.articleObject = JSON.parse(JSON.stringify(this.articleObject))
    },
    imageDragOver (e) {
      let file = e.dataTransfer.files
      let fileReader = new FileReader()
      fileReader.readAsDataURL(file[0])

      fileReader.addEventListener('load', () => {
        let fileDataURI = fileReader.result

        let p = e.path
        let lastElement = this.findAppendTarget(p)
        let appendTarget
        let imgTag = document.createElement('img')
        imgTag.src = fileDataURI
        imgTag.style.maxWidth = '100%'
        if (!lastElement) {
          appendTarget = document.querySelector('.editor')
          appendTarget.appendChild(imgTag)
        } else {
          appendTarget = lastElement.parentElement
          appendTarget.insertBefore(imgTag, lastElement)
        }
      }, false)
    },
    addLink (e) {
      let text = window.getSelection().toString()
      if (text.length) {
        let link = window.prompt('Enter link')
        window.getSelection().deleteFromDocument()
        document.execCommand('insertHTML', false, ` <a href="${link}">${text}</a>`)
      }
    },
    onPaste (e) {
      let p = e.path
      let lastElement = this.findAppendTarget(p)

      let pasteText = e.clipboardData.getData('Text')
      if (/github.com/.test(pasteText)) {
        let regexp = /.*\/\/.*github\.com\/.*\/([a-z0-9]+)/g
        let id = regexp.exec(pasteText)['1']
        e.preventDefault()
        const loadGist = (gistId) => {
          var callbackName = 'gist_callback'
          document.getElementById(id).contentWindow[callbackName] = function (gistData) {
            delete document.getElementById(id).contentWindow[callbackName]
            let html = '<link rel="stylesheet" href="' + gistData.stylesheet + '"></link>'
            html += gistData.div
            iframeDoc.document.body.innerHTML += html
            setTimeout(() => {
              iframe.style.height = iframe.contentWindow.document.body.scrollHeight + 'px'
            }, 0)
            // script.parentNode.removeChild(script)
          }
          let iframeDoc = document.getElementById(id).contentWindow
          let script = iframeDoc.document.createElement('script')
          script.setAttribute('src', 'https://gist.github.com/' + gistId + '.json?callback=' + callbackName)
          iframeDoc.document.body.appendChild(script)
        }
        let iframe = document.createElement('iframe')
        iframe.style.width = '750px'
        iframe.setAttribute('id', id)
        let appendTarget
        if (!lastElement) {
          appendTarget = document.querySelector('.editor')
          appendTarget.appendChild(iframe)
        } else {
          appendTarget = lastElement.parentElement
          appendTarget.insertBefore(iframe, lastElement)
        }
        loadGist(id)
      }
    },
    findAppendTarget (path) {
      let lastElement = ''
      path.some(i => {
        if (i.className.indexOf('editor') >= 0) {
          return true
        }
        lastElement = i
      })
      return lastElement
    }
  }
}
</script>

<style lang="scss">
iframe {
  border: none;
}
.editor
{
  text-align: left;
  margin: 10px;
  padding: 5px;
  height: 90vh;
  border: 1px solid #CCC;

  &:focus {
    outline: none;
  }
  h1
  {
    font-size: 2.5rem;
    &:focus
    {
      outline: 1px solid red;
    }
  }
  h2
  {
    font-size: 2rem;
  }
  h3
  {
    font-size: 1.5rem;
  }
}

.editor {
  overflow-y: auto;
}

.toolbar 
{
  margin: 10px;
  button
  {
    margin-right: 5px;
  }
}
</style>
