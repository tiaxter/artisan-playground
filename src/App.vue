<template>
  <div class='flex'>
    <v-navigation-drawer permanent mini-variant class="h-screen">
      <v-list>

        <v-list-item @click="playCode">
          <v-list-item-icon>
            <v-icon dense>mdi-play-outline</v-icon>
          </v-list-item-icon>
        </v-list-item>

        <v-list-item @click="setProjectFolder">
          <v-list-item-icon>
            <v-icon dense>mdi-folder-outline</v-icon>
          </v-list-item-icon>
        </v-list-item>

        <v-list-item>
          <v-list-item-icon>
            <v-icon dense>mdi-cog-outline</v-icon>
          </v-list-item-icon>
        </v-list-item>
      </v-list>

    </v-navigation-drawer>
    <codemirror
        class='w-1/2'
        v-model='editorCode'
        :options='editorOptions'
        @input='onCodeChange'
    />
    <codemirror
        class='w-1/2'
        :code='outputContent'
        :options='outputOptions'
    />
  </div>
</template>

<script>
import {codemirror} from 'vue-codemirror';
import 'codemirror/addon/scroll/simplescrollbars'
import 'codemirror/addon/scroll/simplescrollbars.css'
import 'codemirror/lib/codemirror.css';
import 'codemirror/theme/material.css'
import 'codemirror/mode/php/php';
import 'codemirror/mode/clike/clike';
import 'codemirror/addon/edit/closebrackets';
import 'codemirror/addon/comment/comment';
import 'codemirror/addon/fold/foldcode';
import 'codemirror/addon/fold/foldgutter';
import 'codemirror/addon/fold/foldgutter.css';
import 'codemirror/addon/fold/indent-fold';
import 'codemirror/addon/fold/comment-fold';
import 'codemirror/addon/fold/brace-fold';
import 'codemirror/addon/scroll/simplescrollbars';
import 'codemirror/addon/scroll/simplescrollbars.css';
import * as _ from 'lodash';
import {existsSync} from 'fs';
import {spawnSync} from 'child_process';
import {remote} from "electron";

export default {
  name: 'App',
  data() {
    const cmOptions = {
      mode: 'text/x-php',
      theme: 'material',
      lineNumbers: true,
      line: true,
      lineWrapping: true,
      tabSize: 4,
      autoCloseBrackets: true,
      indentWithTabs: true,
      foldGutter: true,
      gutters: ["CodeMirror-linenumbers", "CodeMirror-foldgutter"],
      scrollbarStyle: 'overlay',
    };
    return {
      editorCode: '',
      editorOptions: {
        ...cmOptions,
        autoFocus: true,
        extraKeys: {
          'Cmd-Enter': this.playCode,
          'Ctrl-Enter': this.playCode,
          'Ctrl-/': (cm) => cm.toggleComment(),
          'Cmd-/': (cm) => cm.toggleComment(),
        },
      },
      outputContent: '',
      outputOptions: {
        ...cmOptions,
        readOnly: true
      },
      cwd: null,
    }
  },
  methods: {
    setProjectFolder: async function () {
      const path = await remote.dialog.showOpenDialog({
        properties: ['openDirectory']
      });
      if (!path.canceled) {
        this.cwd = path?.filePaths[0];
      }
    },

    isAnPHPFolderProject: function (path) {
      return existsSync(`${path}/artisan`)
    },

    playCode: async function () {
      this.toggleMouseLoading(true);

      if (this.cwd === null && this.isAnPHPFolderProject(process.cwd())) {
        this.cwd = process.cwd();
      } else if (this.cwd !== null && this.isAnPHPFolderProject(this.cwd)) {
        const process = spawnSync('php', ['artisan', 'tinker'], {
          cwd: this.cwd,
          input: this.editorCode.replace(/(?:\/\*(?:[\s\S]*?)\*\/)|(?:[\s;]+\/\/(?:.*)$)/gm, ''),
          encoding: 'utf8'
        });
        this.outputContent = process.output.filter(e => e).join('\n');
      } else {
        await remote.dialog.showMessageBox({
          title: 'Artisan Playground',
          message: 'You must choose a valid PHP Project Directory first!',
          type: 'warning'
        });
      }

      this.toggleMouseLoading(false);
    },

    onCodeChange: _.debounce(function (editor) {this.playCode(editor);}, 350),

    toggleMouseLoading: function (isLoading) {
      document.body.style.cursor = isLoading ? 'progress' : 'default';
    },
  },
  components: {
    codemirror
  }
}
</script>

<style>
:root {
  --editor-code-font: 'MonoLisa', serif;
}

.CodeMirror {
  height: 100vh;
}

.CodeMirror * {
  font-family: var(--editor-code-font);
}
</style>