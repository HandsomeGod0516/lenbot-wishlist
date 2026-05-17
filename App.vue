<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

// =========================================================================
// types
// =========================================================================
type Status = 'pending' | 'picked' | 'ready' | 'done' | 'rejected'

interface WishItem {
  id: number
  user_id: number
  target_project_slug: string | null
  proposed_new_slug: string | null
  title: string
  description: string | null
  status: Status
  created_at: string
  updated_at: string
  username: string
  display_name: string | null
}

interface Project { slug: string; name: string; description: string }

interface Execution {
  id: number
  generated_prompt: string
  status: 'draft' | 'approved' | 'done'
  approved_at: string | null
  created_at: string
}

// =========================================================================
// state
// =========================================================================
const items = ref<WishItem[]>([])
const isAdmin = ref(false)
const projects = ref<Project[]>([])
const errorMsg = ref('')
const okMsg = ref('')
const loading = ref(false)

// 新增表單
const draft = ref({
  targetMode: 'existing' as 'existing' | 'new' | 'unset',
  targetProjectSlug: '',
  proposedNewSlug: '',
  title: '',
  description: '',
})

// admin 勾選
const picked = ref<Set<number>>(new Set())
const extraNote = ref('')

// admin 看 prompt
const currentExec = ref<Execution | null>(null)
const promptCopied = ref(false)

// =========================================================================
// load
// =========================================================================
async function loadProjects() {
  try {
    const r = await fetch('/api/projects', { credentials: 'include' })
    if (!r.ok) return
    const d = await r.json()
    projects.value = d.projects || []
  } catch { /* silent */ }
}

async function loadItems() {
  loading.value = true
  errorMsg.value = ''
  try {
    const r = await fetch('/api/wishlist', { credentials: 'include' })
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    const d = await r.json()
    items.value = d.items || []
    isAdmin.value = !!d.isAdmin
  } catch (e: any) {
    errorMsg.value = '載入失敗：' + (e?.message || e)
  } finally {
    loading.value = false
  }
}

// =========================================================================
// create / delete / status (user)
// =========================================================================
async function createWish() {
  errorMsg.value = ''
  if (!draft.value.title.trim()) {
    errorMsg.value = '標題必填'
    return
  }
  const body: Record<string, any> = {
    title: draft.value.title.trim(),
    description: draft.value.description.trim(),
  }
  if (draft.value.targetMode === 'existing' && draft.value.targetProjectSlug) {
    body.targetProjectSlug = draft.value.targetProjectSlug
  } else if (draft.value.targetMode === 'new' && draft.value.proposedNewSlug) {
    body.proposedNewSlug = draft.value.proposedNewSlug.trim()
  }
  try {
    const r = await fetch('/api/wishlist', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify(body),
    })
    if (!r.ok) {
      const d = await r.json().catch(() => null)
      throw new Error(d?.statusMessage || `HTTP ${r.status}`)
    }
    // reset
    draft.value = { targetMode: 'existing', targetProjectSlug: '', proposedNewSlug: '', title: '', description: '' }
    okMsg.value = '✓ 已送出'
    setTimeout(() => { okMsg.value = '' }, 2000)
    await loadItems()
  } catch (e: any) {
    errorMsg.value = '建立失敗：' + (e?.message || e)
  }
}

async function deleteItem(id: number) {
  if (!confirm('刪除這條願望？')) return
  try {
    await fetch(`/api/wishlist/${id}`, { method: 'DELETE', credentials: 'include' })
    picked.value.delete(id)
    await loadItems()
  } catch (e: any) {
    errorMsg.value = '刪除失敗：' + (e?.message || e)
  }
}

