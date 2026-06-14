<template>
  <div class="gallery-page">
    <div class="gallery-header">
      <h2>🏛️ 匿名广场</h2>
      <p>看看别人挂出来的短信，猜猜上下文是什么</p>
    </div>

    <div v-if="allPosts.length === 0" class="empty-state">
      <div class="icon">📭</div>
      <h3>广场还空空的</h3>
      <p>去情书墙挂出第一条短信，让大家猜猜看吧！</p>
      <router-link to="/wall" class="btn btn-primary">去情书墙</router-link>
    </div>

    <template v-else>
      <div class="tag-filter-bar">
        <div class="filter-row">
          <span class="filter-label">🏷️ 按标签筛选：</span>
          <div class="tag-buttons">
            <button
              class="tag-filter-btn all-btn"
              :class="{ active: !store.selectedTagFilter }"
              @click="store.clearTagFilter()"
            >
              全部
            </button>
            <button
              v-for="tag in allAvailableTags"
              :key="tag.type"
              class="tag-filter-btn"
              :class="['tag-' + tag.type, { active: store.selectedTagFilter === tag.type }]"
              @click="store.setTagFilter(tag.type)"
            >
              {{ tag.text }}
            </button>
          </div>
        </div>
        <div v-if="store.selectedTagFilter" class="filter-active-row">
          <span class="filter-active-info">
            当前筛选：
            <span class="tag" :class="'tag-' + store.selectedTagFilter">
              {{ getTagTextByType(store.selectedTagFilter) }}
            </span>
            <span class="filter-count">（共 {{ filteredPosts.length }} 条）</span>
          </span>
          <button class="clear-filter-btn" @click="store.clearTagFilter()">
            ✕ 清除筛选
          </button>
        </div>
      </div>

      <div v-if="filteredPosts.length === 0" class="empty-state filtered-empty">
        <div class="icon">🔍</div>
        <h3>没有找到匹配的短信</h3>
        <p>换个标签试试，或者看看其他内容吧～</p>
        <button class="btn btn-primary" @click="store.clearTagFilter()">
          查看全部
        </button>
      </div>

      <div v-else class="posts-grid">
        <div 
          v-for="post in filteredPosts" 
          :key="post.id"
          class="post-card card"
        >
          <div class="post-header">
            <span class="post-date">{{ formatDate(post.date) }}</span>
            <span 
              v-if="post.isSent" 
              class="post-type sent"
            >📤 发出的短信</span>
            <span 
              v-else 
              class="post-type received"
            >📥 收到的短信</span>
          </div>

          <div class="post-message">
            "{{ post.message }}"
          </div>

          <div class="post-tags" v-if="post.tags.length > 0">
            <span 
              v-for="tag in post.tags" 
              :key="tag.type"
              class="tag clickable-tag"
              :class="['tag-' + tag.type, { active: store.selectedTagFilter === tag.type }]"
              @click="store.setTagFilter(tag.type)"
            >
              {{ tag.text }}
            </span>
          </div>

          <div class="post-meta" v-if="!post.anonymous && post.originalConversation">
            <span>来自: {{ post.originalConversation }}</span>
          </div>

          <div class="post-stats">
            <span>💭 {{ post.guesses.length }} 个猜测</span>
            <span v-if="post.context">🔓 已揭晓</span>
            <span v-else>🔒 待揭晓</span>
          </div>

          <div class="post-actions">
            <router-link 
              :to="'/guess/' + post.id" 
              class="btn btn-primary"
            >
              我来猜猜 →
            </router-link>
            
            <button 
              v-if="!post.context"
              class="btn btn-secondary"
              @click="revealContext(post)"
            >
              揭晓答案
            </button>
          </div>

          <div v-if="post.context" class="context-reveal">
            <div class="context-section" v-if="post.context.prev">
              <label>👆 上一条：</label>
              <p>{{ post.context.prev }}</p>
            </div>
            <div class="context-section" v-if="post.context.next">
              <label>👇 下一条：</label>
              <p>{{ post.context.next }}</p>
            </div>
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { store } from '@/store'

