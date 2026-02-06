<template>
  <div class="blog-container">
    <v-card class="pa-4 blog-main-card">
      
      <v-container v-if="!selectedPost" class="blog-list-container">
        <div v-if="selectedTag" class="filter-info mb-2">
          <v-chip color="var(--leleo-vcard-color)" closable @click:close="$emit('clearTag')">
            标签: {{ selectedTag }}
          </v-chip>
        </div>
        <v-card
          v-for="post in filteredPosts"
          :key="post.id"
          class="ma-2 cursor-pointer blog-card"
          hover
          @click="selectPost(post)"
        >
          <v-card-title style="color: var(--leleo-vcard-color); font-weight: 600;">{{ post.title }}</v-card-title>
          <v-card-subtitle style="color: rgba(255, 255, 255, 0.7);">
            {{ post.date }} · {{ post.author }}
          </v-card-subtitle>
          <v-card-text style="color: rgba(255, 255, 255, 0.85);">{{ post.excerpt }}</v-card-text>
          <v-card-actions v-if="post.tags && post.tags.length > 0">
            <v-chip
              v-for="tag in post.tags"
              :key="tag"
              size="small"
              class="ma-1"
              color="var(--leleo-vcard-color)"
              variant="outlined"
            >
              {{ tag }}
            </v-chip>
          </v-card-actions>
        </v-card>
      </v-container>
      
      <v-container v-else>
        <v-btn
          variant="tonal"
          color="var(--leleo-vcard-color)"
          class="mb-4"
          @click="selectedPost = null"
        >
          <v-icon left>mdi-arrow-left</v-icon>
          返回博客列表
        </v-btn>
        
        <div class="blog-content-container">
          <div class="blog-content" v-html="selectedPost.content"></div>
        </div>
      </v-container>
    </v-card>
  </div>
</template>