async function patchStatus(id: number, status: Status) {
  try {
    const r = await fetch(`/api/wishlist/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({ status }),
    })
    if (!r.ok) {
      const d = await r.json().catch(() => null)
      throw new Error(d?.statusMessage || `HTTP ${r.status}`)
    }
    await loadItems()
  } catch (e: any) {
    errorMsg.value = '更新失敗：' + (e?.message || e)
  }
}

// admin dropdown 觸發 — 雙保險再 confirm 一次「降級」操作 (done → 其他)
async function onStatusChange(id: number, next: Status) {
  const cur = items.value.find(i => i.id === id)
  if (!cur || cur.status === next) return
  if (cur.status === 'done' && next !== 'done') {
    if (!confirm(`要把已完成的「${cur.title}」改回「${STATUS_LABEL[next]}」嗎？`)) {
      await loadItems()  // 強制 dropdown 顯示回原本值
      return
    }
  }
  await patchStatus(id, next)
}

// =========================================================================
// admin: execute (產 prompt)
// =========================================================================
const pickedCount = computed(() => picked.value.size)

function togglePick(id: number) {
  if (picked.value.has(id)) picked.value.delete(id)
  else picked.value.add(id)
  // force reactivity (Set 不會自動觸發)
  picked.value = new Set(picked.value)
}

const executing = ref(false)
async function execute() {
  if (!pickedCount.value) {
    errorMsg.value = '請先勾選至少一條願望'
    return
  }
  if (!confirm(`要把 ${pickedCount.value} 條願望丟給 gemma4:31b 產 prompt 嗎？\n(會把這些 items 狀態改為 picked，並建立一筆 execution 記錄)`)) return
  executing.value = true
  errorMsg.value = ''
  try {
    const r = await fetch('/api/admin/wishlist/execute', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({
        itemIds: Array.from(picked.value),
        extraNote: extraNote.value.trim() || undefined,
      }),
    })
    if (!r.ok) {
      const d = await r.json().catch(() => null)
      throw new Error(d?.statusMessage || `HTTP ${r.status}`)
    }
    const d = await r.json()
    currentExec.value = {
      id: d.executionId,
      generated_prompt: d.prompt,
      status: 'draft',
      approved_at: null,
      created_at: new Date().toISOString(),
    }
    picked.value = new Set()
    extraNote.value = ''
    await loadItems()
  } catch (e: any) {
    errorMsg.value = '產 prompt 失敗：' + (e?.message || e)
  } finally {
    executing.value = false
  }
}

async function approveExec() {
  if (!currentExec.value) return
  if (!confirm('確認 prompt 沒問題 → 這批 items 會轉為「待 Claude 接手」(ready) 狀態。')) return
  try {
    await fetch(`/api/admin/wishlist/executions/${currentExec.value.id}/approve`, {
      method: 'POST', credentials: 'include',
    })
    currentExec.value.status = 'approved'
    await loadItems()
    okMsg.value = '✓ 已確認，items 轉為 ready (等 Claude 接手)。複製下方 prompt 給 Claude Code 跑；Claude 做完後請手動把 status 改成 done。'
    setTimeout(() => { okMsg.value = '' }, 8000)
  } catch (e: any) {
    errorMsg.value = '確認失敗：' + (e?.message || e)
  }
}

async function copyPrompt() {
  if (!currentExec.value) return
  try {
    await navigator.clipboard.writeText(currentExec.value.generated_prompt)
    promptCopied.value = true
    setTimeout(() => { promptCopied.value = false }, 1500)
  } catch {
    errorMsg.value = '無法複製，請手動選取'
  }
}

function closeExec() {
  currentExec.value = null
  promptCopied.value = false
}

// =========================================================================
// util
// =========================================================================
const STATUS_LABEL: Record<Status, string> = {
  pending: '待處理',
  picked:  '已挑選',
  ready:   '待 Claude 接手',
  done:    '已完成',
  rejected:'已拒絕',
}
const STATUS_ORDER: Status[] = ['pending', 'picked', 'ready', 'done', 'rejected']

function targetLabel(it: WishItem): string {
  if (it.proposed_new_slug) return `新建：${it.proposed_new_slug}`
  if (it.target_project_slug) {
    const p = projects.value.find(p => p.slug === it.target_project_slug)
    return p ? p.name : it.target_project_slug
  }
  return '未指定'
}

function relTime(iso: string): string {
  const sec = (Date.now() - new Date(iso).getTime()) / 1000
  if (sec < 60) return '剛剛'
  if (sec < 3600) return `${Math.floor(sec / 60)} 分鐘前`
  if (sec < 86400) return `${Math.floor(sec / 3600)} 小時前`
  const d = new Date(iso)
  return `${d.getMonth() + 1}/${d.getDate()}`
}

onMounted(() => {
  loadProjects()
  loadItems()
})
</script>

<template>
  <div class="wish">
    <header class="head">
      <div>
        <p class="kicker">WISHLIST</p>
        <h2>願望清單</h2>
        <p class="sub" v-if="isAdmin">你是管理者 — 可以勾選願望、產 prompt 給 Claude 執行</p>
        <p class="sub" v-else>提你想要的功能 / 想改的子專案，管理者會挑選實作</p>
      </div>
      <button class="btn-ghost" @click="loadItems" :disabled="loading">↻ 重新載入</button>
    </header>

    <p v-if="errorMsg" class="alert">{{ errorMsg }}</p>
    <p v-if="okMsg" class="ok">{{ okMsg }}</p>

    <!-- 新增表單 -->
    <section class="card form">
      <h3>新增一條願望</h3>
      <div class="row">
        <label class="lbl">
          <span>目標</span>
          <div class="target-modes">
            <label><input type="radio" v-model="draft.targetMode" value="existing"> 既有子專案</label>
            <label><input type="radio" v-model="draft.targetMode" value="new"> 新建子專案</label>
            <label><input type="radio" v-model="draft.targetMode" value="unset"> 不確定 / 全域</label>
          </div>
        </label>
      </div>
      <div class="row" v-if="draft.targetMode === 'existing'">
        <label class="lbl">
          <span>子專案</span>
          <select v-model="draft.targetProjectSlug">
            <option value="">— 請選 —</option>
            <option v-for="p in projects" :key="p.slug" :value="p.slug">{{ p.name }} ({{ p.slug }})</option>
          </select>
        </label>
      </div>
      <div class="row" v-if="draft.targetMode === 'new'">
        <label class="lbl">
          <span>擬定 slug</span>
          <input v-model="draft.proposedNewSlug" placeholder="例：notes (a-z 開頭，a-z0-9-)" />
        </label>
      </div>
      <div class="row">
        <label class="lbl">
          <span>標題</span>
          <input v-model="draft.title" placeholder="一句話描述你要的功能 / 改動" maxlength="255" />
        </label>
      </div>
      <div class="row">
        <label class="lbl">
          <span>細節</span>
          <textarea v-model="draft.description" rows="3" placeholder="再說明一下：為什麼要做、UI 怎麼運作、邊角情境…"></textarea>
        </label>
      </div>
      <div class="row right">
        <button class="btn-primary" :disabled="!draft.title.trim()" @click="createWish">送出願望</button>
      </div>
    </section>

    <!-- 列表 -->
    <section v-if="!loading && !items.length" class="empty">
      還沒有任何願望。先提一條吧 🌱
    </section>

    <section v-else-if="items.length" class="card list-card">
      <h3>{{ isAdmin ? '所有人的願望' : '我的願望' }} <span class="count">({{ items.length }})</span></h3>

      <!-- admin batch bar -->
      <div v-if="isAdmin" class="batch-bar">
        <div class="batch-info">
          已勾選 <strong>{{ pickedCount }}</strong> 條
          <span v-if="pickedCount">— 將丟給 gemma4:31b 產 prompt</span>
        </div>
        <input v-model="extraNote" class="batch-note" placeholder="(可選) 給 Claude 的補充指示…" />
        <button class="btn-primary" :disabled="!pickedCount || executing" @click="execute">
          {{ executing ? '產 prompt 中…' : `▶ 產 prompt (${pickedCount})` }}
        </button>
      </div>

      <ul class="items">
        <li v-for="it in items" :key="it.id" :class="['item', `s-${it.status}`]">
          <div class="item-check" v-if="isAdmin">
            <input
              type="checkbox"
              :checked="picked.has(it.id)"
              :disabled="it.status === 'done' || it.status === 'rejected'"
              @change="togglePick(it.id)"
            />
          </div>
          <div class="item-body">
            <div class="item-head">
              <span class="title">{{ it.title }}</span>
              <span :class="['badge', `b-${it.status}`]">{{ STATUS_LABEL[it.status] }}</span>
            </div>
            <div class="item-meta">
              <span class="tgt">📦 {{ targetLabel(it) }}</span>
              <span class="who">by @{{ it.username }}</span>
              <span class="when">{{ relTime(it.created_at) }}</span>
            </div>
            <p v-if="it.description" class="desc">{{ it.description }}</p>
            <div class="item-actions">
              <!-- admin: 任意改 status (狀態修正) -->
              <label v-if="isAdmin" class="status-edit">
                <span>狀態</span>
                <select :value="it.status" @change="onStatusChange(it.id, ($event.target as HTMLSelectElement).value as Status)">
                  <option v-for="s in STATUS_ORDER" :key="s" :value="s">{{ STATUS_LABEL[s] }}</option>
                </select>
              </label>
              <!-- admin 快捷：把 pending / picked 直接「啟動」轉 ready -->
              <button
                v-if="isAdmin && (it.status === 'pending' || it.status === 'picked')"
                class="mini accent"
                title="跳過 gemma 產 prompt，直接標記為等 Claude 接手"
                @click="patchStatus(it.id, 'ready')"
              >✓ 啟動自動編寫</button>
              <!-- admin 快捷：標記 Claude 已做完 -->
              <button
                v-if="isAdmin && it.status === 'ready'"
                class="mini"
                @click="patchStatus(it.id, 'done')"
              >Claude 已完成 → done</button>
              <button class="mini danger" @click="deleteItem(it.id)">刪除</button>
            </div>
          </div>
        </li>
      </ul>
    </section>

    <!-- prompt modal -->
    <div v-if="currentExec" class="modal-mask" @click.self="closeExec">
      <div class="modal">
        <header class="modal-head">
          <div>
            <p class="kicker">EXECUTION · #{{ currentExec.id }}</p>
            <h3>給 Claude 的 prompt</h3>
            <p class="sub">由 gemma4:31b 產出，請先過目；確認 OK 後複製給 Claude Code 執行</p>
          </div>
          <button class="close" @click="closeExec">✕</button>
        </header>
        <pre class="prompt"><code>{{ currentExec.generated_prompt }}</code></pre>
        <div v-if="currentExec.status === 'approved'" class="approved-hint">
          ✓ 已確認 — 把上面 prompt 複製貼進 Claude Code (一個視窗開在 LenFirstWeb/ 底下)，Claude 會直接改檔 + git commit + push。
        </div>
        <footer class="modal-foot">
          <button class="btn-ghost" @click="copyPrompt">{{ promptCopied ? '✓ 已複製' : '複製到剪貼簿' }}</button>
          <span class="spacer" />
          <button v-if="currentExec.status === 'draft'" class="btn-primary" @click="approveExec">確認執行</button>
          <button class="btn-ghost" @click="closeExec">關閉</button>
        </footer>
      </div>
    </div>
  </div>
</template>

<style scoped>
.wish {
  max-width: 920px;
  margin: 0 auto;
  padding: 28px 32px 80px;
  color: var(--tx-1, #eee);
  font-family: inherit;
}
.head {
  display: flex; align-items: flex-end; justify-content: space-between;
  gap: 16px;
  margin-bottom: 24px;
}
.kicker {
  font-size: 11px; letter-spacing: 2px;
  color: var(--tx-3, #888);
  margin: 0 0 6px;
}
h2 { margin: 0; font-weight: 600; letter-spacing: .5px; }
.sub { color: var(--tx-3, #888); font-size: 13px; margin: 6px 0 0; }
.btn-ghost {
  background: transparent;
  border: 1px solid var(--line-2, rgba(255,255,255,.15));
  color: var(--tx-2, #ccc);
  padding: 8px 14px;
  border-radius: 6px;
  font: inherit;
  cursor: pointer;
  transition: border-color .15s, color .15s, background .15s;
}
.btn-ghost:hover:not(:disabled) { color: var(--tx-1); border-color: var(--line-3, rgba(255,255,255,.3)); }
.btn-ghost:disabled { opacity: .4; cursor: not-allowed; }
.btn-primary {
  background: var(--tx-1, #eee); color: #000;
  border: none;
  padding: 9px 18px; border-radius: 6px;
  font-weight: 600; cursor: pointer;
  transition: opacity .15s;
}
.btn-primary:hover:not(:disabled) { opacity: .85; }
.btn-primary:disabled { background: var(--bg-elev, #2a2a2a); color: var(--tx-4, #555); cursor: not-allowed; }

.alert { color: #ff8888; margin: 8px 0; }
.ok    { color: #82e7a8; margin: 8px 0; }

.card {
  background: var(--bg-elev, #1c1c1c);
  border: 1px solid var(--line-1, rgba(255,255,255,.08));
  border-radius: 10px;
  padding: 20px 22px;
  margin-bottom: 20px;
}
.card h3 { margin: 0 0 14px; font-size: 15px; font-weight: 600; letter-spacing: .3px; }
.count { color: var(--tx-3, #888); font-weight: 400; margin-left: 6px; }

.form .row { margin-bottom: 12px; }
.form .row.right { text-align: right; }
.lbl { display: grid; grid-template-columns: 80px 1fr; gap: 10px; align-items: center; }
.lbl > span { color: var(--tx-3, #888); font-size: 12px; letter-spacing: 1px; }
.lbl input, .lbl textarea, .lbl select {
  background: var(--bg-input, #0e0e0e);
  border: 1px solid var(--line-2, rgba(255,255,255,.12));
  color: var(--tx-1);
  padding: 9px 12px;
  border-radius: 6px;
  font: inherit;
  outline: none;
  width: 100%;
  font-size: 14px;
}
.lbl input:focus, .lbl textarea:focus, .lbl select:focus { border-color: var(--tx-2, #aaa); }
.target-modes { display: flex; gap: 14px; color: var(--tx-2); font-size: 14px; }
.target-modes input { margin-right: 4px; }

.empty {
  text-align: center;
  padding: 60px;
  color: var(--tx-3, #777);
}

.batch-bar {
  display: grid;
  grid-template-columns: auto 1fr auto;
  gap: 12px;
  align-items: center;
  padding: 12px 14px;
  background: var(--bg-input, #0e0e0e);
  border: 1px dashed var(--line-2, rgba(255,255,255,.15));
  border-radius: 6px;
  margin-bottom: 14px;
}
.batch-info { font-size: 13px; color: var(--tx-2); }
.batch-info strong { color: var(--tx-1); }
.batch-note {
  background: transparent;
  border: 1px solid var(--line-2);
  color: var(--tx-1);
  padding: 7px 10px; border-radius: 5px;
  font: inherit; font-size: 13px;
  outline: none;
}
.batch-note:focus { border-color: var(--tx-2); }

.items { list-style: none; padding: 0; margin: 0; display: flex; flex-direction: column; gap: 10px; }
.item {
  display: grid;
  grid-template-columns: auto 1fr;
  gap: 12px;
  padding: 14px 16px;
  background: var(--bg-input, #0e0e0e);
  border: 1px solid var(--line-1, rgba(255,255,255,.06));
  border-radius: 8px;
}
.item.s-picked { border-color: rgba(130,231,168,.4); }
.item.s-done   { opacity: .6; }
.item.s-rejected { opacity: .45; }

.item-check { padding-top: 2px; }
.item-check input { transform: scale(1.2); cursor: pointer; }
.item-check input:disabled { cursor: not-allowed; }

.item-head { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.title { font-weight: 600; font-size: 15px; }
.badge {
  font-size: 11px; letter-spacing: .5px;
  padding: 2px 8px;
  border-radius: 4px;
  border: 1px solid currentColor;
  font-weight: 500;
}
.b-pending  { color: #f0d97a; }
.b-picked   { color: #82e7a8; }
.b-ready    { color: #7ab8ff; }
.b-done     { color: var(--tx-3, #888); }
.b-rejected { color: #ff8888; }

.item-meta {
  display: flex; gap: 14px; flex-wrap: wrap;
  font-size: 12px;
  color: var(--tx-3, #888);
  margin-top: 4px;
}
.tgt { color: var(--tx-2, #ccc); }
.desc {
  margin: 8px 0 0;
  font-size: 14px;
  color: var(--tx-2, #ccc);
  white-space: pre-wrap;
  line-height: 1.5;
}
.item-actions { display: flex; gap: 8px; margin-top: 10px; }
.mini {
  background: transparent;
  border: 1px solid var(--line-2);
  color: var(--tx-2);
  padding: 4px 10px;
  border-radius: 4px;
  font-size: 12px;
  font: inherit; font-size: 12px;
  cursor: pointer;
}
.mini:hover { color: var(--tx-1); border-color: var(--line-3); }
.mini.danger { color: #ff8888; border-color: rgba(255,136,136,.4); }
.mini.danger:hover { background: rgba(255,136,136,.1); }
.mini.accent { color: #7ab8ff; border-color: rgba(122,184,255,.5); }
.mini.accent:hover { background: rgba(122,184,255,.12); border-color: rgba(122,184,255,.8); }

.status-edit { display: inline-flex; align-items: center; gap: 6px; font-size: 11px; color: var(--tx-3, #888); }
.status-edit > span { letter-spacing: .5px; }
.status-edit select {
  background: var(--bg-input, #0e0e0e);
  border: 1px solid var(--line-2, rgba(255,255,255,.12));
  color: var(--tx-1, #eee);
  font: inherit; font-size: 12px;
  padding: 3px 8px; border-radius: 4px;
  outline: none; cursor: pointer;
}
.status-edit select:focus { border-color: var(--tx-2, #aaa); }

/* ---------------- modal ---------------- */
.modal-mask {
  position: fixed; inset: 0;
  background: rgba(0,0,0,.7);
  display: grid; place-items: center;
  z-index: 100;
  padding: 20px;
}
.modal {
  background: var(--bg-elev, #1c1c1c);
  border: 1px solid var(--line-2, rgba(255,255,255,.15));
  border-radius: 12px;
  width: min(900px, 100%);
  max-height: 88vh;
  display: flex; flex-direction: column;
  overflow: hidden;
}
.modal-head {
  display: flex; align-items: flex-start; justify-content: space-between;
  padding: 18px 22px;
  border-bottom: 1px solid var(--line-1);
}
.modal-head h3 { margin: 4px 0 0; }
.close {
  background: transparent; border: none; color: var(--tx-3); font-size: 18px;
  cursor: pointer; padding: 4px 10px;
}
.close:hover { color: var(--tx-1); }
.prompt {
  flex: 1;
  margin: 0;
  padding: 18px 22px;
  overflow: auto;
  background: var(--bg-input, #0e0e0e);
  color: var(--tx-1);
  font: 13px/1.6 ui-monospace, "JetBrains Mono", Menlo, monospace;
  white-space: pre-wrap;
}
.approved-hint {
  padding: 12px 22px;
  background: rgba(130,231,168,.07);
  border-top: 1px solid rgba(130,231,168,.2);
  color: #82e7a8;
  font-size: 13px;
}
.modal-foot {
  display: flex; align-items: center; gap: 10px;
  padding: 14px 22px;
  border-top: 1px solid var(--line-1);
}
.spacer { flex: 1; }
</style>
