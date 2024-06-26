<template>
  <div id="e" ref="editor" class="codemirror" style="position: relative;">
    <slot></slot>
  </div>
</template>

<script setup lang="ts">
import { EditorView, keymap, ViewUpdate, WidgetType, Decoration } from '@codemirror/view';
import { EditorState, StateEffect } from '@codemirror/state';
import { history, historyKeymap } from '@codemirror/history';
import { standardKeymap, insertTab } from '@codemirror/commands';
import { lineNumbers } from '@codemirror/gutter';
import { basicSetup } from '@codemirror/basic-setup';
// import { oneDarkTheme, oneDarkHighlightStyle } from '@codemirror/theme-one-dark';
import { autocompletion } from '@codemirror/autocomplete';
import { getCurrentInstance, ref, watch, onMounted } from 'vue';
import { generateLang } from '~/utils/highlight';
import { useStore } from '../store'
import {syntaxTree} from "@codemirror/language"

import { ViewPlugin } from "@codemirror/view"
import type { DecorationSet } from "@codemirror/view"

const editor = ref<HTMLDivElement>()
const store = useStore()

const emit = defineEmits<(e: 'change', v: ViewUpdate) => void>();

const reloadEditor = (highlight = false) => {
  store.editor.dispatch({
    effects: StateEffect.reconfigure.of(getExts(highlight))
  })
}

store._reloadEditor = reloadEditor

function getExts(highlight = false) {
  return [
    basicSetup,
    history(),
    EditorView.lineWrapping,
    // oneDarkTheme,
    keymap.of([
      ...standardKeymap,
      ...historyKeymap,
      // Tab Keymap
      {
        key: 'Tab',
        run: insertTab,
      },
    ]),
    checkboxPlugin,

    ...highlight ? generateLang(store.pcList, store.exportOptions) : [],
    EditorView.updateListener.of((v: ViewUpdate) => {
      if (v.docChanged) {
        emit('change', v);
        // temp1.view.state.doc.toString()
      }
    }),
    EditorView.theme({
      ".cm-gutters": {
        backgroundColor: "#0000",
      },
    })
  ]
}
class CheckboxWidget extends WidgetType {
  constructor(readonly url: string) { super() }

  eq(other: CheckboxWidget) { return other.url == this.url }

  toDOM() {
    let wrap = document.createElement("span")
    wrap.setAttribute("aria-hidden", "true")
    wrap.className = "cm-my-image" // edit-image
    let box = wrap.appendChild(document.createElement("img"))
    box.src = this.url
    // box.setAttribute('crossOrigin', 'anonymous')
    box.setAttribute('data-original', this.url)
    return wrap
  }

  ignoreEvent() { return false }
}


function checkboxes(view: EditorView) {
  let widgets: any = []
  for (let {from, to} of view.visibleRanges) {
    syntaxTree(view.state).iterate({
      from, to,
      enter: (type, from, to) => {
        if (type.name.startsWith("image-")) {
          const text = view.state.doc.sliceString(from, to)
          // ob11 - gocq
          let m = /url=([^\]]+)]/.exec(text) as RegExpExecArray
          if (m) {
            let deco = Decoration.widget({
              widget: new CheckboxWidget(m[1]),
              side: 0
            })
            widgets.push(deco.range(to))
          }

          m = /\[(?:image|图):([^\]]+)?([^\]]+)\]/.exec(text) as RegExpExecArray
          if (m) {
            let deco = Decoration.widget({
              widget: new CheckboxWidget(m[1]),
              side: 0
            })
            widgets.push(deco.range(to))
          }

          // ob11 - lagrange
          m = /file=(https?:\/\/[^\]]+)\]/.exec(text) as RegExpExecArray
          if (m) {
            let deco = Decoration.widget({
              widget: new CheckboxWidget(m[1]),
              side: 0
            })
            widgets.push(deco.range(to))
          }

          // ob11 - llob(new)
          m = /file=([A-Za-z0-9]{32,64})(\.[a-zA-Z]+?)\]/.exec(text) as RegExpExecArray
          if (m) {
            let deco = Decoration.widget({
              widget: new CheckboxWidget(`https://gchat.qpic.cn/gchatpic_new/0/0-0-${m[1]}/0?term=2,subType=1`),
              side: 0
            })
            widgets.push(deco.range(to))
          }

          // ob11 - llob(old)
          m = /file=file:\/\/[^\]]+([A-Za-z0-9]{32})(\.[a-zA-Z]+?)\]/.exec(text) as RegExpExecArray
          if (m) {
            let deco = Decoration.widget({
              widget: new CheckboxWidget(`https://gchat.qpic.cn/gchatpic_new/0/0-0-${m[1].toUpperCase()}/0?term=2,subType=1`),
              side: 0
            })
            widgets.push(deco.range(to))
          }

          m = /\[mirai:image:\{([A-Z0-9]+)-([A-Z0-9]+)-([A-Z0-9]+)-([A-Z0-9]+)-([A-Z0-9]+)}([^\]]+?)\]/.exec(text) as RegExpExecArray
          if (m) {
            const url = `https://gchat.qpic.cn/gchatpic_new/0/0-0-${m[1]}${m[2]}${m[3]}${m[4]}${m[5]}/0?term=2`
            let deco = Decoration.widget({
              widget: new CheckboxWidget(url),
              side: 0
            })
            widgets.push(deco.range(to))
          }
        }
      }
    })
  }
  return Decoration.set(widgets)
}