<script>
export default {
  props: ['configdata', 'selectedTag'],
  data() {
    return {
      posts: [],
      selectedPost: null,
      marked: null
    };
  },
  computed: {
    blurValue() {
      return this.configdata?.blur || 5;
    },
    filteredPosts() {
      if (!this.selectedTag) return this.posts;
      return this.posts.filter(post => post.tags && post.tags.includes(this.selectedTag));
    },
    allTags() {
      const tags = new Set();
      this.posts.forEach(post => {
        if (post.tags) {
          post.tags.forEach(tag => tags.add(tag));
        }
      });
      return Array.from(tags);
    }
  },
  mounted() {
    // 确保 marked 库加载完成后再加载文章
    if (window.marked) {
      this.loadMarked();
      this.loadPosts();
    } else {
      // 如果 marked 库还没有加载，等待一下再尝试
      setTimeout(() => {
        this.loadMarked();
        this.loadPosts();
      }, 100);
    }
    // 添加代码复制功能
    this.$nextTick(() => {
      this.addCopyButtons();
    });
  },
  updated() {
    // 在 DOM 更新后添加复制按钮
    this.$nextTick(() => {
      this.addCopyButtons();
    });
  },
  methods: {
    loadMarked() {
      // 直接使用 window.marked
      this.marked = window.marked;
    },
    async loadPosts() {
      try {
        // 尝试加载索引文件
        let fileList = [];
        try {
          const indexResponse = await fetch('/post/index.json');
          if (indexResponse.ok) {
            fileList = await indexResponse.json();
          }
        } catch (e) {
          // 索引文件不存在，使用默认列表
        }
        
        // 如果没有索引文件，使用默认的文件列表
        if (fileList.length === 0) {
          fileList = [
            '2025-04-07-C++-A+B-problem.md',
            '2025-04-12-Deeply-understand-the-binary-search-algorithm-in-C++.md',
            '2025-04-14-Luogu-P2392-Problem-Solution-Integration.markdown',
            '2025-05-05-Luogu-P1032-Problem.markdown',
            '2025-05-05-Luogu-P1160-Problem.markdown',
            '2025-10-7-What\'s_DilWorth.md',
            '2025-10-8-Interval-Dynamic-Programming.md',
            'example1.md',
            'example2.md'
          ];
        }
        
        const posts = [];
        
        for (const filename of fileList) {
          const response = await fetch(`/post/${filename}`);
          if (response.ok) {
            const content = await response.text();
            
            // 解析 Front Matter 和内容
            const { frontMatter, body } = this.parseFrontMatter(content);
            
            // 提取标题（优先使用 Front Matter 中的 title）
            let title = frontMatter.title;
            if (!title) {
              title = this.extractTitle(body);
            }
            
            // 提取日期（优先使用 Front Matter 中的 date）
            let date = frontMatter.date;
            if (!date) {
              // 尝试从文件名中提取日期
              const dateMatch = filename.match(/^(\d{4}-\d{2}-\d{2})/);
              date = dateMatch ? dateMatch[1] : new Date().toLocaleDateString();
            }
            
            // 提取摘要
            const excerpt = this.extractExcerpt(body);
            
            // 跳过无标题的文章
            if (title && title !== '无标题') {
              // 使用 marked 解析 markdown
              let parsedContent = body;
              if (window.marked && window.marked.parse) {
                parsedContent = window.marked.parse(body);
              }
              
              posts.push({
                id: filename.replace(/\.(md|markdown)$/i, ''),
                title: title,
                content: parsedContent,
                excerpt: excerpt,
                date: date,
                author: frontMatter.author || 'ZZJ',
                tags: frontMatter.tags || []
              });
            }
          }
        }
        
        // 按日期排序（最新的在前）
        posts.sort((a, b) => new Date(b.date) - new Date(a.date));
        
        // 如果没有找到任何文章，显示一个提示
        if (posts.length === 0) {
          posts.push({
            id: 'no-posts',
            title: '暂无博客文章',
            content: '<p>请在 public/post 目录下添加 markdown 文件来创建博客文章。</p>',
            excerpt: '暂无博客文章',
            date: new Date().toLocaleDateString()
          });
        }
        
        this.posts = posts;
        
        // 更新标签列表（在 posts 赋值后）
        this.$nextTick(() => {
          this.$emit('updateTags', this.allTags);
        });
      } catch (error) {
        console.error('加载博客文章失败:', error);
        
        // 显示错误信息
        this.posts = [{
          id: 'error',
          title: '加载失败',
          content: `<p>加载博客文章时出错: ${error.message}</p>`,
          excerpt: '加载失败',
          date: new Date().toLocaleDateString()
        }];
      }
    },
    parseFrontMatter(content) {
      const frontMatter = {};
      let body = content;
      
      // 检查是否有 Front Matter
      const match = content.match(/^---\s*\n([\s\S]*?)\n---\s*\n([\s\S]*)$/);
      if (match) {
        const fmText = match[1];
        body = match[2];
        
        // 解析 YAML 格式的 Front Matter
        const lines = fmText.split('\n');
        let currentKey = '';
        let currentValue = '';
        let inArray = false;
        
        for (const line of lines) {
          const trimmed = line.trim();
          
          // 检查是否是数组项
          if (trimmed.startsWith('- ')) {
            if (!inArray) {
              inArray = true;
              frontMatter[currentKey] = [];
            }
            frontMatter[currentKey].push(trimmed.substring(2).trim());
          } else {
            inArray = false;
            const colonIndex = trimmed.indexOf(':');
            if (colonIndex > 0) {
              currentKey = trimmed.substring(0, colonIndex).trim();
              currentValue = trimmed.substring(colonIndex + 1).trim();
              // 去除引号
              if ((currentValue.startsWith('"') && currentValue.endsWith('"')) ||
                  (currentValue.startsWith("'") && currentValue.endsWith("'"))) {
                currentValue = currentValue.slice(1, -1);
              }
              frontMatter[currentKey] = currentValue;
            }
          }
        }
      }
      
      return { frontMatter, body };
    },
    extractTitle(content) {
      const match = content.match(/^#\s+(.+)$/m);
      return match ? match[1] : '无标题';
    },
    extractExcerpt(content) {
      // 移除 markdown 标记，提取纯文本
      let text = content
        .replace(/^---[\s\S]*?---/m, '') // 移除 Front Matter
        .replace(/^#{1,6}\s+.+$/gm, '') // 移除所有标题 (h1-h6)
        .replace(/```[\s\S]*?```/g, '') // 移除代码块
        .replace(/`([^`]+)`/g, '$1') // 移除行内代码标记
        .replace(/\*\*([^*]+)\*\*/g, '$1') // 移除加粗
        .replace(/\*([^*]+)\*/g, '$1') // 移除斜体
        .replace(/__([^_]+)__/g, '$1') // 移除下划线加粗
        .replace(/_([^_]+)_/g, '$1') // 移除下划线斜体
        .replace(/\[([^\]]+)\]\([^)]+\)/g, '$1') // 移除链接，保留文本
        .replace(/!\[([^\]]*)\]\([^)]+\)/g, '') // 移除图片
        .replace(/>\s?/g, '') // 移除引用标记
        .replace(/\n/g, ' ') // 换行转空格
        .replace(/\s+/g, ' ') // 多个空格合并为一个
        .trim();
      
      return text.length > 120 ? text.substring(0, 120) + '...' : text;
    },
    selectPost(post) {
      this.selectedPost = post;
      // 选择文章后延迟添加复制按钮，确保 DOM 已更新
      setTimeout(() => {
        this.addCopyButtons();
      }, 100);
    },
    addCopyButtons() {
      // 获取博客内容容器中的所有 pre 元素（代码块）
      const container = this.$el.querySelector('.blog-content');
      if (!container) return;
      
      const codeBlocks = container.querySelectorAll('pre');
      codeBlocks.forEach(pre => {
        // 检查是否已经添加了复制按钮
        if (pre.querySelector('.copy-btn')) return;
        
        // 确保 pre 元素有相对定位
        pre.style.position = 'relative';
        
        // 创建复制按钮
        const copyBtn = document.createElement('button');
        copyBtn.className = 'copy-btn';
        copyBtn.innerHTML = '复制';
        copyBtn.title = '复制代码';
        
        // 添加点击事件
        copyBtn.addEventListener('click', (e) => {
          e.stopPropagation();
          e.preventDefault();
          const code = pre.querySelector('code') || pre;
          const text = code.innerText || code.textContent;
          
          // 复制到剪贴板
          navigator.clipboard.writeText(text).then(() => {
            copyBtn.innerHTML = '已复制';
            copyBtn.classList.add('copied');
            setTimeout(() => {
              copyBtn.innerHTML = '复制';
              copyBtn.classList.remove('copied');
            }, 2000);
          }).catch(err => {
            console.error('复制失败:', err);
            // 降级方案
            const textarea = document.createElement('textarea');
            textarea.value = text;
            document.body.appendChild(textarea);
            textarea.select();
            document.execCommand('copy');
            document.body.removeChild(textarea);
            copyBtn.innerHTML = '已复制';
            copyBtn.classList.add('copied');
            setTimeout(() => {
              copyBtn.innerHTML = '复制';
              copyBtn.classList.remove('copied');
            }, 2000);
          });
        });
        
        // 将按钮添加到 pre 元素中
        pre.appendChild(copyBtn);
      });
    }
  }
};
</script>

<style>
.blog-container {
  width: 100%;
  min-height: 100%;
}

/* 博客主卡片样式 - 使用 CSS 变量实现毛玻璃效果 */
.blog-main-card {
  backdrop-filter: blur(var(--leleo-blur)) !important;
  background-color: rgba(255, 255, 255, 0.15) !important;
  border-radius: 12px !important;
  border: 1px solid rgba(255, 255, 255, 0.3) !important;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1) !important;
}

/* 博客主卡片滚动条 */
.blog-main-card::-webkit-scrollbar {
  width: 8px;
}

.blog-main-card::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
}

.blog-main-card::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 4px;
}

.blog-main-card::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}

/* 博客列表容器 */
.blog-list-container {
  max-height: calc(100vh - 80px);
  overflow-y: auto;
  padding-right: 8px;
}

/* 博客列表滚动条 */
.blog-list-container::-webkit-scrollbar {
  width: 6px;
}

.blog-list-container::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.05);
  border-radius: 3px;
}

.blog-list-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 3px;
}

.blog-list-container::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.4);
}

/* 博客列表滚动条 */
.blog-list-container::-webkit-scrollbar {
  width: 6px;
}

.blog-list-container::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.05);
  border-radius: 3px;
}

.blog-list-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 3px;
}

.blog-list-container::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.4);
}

.cursor-pointer {
  cursor: pointer;
}

/* 博客卡片样式 - 使用 CSS 变量实现毛玻璃效果 */
.blog-card {
  backdrop-filter: blur(var(--leleo-blur)) !important;
  background-color: rgba(255, 255, 255, 0.2) !important;
  border-radius: 8px !important;
  border: 1px solid rgba(255, 255, 255, 0.2) !important;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1) !important;
  transition: all 0.3s ease;
}

.blog-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.15) !important;
}

/* 博客内容容器样式 - 使用 CSS 变量实现毛玻璃效果 */
.blog-content-container {
  backdrop-filter: blur(var(--leleo-blur));
  background-color: rgba(255, 255, 255, 0.25);
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  padding: 2rem;
  max-height: calc(100vh - 120px);
  overflow-y: auto;
}

/* 博客内容滚动条 */
.blog-content-container::-webkit-scrollbar {
  width: 8px;
}

.blog-content-container::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
}

.blog-content-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 4px;
}

.blog-content-container::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}

/* 自定义滚动条样式 */
.blog-content-container::-webkit-scrollbar {
  width: 8px;
}

.blog-content-container::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
}

.blog-content-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 4px;
}

.blog-content-container::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}

.blog-content {
  line-height: 1.6;
}

.blog-content h1 {
  font-size: 2rem;
  margin: 1.5rem 0;
  color: var(--leleo-vcard-color);
}

.blog-content h2 {
  font-size: 1.5rem;
  margin: 1.2rem 0;
  color: var(--leleo-vcard-color);
}

.blog-content h3 {
  font-size: 1.2rem;
  margin: 1rem 0;
  color: var(--leleo-vcard-color);
}

.blog-content p {
  margin: 1rem 0;
}

.blog-content ul, .blog-content ol {
  margin: 1rem 0 1rem 2rem;
}

.blog-content li {
  margin: 0.5rem 0;
}

/* 表格样式 */
.blog-content table {
  width: 100%;
  border-collapse: collapse;
  margin: 1.5rem 0;
  background-color: rgba(255, 255, 255, 0.05);
  border-radius: 8px;
  overflow: hidden;
}

.blog-content th,
.blog-content td {
  padding: 12px 16px;
  text-align: left;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.blog-content th {
  background-color: rgba(255, 255, 255, 0.1);
  font-weight: 600;
  color: var(--leleo-vcard-color);
}

.blog-content tr:hover {
  background-color: rgba(255, 255, 255, 0.05);
}

.blog-content tr:last-child td {
  border-bottom: none;
}

.blog-content code {
  background-color: #f5f5f5;
  padding: 0.2rem 0.4rem;
  border-radius: 4px;
  font-family: monospace;
}

/* 代码框容器 */
.blog-content pre {
  background-color: #1e1e1e;
  border-radius: 10px;
  overflow: hidden;
  margin: 1.5rem 0;
  position: relative;
  border: 1px solid rgba(255, 255, 255, 0.08);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.4);
}

/* 代码框标题栏 */
.blog-content pre::before {
  content: '';
  display: block;
  height: 32px;
  background: linear-gradient(180deg, #2d2d2d 0%, #252525 100%);
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
}

/* 代码框标题栏圆点 */
.blog-content pre::after {
  content: '';
  position: absolute;
  top: 10px;
  left: 12px;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: linear-gradient(135deg, #ff5f56, #ff4438);
  box-shadow: 20px 0 0 #ffbd2e, 40px 0 0 #27ca40;
}

/* 代码内容区域 */
.blog-content pre code {
  display: block;
  background-color: transparent;
  padding: 16px 20px;
  color: #d4d4d4;
  font-family: 'Consolas', 'Monaco', 'Courier New', 'Fira Code', monospace;
  font-size: 14px;
  line-height: 1.6;
  overflow-x: auto;
}

/* 行内代码 */
.blog-content code {
  background-color: rgba(255, 255, 255, 0.1);
  color: #e6e6e6;
  padding: 2px 6px;
  border-radius: 4px;
  font-family: 'Consolas', 'Monaco', 'Courier New', 'Fira Code', monospace;
  font-size: 0.9em;
}

/* 代码复制按钮样式 */
.copy-btn {
  position: absolute;
  top: 5px;
  right: 10px;
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 5px;
  padding: 4px 10px;
  cursor: pointer;
  color: #a0a0a0;
  font-size: 11px;
  font-weight: 500;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
}

.copy-btn:hover {
  background: rgba(255, 255, 255, 0.15);
  border-color: rgba(255, 255, 255, 0.2);
  color: #d0d0d0;
}

.copy-btn:active {
  background: rgba(255, 255, 255, 0.1);
}

.copy-btn.copied {
  background: rgba(76, 175, 80, 0.2);
  border-color: rgba(76, 175, 80, 0.3);
  color: #81c784;
}

.blog-content blockquote {
  border-left: 4px solid var(--leleo-vcard-color);
  padding-left: 1rem;
  margin: 1rem 0;
  font-style: italic;
}

.blog-content a {
  color: var(--leleo-vcard-color);
  text-decoration: none;
}

.blog-content a:hover {
  text-decoration: underline;
}
</style>