const demoPosts = [
  {
    id: 'demo1',
    message: '想你了，真的好想好想',
    date: Date.now() - 86400000,
    isSent: false,
    context: null,
    tags: [{ type: 'miss', text: '思念' }],
    guesses: [
      { id: 'g1', text: '应该是异地恋，好久没见了吧？', likes: 12 },
      { id: 'g2', text: '会不会是刚吵架和好？', likes: 8 }
    ],
    anonymous: true,
    originalConversation: null
  },
  {
    id: 'demo2',
    message: '晚安，梦里见 🌙',
    date: Date.now() - 86400000 * 2,
    isSent: true,
    context: {
      prev: '今天好累啊，先睡了',
      next: '晚安呀，明天见 ❤️'
    },
    tags: [{ type: 'night', text: '晚安' }],
    guesses: [
      { id: 'g3', text: '好甜！应该是热恋期吧', likes: 25 }
    ],
    anonymous: true,
    originalConversation: null
  },
  {
    id: 'demo3',
    message: '对不起，我错了还不行吗 😢',
    date: Date.now() - 86400000 * 3,
    isSent: true,
    context: null,
    tags: [{ type: 'sorry', text: '道歉' }],
    guesses: [
      { id: 'g4', text: '是不是又忘了什么纪念日？', likes: 32 },
      { id: 'g5', text: '我猜是打游戏忘了回消息', likes: 28 },
      { id: 'g6', text: '感觉是惹女朋友生气了哈哈哈', likes: 19 }
    ],
    anonymous: true,
    originalConversation: null
  },
  {
    id: 'demo4',
    message: '宝宝今天超乖的，奖励一个抱抱～ 🤗',
    date: Date.now() - 86400000 * 4,
    isSent: true,
    context: null,
    tags: [{ type: 'cute', text: '撒娇' }],
    guesses: [
      { id: 'g7', text: '撒狗粮来了！', likes: 15 }
    ],
    anonymous: true,
    originalConversation: null
  },
  {
    id: 'demo5',
    message: '你能不能不要每次都这样！真的很烦！',
    date: Date.now() - 86400000 * 5,
    isSent: false,
    context: null,
    tags: [{ type: 'quarrel', text: '争吵' }],
    guesses: [
      { id: 'g8', text: '吵得挺凶的，快去哄！', likes: 22 }
    ],
    anonymous: true,
    originalConversation: null
  },
  {
    id: 'demo6',
    message: '我爱你，一生一世 💕',
    date: Date.now() - 86400000 * 6,
    isSent: true,
    context: null,
    tags: [{ type: 'love', text: '爱意' }],
    guesses: [
      { id: 'g9', text: '求婚现场？', likes: 40 }
    ],
    anonymous: true,
    originalConversation: null
  },
  {
    id: 'demo7',
    message: '秒回你的消息，是不是很感动～',
    date: Date.now() - 86400000 * 7,
    isSent: true,
    context: null,
    tags: [{ type: 'freq', text: '秒回' }],
    guesses: [
      { id: 'g10', text: '这才是真爱啊', likes: 18 }
    ],
    anonymous: true,
    originalConversation: null
  }
]

const allPosts = computed(() => {
  return [
    ...store.anonymousPosts,
    ...demoPosts
  ].sort((a, b) => b.date - a.date)
})

const allAvailableTags = computed(() => {
  const tagMap = new Map()
  allPosts.value.forEach(post => {
    post.tags.forEach(tag => {
      if (!tagMap.has(tag.type)) {
        tagMap.set(tag.type, tag)
      }
    })
  })
  return Array.from(tagMap.values())
})

const filteredPosts = computed(() => {
  if (!store.selectedTagFilter) {
    return allPosts.value
  }
  return allPosts.value.filter(post =>
    post.tags.some(tag => tag.type === store.selectedTagFilter)
  )
})

function getTagTextByType(type) {
  const tag = allAvailableTags.value.find(t => t.type === type)
  return tag ? tag.text : type
}

function revealContext(post) {
  if (!post.context) {
    post.context = {
      prev: '这是上一条消息的内容',
      next: '这是下一条消息的内容'
    }
  }
}

function formatDate(timestamp) {
  const d = new Date(timestamp)
  const now = Date.now()
  const diff = now - timestamp
  
  if (diff < 3600000) {
    return Math.floor(diff / 60000) + ' 分钟前'
  }
  if (diff < 86400000) {
    return Math.floor(diff / 3600000) + ' 小时前'
  }
  if (diff < 86400000 * 7) {
    return Math.floor(diff / 86400000) + ' 天前'
  }
  
  return `${d.getFullYear()}/${d.getMonth() + 1}/${d.getDate()}`
}
</script>

<style scoped>
.gallery-page {
  max-width: 1200px;
  margin: 0 auto;
}

.gallery-header {
  text-align: center;
  margin-bottom: 2rem;
}

.gallery-header h2 {
  font-size: 2rem;
  color: var(--love-red);
  margin-bottom: 0.5rem;
}

.gallery-header p {
  color: var(--text-light);
}

.tag-filter-bar {
  margin-bottom: 2rem;
  padding: 1.25rem;
  background: white;
  border-radius: 12px;
  border: 1px solid var(--border);
  box-shadow: var(--shadow);
}

.filter-row {
  display: flex;
  align-items: flex-start;
  gap: 1rem;
  flex-wrap: wrap;
}