const checkboxPlugin = ViewPlugin.fromClass(class {
  decorations: DecorationSet

  constructor(view: EditorView) {
    this.decorations = checkboxes(view)

    // store.reloadEditor2 = () => {
    //   this.decorations = checkboxes(view)
    // }
  }

  update(update: ViewUpdate) {
    if (update.docChanged || update.viewportChanged) {
      this.decorations = checkboxes(update.view)
    }
  }
}, {
  decorations: v => v.decorations,

  eventHandlers: {
    mousedown: (e, view) => {
      // let target = e.target as HTMLElement
      // if (target.nodeName == "INPUT" &&
      //     target.parentElement!.classList.contains("cm-boolean-toggle"))
      //   return toggleBoolean(view, view.posAtDOM(target))
    }
  }
})

function toggleBoolean(view: EditorView, pos: number) {
  // let before = view.state.doc.sliceString(Math.max(0, pos - 5), pos)
  // let change
  // if (before == "false")
  //   change = {from: pos - 5, to: pos, insert: "true"}
  // else if (before.endsWith("true"))
  //   change = {from: pos - 4, to: pos, insert: "false"}
  // else
  //   return false
  // view.dispatch({changes: change})
  return true
}

const createEditor = (editorContainer: any, doc: any) => {
  if (store.editor) {
    store.editor.destroy();
  }

const startState = EditorState.create({
  //doc为编辑器默认内容
  doc: `海豹一号机(2589922907) 2022/03/21 19:05:05
新的故事开始了，祝旅途愉快！
记录已经开启。

木落(303451945) 2022/03/21 19:06:01
（好的测试开始了）

木落(303451945) 2022/03/21 19:06:21
[mirai:image:{829E3684-0489-D929-ABCE-674F2992FDC4}.jpg]

木落(303451945) 2022/03/21 19:06:47
从前有一座房子 [CQ:image,file=ee9f302089511a1a096a69d19985c35c.image,url=https://gchat.qpic.cn/gchatpic_new/303451945/2074687852-2830743842-EE9F302089511A1A096A69D19985C35C/0?term=2,subType=1]

木落(303451945) 2022/03/21 19:06:53
房前有两棵树

木落(303451945) 2022/03/21 19:07:06
一棵是枣树，另一颗也是枣树

木落(303451945) 2022/03/21 19:07:10
.ra 灵感

海豹一号机(2589922907) 2022/03/21 19:07:10
由于a 灵感，<木落>掷出了 D20=5

木落(303451945) 2022/03/21 19:07:20
啊没开扩展

木落(303451945) 2022/03/21 19:07:37
.ext coc7 on

海豹一号机(2589922907) 2022/03/21 19:07:37
打开扩展 coc7
检测到可能冲突的扩展，建议关闭: dnd5e

木落(303451945) 2022/03/21 19:07:55
.ra 灵感60

海豹一号机(2589922907) 2022/03/21 19:07:55
<木落>的灵感60检定结果为: d100=1/60, ([1d100=1]) 大成功！

木落(303451945) 2022/03/21 19:08:21
（？？？？？）

木落(303451945) 2022/03/21 19:08:23
.setcoc

海豹一号机(2589922907) 2022/03/21 19:08:23
当前房规: 0

木落(303451945) 2022/03/21 19:13:44
.r

海豹一号机(2589922907) 2022/03/21 19:13:44
<木落>掷出了 D20=7

木落(303451945) 2022/03/21 19:14:02
.nn 折影

海豹一号机(2589922907) 2022/03/21 19:14:02
<木落>(303451945)的昵称被设定为<折影>

折影(303451945) 2022/03/21 19:14:05
,r

折影(303451945) 2022/03/21 19:14:12
.r

海豹一号机(2589922907) 2022/03/21 19:14:12
<折影>掷出了 D20=15

折影(303451945) 2022/03/21 19:14:24
就这样吧

折影(303451945) 2022/03/21 19:14:29
草草结束

折影(303451945) 2022/03/21 19:14:35
.log end

海豹一号机(2589922907) 2022/03/21 19:14:35
故事落下了帷幕。
记录已经关闭。
`,
    extensions: getExts(),
  });

  store.editor = new EditorView({
    state: startState,
    parent: editorContainer,
  });

}

onMounted(() => {
  createEditor(editor.value, '')
})
</script>

<style>
/* 这个$props没有写错,不要改 */
.cm-editor {
  /* height: v-bind("$props.initHeight"); */
  height: 50rem;
  font-size: 18px;

  outline: 0 !important;
  /* height: 50rem; */
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.12), 0 0 6px rgba(0, 0, 0, 0.04);
}

.codemirror {
  height: 50rem;
}

.test {
  font-size: 2rem;
}

.cm-my-image > img {
  max-width: 8rem;
  max-height: 6rem;
}
</style>
