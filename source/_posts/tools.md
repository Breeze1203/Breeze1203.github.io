---
title: 在线工具
date: 2026-08-12 21:30:00
layout: page
---

<style>
.tool-selector {
  margin-bottom: 15px;
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.tool-btn {
  padding: 5px 12px;
  background: transparent;
  border: 1px solid #666;
  color: inherit;
  cursor: pointer;
  border-radius: 4px;
  font-size: 0.9em;
}

.tool-btn.active {
  border-color: #2bbc8a;
  color: #2bbc8a;
  font-weight: bold;
}

.tool-iframe {
  width: 100%;
  height: 75vh;
  border: 1px solid #444;
  border-radius: 6px;
  background: #ffffff; /* 防止某些无背景颜色的工具透出暗黑模式背景 */
}
</style>

<div class="tool-selector">
  <button class="tool-btn active" onclick="loadTool(this, 'https://breeze1203.github.io/tools/explain_tool/')">SQL 执行计划</button>
</div>

<iframe id="tool-frame" class="tool-iframe" src="https://breeze1203.github.io/tools/explain_tool/"></iframe>
<script>
function loadTool(btn, url) {
  document.getElementById('tool-frame').src = url;
  
  // 切换按钮高亮样式
  const btns = document.querySelectorAll('.tool-btn');
  btns.forEach(b => b.classList.remove('active'));
  btn.classList.add('active');
}
</script>