.filter-label {
  color: var(--text-dark);
  font-weight: 500;
  flex-shrink: 0;
  padding-top: 0.35rem;
}

.tag-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  flex: 1;
}

.tag-filter-btn {
  padding: 0.4rem 1rem;
  border: 2px solid var(--border);
  background: white;
  border-radius: 20px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.3s;
  color: var(--text-light);
}

.tag-filter-btn:hover {
  transform: translateY(-2px);
}

.tag-filter-btn.all-btn.active {
  background: linear-gradient(135deg, var(--love-red), var(--love-pink));
  color: white;
  border-color: transparent;
}

.tag-filter-btn.tag-love.active {
  background: #ffe5e5;
  color: var(--love-red);
  border-color: var(--love-red);
}

.tag-filter-btn.tag-freq.active {
  background: #fff3e0;
  color: var(--secondary);
  border-color: var(--secondary);
}

.tag-filter-btn.tag-quarrel.active {
  background: #ffebee;
  color: #c62828;
  border-color: #c62828;
}

.tag-filter-btn.tag-cute.active {
  background: #fce4ec;
  color: #ad1457;
  border-color: #ad1457;
}

.tag-filter-btn.tag-sorry.active {
  background: #fff8e1;
  color: #f57f17;
  border-color: #f57f17;
}

.tag-filter-btn.tag-night.active {
  background: #e8eaf6;
  color: #283593;
  border-color: #283593;
}

.tag-filter-btn.tag-miss.active {
  background: #f3e5f5;
  color: #6a1b9a;
  border-color: #6a1b9a;
}

.filter-active-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px dashed var(--border);
}

.filter-active-info {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: var(--text-dark);
}

.filter-count {
  color: var(--text-light);
  font-size: 0.9rem;
}

.clear-filter-btn {
  padding: 0.4rem 0.9rem;
  border: 2px solid var(--love-pink);
  background: white;
  color: var(--love-red);
  border-radius: 20px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.3s;
  font-weight: 500;
}

.clear-filter-btn:hover {
  background: var(--love-pink);
  color: white;
}

.filtered-empty {
  background: white;
  border-radius: 16px;
  border: 1px solid var(--border);
  box-shadow: var(--shadow);
  margin: 2rem auto;
  max-width: 500px;
}

.filtered-empty .icon {
  font-size: 4rem;
}

.posts-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
  gap: 1.5rem;
}

.post-card {
  display: flex;
  flex-direction: column;
  transition: all 0.3s;
}

.post-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 30px rgba(231, 76, 60, 0.15);
}

.post-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.post-date {
  color: var(--text-light);
  font-size: 0.85rem;
}

.post-type {
  font-size: 0.8rem;
  padding: 0.25rem 0.75rem;
  border-radius: 12px;
}

.post-type.sent {
  background: #ffe5e5;
  color: var(--love-red);
}

.post-type.received {
  background: #f3e5f5;
  color: var(--accent);
}

.post-message {
  font-size: 1.2rem;
  line-height: 1.6;
  color: var(--text-dark);
  padding: 1rem 0;
  border-top: 1px solid var(--border);
  border-bottom: 1px solid var(--border);
  margin-bottom: 1rem;
  font-style: italic;
}

.post-tags {
  margin-bottom: 1rem;
}

.clickable-tag {
  cursor: pointer;
  transition: all 0.3s;
  border: 2px solid transparent;
}

.clickable-tag:hover {
  transform: scale(1.05);
  opacity: 0.9;
}

.clickable-tag.active {
  border-color: currentColor;
  box-shadow: 0 0 0 2px rgba(0,0,0,0.05);
}

.post-meta {
  color: var(--text-light);
  font-size: 0.9rem;
  margin-bottom: 1rem;
}

.post-stats {
  display: flex;
  gap: 1.5rem;
  color: var(--text-light);
  font-size: 0.9rem;
  margin-bottom: 1rem;
}

.post-actions {
  display: flex;
  gap: 0.75rem;
  margin-top: auto;
}

.context-reveal {
  margin-top: 1rem;
  padding: 1rem;
  background: var(--bg-light);
  border-radius: 8px;
  border-left: 3px solid var(--love-pink);
}

.context-section {
  margin-bottom: 0.75rem;
}

.context-section:last-child {
  margin-bottom: 0;
}

.context-section label {
  display: block;
  font-size: 0.8rem;
  color: var(--text-light);
  margin-bottom: 0.25rem;
}

.context-section p {
  font-size: 0.95rem;
  color: var(--text-dark);
  padding: 0.5rem;
  background: white;
  border-radius: 6px;
}

@media (max-width: 768px) {
  .filter-row {
    flex-direction: column;
  }

  .filter-active-row {
    flex-direction: column;
    gap: 0.75rem;
    align-items: flex-start;
  }
}
</style>
