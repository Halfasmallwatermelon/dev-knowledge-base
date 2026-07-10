# 现代 CSS

## Flexbox

```css
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 1rem;
}

.item {
  flex: 1;  /* 等分空间 */
  flex-shrink: 0;  /* 不缩小 */
}

/* 主轴方向 */
.row { flex-direction: row; }
.column { flex-direction: column; }
```

## Grid

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
}

/* 响应式 */
@media (max-width: 768px) {
  .grid {
    grid-template-columns: 1fr;
  }
}

/* 命名区域 */
.layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  grid-template-columns: 200px 1fr;
}
```

## CSS 变量

```css
:root {
  --primary-color: #3498db;
  --font-size: 16px;
}

.button {
  background: var(--primary-color);
  font-size: var(--font-size);
}

/* 暗色模式 */
@media (prefers-color-scheme: dark) {
  :root {
    --primary-color: #2980b9;
  }
}
```

## 动画

```css
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.animate {
  animation: fadeIn 0.3s ease-in-out;
}

/* 过渡 */
.button {
  transition: all 0.3s ease;
}

.button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}
```

## Tailwind CSS

```html
<div class="flex items-center justify-between p-4 bg-white rounded-lg shadow-md">
  <h2 class="text-xl font-bold">Title</h2>
  <button class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">
    Click me
  </button>
</div>
```