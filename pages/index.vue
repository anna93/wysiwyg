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
      <button @click="convertToCode">code</button>
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
    convertToCode () {
      // this function converts the selected text/or the current block
      // into a code block
      // debugger // eslint-disable-line
      let currentNode = window.getSelection().focusNode
      let data = currentNode.data
      let newNode = document.createElement('div')
      newNode.setAttribute('class', 'pre')
      newNode.innerHTML = data
      // replace current content by newly generated code block 
      // check which one to remove
      // if parentNode is not the editor div, remove parent
      // else remove the the current Node
      if (currentNode.parentNode.className.indexOf('editor') >= 0) {
        currentNode.parentNode.replaceChild(newNode, currentNode)
      } else {
        currentNode.parentNode.parentNode.replaceChild(newNode, currentNode.parentNode)
      }
      newNode.insertAdjacentHTML('afterend', '<div><br></div>')
    },
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
      // this gets called everytime a click or a selection happens in the editor.
      // this method helps in converting to ul and ol and breaking lines consistently on return key
      if (isMouse) {
        this.selection = window.getSelection()
      } else if (e.shiftKey === true && /Arrow/.test(e.code)) {
        this.selection = window.getSelection()
      } else if (e.keyCode === 13) {
        // TODO: the behavior of enter key should depend on enclosing tags of cursor
        // for example in list tags it should create new li and inside h1 it should split h1 and a new
        // line should be created from the rest
        let currentNode = window.getSelection().anchorNode.parentNode
        if (currentNode.className.indexOf('pre') >= 0) {
          e.preventDefault()
          let selection = window.getSelection()
          let currentNode = selection.getRangeAt(0)
          let node = document.createElement('br')
          currentNode.insertNode(node)
        }
      } else {
        if (this.lastKey === 189 && e.keyCode === 32) { // - followed by space should convert into unordered list
          document.execCommand('insertUnOrderedList', false)
        } else if (this.lastKey === 49 && e.keyCode === 190) { // 1 followd by . should convert into ordered list
          document.execCommand('insertOrderedList', false)
        }
        this.lastKey = e.keyCode
      }
      if (!isMouse) {
        this.updateArticleObject()
      }
    },
    updateArticleObject () {
      // for future use, this article object will be used to sync the text with the server side
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
      // allows drag and drop of images in the text area
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
      // for now this method asks for prompt to add link to a pre selected text
      // TODO: Use a nice html modal for link prompt
      let text = window.getSelection().toString()
      if (text.length) {
        let link = window.prompt('Enter link')
        window.getSelection().deleteFromDocument()
        document.execCommand('insertHTML', false, ` <a href="${link}">${text}</a>`)
      }
    },
    onPaste (e) {
      // Using embedly to convert links to preview cards
      let p = e.path
      let lastElement = this.findAppendTarget(p)

      let pasteText = e.clipboardData.getData('Text')
      // if link is github gist, do not use embedly, use githubs own script to render gist snippet
      if (/gist.github.com/.test(pasteText)) {
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
      } else if (/http/.test(pasteText)) {
        // use embedly to render other links as cards
        e.preventDefault()
        let link = document.createElement('a')
        link.setAttribute('href', pasteText)
        link.setAttribute('class', 'embedly-card')
        link.setAttribute('data-card-controls', 0)
        let script = document.createElement('script')
        script.setAttribute('async', true)
        script.setAttribute('src', '//cdn.embedly.com/widgets/platform.js')
        let appendTarget
        if (!lastElement) {
          appendTarget = document.querySelector('.editor')
          appendTarget.appendChild(link)
          appendTarget.appendChild(script)
        } else {
          appendTarget = lastElement.parentElement
          appendTarget.insertBefore(link, lastElement)
          appendTarget.insertBefore(script, lastElement)
        }
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
.pre {
  padding: 5px 5px 5px 20px;
  background: #cecece;
}
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
