<template>
  <div>
    <h1>텍스트 편집기</h1>
    <div id="editor-container" class="editor-container">
      <div
        v-for="(line, index) in lines"
        :key="line.id"
        :id="line.id"
        class="editor-line"
        :class="{ active: index === focusedIndex }"
        contenteditable="true"
        maxlength="100"
        @input="handleInput(index, $event)"
        @keydown="handleKeyDown(index, $event)"
        :ref="'line-' + index"
      >
        {{ line.text }}
      </div>
    </div>
  </div>
</template>
<script>
import { v4 as uuidv4 } from "uuid";

export default {
  data() {
    return {
      lines: [{ id: uuidv4(), text: "" }],
    };
  },
  created() {
    // 로컬 스토리지에서 블록 데이터 불러오기
    const savedLines = localStorage.getItem("editorLines");
    if (savedLines) {
      this.lines = JSON.parse(savedLines);
    }
  },
  beforeUnmount() {
    // 블록 데이터를 로컬 스토리지에 저장
    localStorage.setItem("editorLines", JSON.stringify(this.lines));
  },
  methods: {
    handleInput(index, event) {
      const text = event.target.textContent;
      if (text.length > 100) {
        event.target.textContent = text.slice(0, 100);
        this.addNewLine(index, text.slice(100));
      } else {
        this.lines[index].text = text;
        // 블록 데이터를 로컬 스토리지에 저장
        localStorage.setItem("editorLines", JSON.stringify(this.lines));
      }
    },
    handleKeyDown(index, event) {
      const text = event.target.textContent;
      if (text.length >= 100 && !this.isAllowedKey(event.keyCode)) {
        event.preventDefault();
        if (!event.ctrlKey && !event.shiftKey) {
          this.addNewLine(index, "");
        }
      }
      if (event.keyCode === 13) {
        event.preventDefault();
        if (!event.ctrlKey && !event.shiftKey) {
          const cursorPosition = this.getCursorPosition(event.target);
          const beforeText = text.slice(0, cursorPosition);
          const afterText = text.slice(cursorPosition);
          this.lines[index].text = beforeText;
          this.addNewLine(index, afterText, true);
        }
      }
      if (
        event.keyCode === 8 &&
        this.getCursorPosition(event.target) === 0 &&
        index > 0
      ) {
        event.preventDefault();
        const currentLineText = this.lines[index].text;
        const cursorPosition = this.lines[index - 1].text.length;
        this.lines[index - 1].text += currentLineText;
        this.removeLine(index);
        this.$nextTick(() => {
          this.focusLine(this.$refs[`line-${index - 1}`][0], cursorPosition);
        });
      }
      if (
        event.keyCode === 46 &&
        this.getCursorPosition(event.target) === text.length &&
        index < this.lines.length - 1
      ) {
        event.preventDefault();
        const cursorPosition = this.lines[index].text.length;
        this.lines[index].text += this.lines[index + 1].text;
        this.removeLine(index + 1);
        this.$nextTick(() => {
          this.focusLine(this.$refs[`line-${index}`][0], cursorPosition);
        });
      }
      if (
        event.keyCode === 37 &&
        this.getCursorPosition(event.target) === 0 &&
        index > 0
      ) {
        event.preventDefault();
        this.focusLine(this.$refs[`line-${index - 1}`][0], false);
      }
      if (
        event.keyCode === 39 &&
        this.getCursorPosition(event.target) === text.length &&
        index < this.lines.length - 1
      ) {
        event.preventDefault();
        this.focusLine(this.$refs[`line-${index + 1}`][0], true);
      }
      if (event.keyCode === 38 && index > 0) {
        event.preventDefault();
        this.$refs[`line-${index - 1}`][0].focus();
      }
      if (event.keyCode === 40 && index < this.lines.length - 1) {
        event.preventDefault();
        this.$refs[`line-${index + 1}`][0].focus();
      }
    },
    addNewLine(index, remainingText, focusStart = false) {
      const newLine = {
        id: uuidv4(),
        text: remainingText,
      };
      this.lines.splice(index + 1, 0, newLine);
      // 블록 데이터를 로컬 스토리지에 저장
      localStorage.setItem("editorLines", JSON.stringify(this.lines));
      this.$nextTick(() => {
        const newIndex = index + 1;
        const newLineElement = this.$refs[`line-${newIndex}`][0];
        this.focusLine(newLineElement, focusStart);
      });
    },
    isAllowedKey(keyCode) {
      const allowedKeys = [8, 37, 38, 39, 40, 46];
      return allowedKeys.includes(keyCode);
    },
    removeLine(index) {
      if (this.lines.length > 1) {
        this.lines.splice(index, 1);
        localStorage.setItem("editorLines", JSON.stringify(this.lines));
        this.$nextTick(() => {
          const focusIndex = index > 0 ? index - 1 : 0;
          const prevLine = this.$refs[`line-${focusIndex}`][0];
          this.focusLine(prevLine);
        });
      }
    },
    focusLine(line, cursorPosition = 0) {
      if (line) {
        line.focus();
        const range = document.createRange();
        const textNode = line.firstChild;
        if (textNode) {
          range.setStart(textNode, cursorPosition);
          range.collapse(true);
          const selection = window.getSelection();
          selection.removeAllRanges();
          selection.addRange(range);
        }
      }
    },
    getCursorPosition(element) {
      let caretOffset = 0;
      const doc = element.ownerDocument || element.document;
      const win = doc.defaultView || doc.parentWindow;
      const sel = win.getSelection();
      if (sel.rangeCount > 0) {
        const range = win.getSelection().getRangeAt(0);
        const preCaretRange = range.cloneRange();
        preCaretRange.selectNodeContents(element);
        preCaretRange.setEnd(range.endContainer, range.endOffset);
        caretOffset = preCaretRange.toString().length;
      }
      return caretOffset;
    },
  },
};
</script>
<style>
.editor-container {
  border: 1px solid #696969;
  padding: 10px;
  border-radius: 5px;
  background-color: #fff;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}
.editor-line {
  position: relative;
  height: 20px;
  padding: 4px;
  font-size: 16px;
  overflow: hidden;
  white-space: nowrap;
  border-radius: 5px;
  transition: background-color 0.3s ease;
}
.editor-line:hover {
  background-color: #ccc;
}
</style>