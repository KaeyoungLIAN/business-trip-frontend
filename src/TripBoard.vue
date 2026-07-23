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
          <td
            v-for="col in 7"
            :key="col"
            v-show="isCellStart(row, col - 1)"
            :colspan="cellSpan(row, col - 1)"
            class="cell"
          >
            {{ cellName(row, col - 1) }}
          </td>
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
    rows(): PersonCell[][] {
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
  const rows: PersonCell[][] = []
  // 按星期一的顺序排序，确保表格行顺序与星期一一致
  const monday = data[DAYS[0]] ?? []
  const allNames = [...new Set([...monday, ...dayLists.flat()])]

  for (const name of allNames) {
    const row: PersonCell[] = []
    let col = 0
    while (col < 7) {
      const namesToday = dayLists[col]
      const idx = namesToday.indexOf(name)
      if (idx === -1) {
        col++
        continue
      }
      let span = 1
      while (col + span < 7) {
        const nextNames = dayLists[col + span]
        if (nextNames.indexOf(name) === idx) {
          span++
        } else {
          break
        }
      }
      row.push({ name, colStart: col, colSpan: span })
      col += span
    }
    if (row.length > 0) rows.push(row)
  }
  return rows
}
</script>
