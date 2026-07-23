<template>
  <div class="app">
    <div class="toolbar">
      <h1>出差看板</h1>
      <button @click="copyTable" class="btn">复制表格</button>
      <textarea v-model="inputText" rows="8" class="editor" spellcheck="false"></textarea>
    </div>

    <div v-if="!parsedData" class="empty">输入格式有误，请检查</div>

    <table v-else ref="tableEl" class="board">
      <thead>
        <tr>
          <th class="corner"></th>
          <th v-for="d in days" :key="d">{{ d }}</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(row, ri) in rows" :key="ri">
          <td class="index">{{ ri + 1 }}</td>
          <template v-for="(cell, ci) in row" :key="ci">
            <!-- 如果前面有空白天，补上空单元格 -->
            <td v-for="gap in (ci === 0 ? cell.colStart : cell.colStart - row[ci - 1].colStart - row[ci - 1].colSpan)" :key="'gap-' + ci + '-' + gap" class="cell empty"></td>
            <td :colspan="cell.colSpan" class="cell">{{ cell.name }}</td>
          </template>
          <!-- 行末剩余空白 -->
          <td v-for="gap in (7 - (row.length > 0 ? row[row.length - 1].colStart + row[row.length - 1].colSpan : 0))" :key="'endgap-' + gap" class="cell empty"></td>
        </tr>
        <tr v-if="rows.length === 0">
          <td colspan="8" class="empty">暂无出差记录</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

const DAYS = ['星期一', '星期二', '星期三', '星期四', '星期五', '星期六', '星期日']

interface PersonCell {
  name: string
  colStart: number
  colSpan: number
}

export default defineComponent({
  name: 'TripBoard',
  data() {
    return {
      days: DAYS,
      inputText: `{
  星期一: ['张三', '孙七'],
  星期二: ['张三', '孙七'],
  星期三: ['张三', '王五'],
  星期四: ['吴九', '王五'],
  星期五: ['李四', '王五'],
  星期六: ['周八'],
  星期日: ['周八', '赵六']
}`,
    }
  },
  computed: {
    parsedData(): Record<string, string[]> | null {
      try {
        const cleaned = this.inputText
          .replace(/([\u4e00-\u9fff\w]+)\s*:/g, '"$1":')
          .replace(/'/g, '"')
        return JSON.parse(cleaned)
      } catch {
        return null
      }
    },
    rows(): (PersonCell | null)[][] {
      const data = this.parsedData
      if (!data) return []
      return buildRows(data)
    },
  },
  methods: {
    isCellStart(row: PersonCell[], col: number) {
      const cell = row.find((c) => c.colStart === col)
      return !!cell
    },
    cellSpan(row: PersonCell[], col: number) {
      const cell = row.find((c) => c.colStart === col)
      return cell ? cell.colSpan : 1
    },
    cellName(row: PersonCell[], col: number) {
      const cell = row.find((c) => c.colStart <= col && col < c.colStart + c.colSpan)
      return cell ? cell.name : ''
    },
    copyTable() {
      const el = this.$refs.tableEl as HTMLElement
      if (!el) return
      const range = document.createRange()
      range.selectNode(el)
      window.getSelection()?.removeAllRanges()
      window.getSelection()?.addRange(range)
      document.execCommand('copy')
      window.getSelection()?.removeAllRanges()
    },
  },
})

function buildRows(data: Record<string, string[]>): PersonCell[][] {
  const dayLists = DAYS.map((d) => data[d] ?? [])
  // 最大行数 = 所有天中 values 的最大长度
  const maxRows = Math.max(...dayLists.map((l) => l.length), 0)
  const rows: PersonCell[][] = []

  for (let r = 0; r < maxRows; r++) {
    const row: PersonCell[] = []
    let col = 0
    while (col < 7) {
      const namesToday = dayLists[col]
      const name = r < namesToday.length ? namesToday[r] : null
      if (name === null) {
        col++
        continue
      }
      // 向后找连续天数：同一人且同一 index
      let span = 1
      while (col + span < 7) {
        const nextNames = dayLists[col + span]
        if (r < nextNames.length && nextNames[r] === name) {
          span++
        } else {
          break
        }
      }
      row.push({ name, colStart: col, colSpan: span })
      col += span
    }
    rows.push(row)
  }
  return rows
}
</script>
