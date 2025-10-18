---
title: 零碎记录
layout: page
---

<div id="doodles-list">
    
</div>

<script>
window.addEventListener('DOMContentLoaded', function() {
    const resultsContainer = document.getElementById('doodles-list');
    const targetCategory = 'Essay';
    if (!resultsContainer) {
        console.error("未找到用于渲染结果的容器 #doodles-list。");
        return;
    }
    
    fetch('/search.xml')
        .then(response => {
            if (!response.ok) {
                throw new Error(`无法加载 search.xml，状态码: ${response.status}`);
            }
            return response.text();
        })
        .then(str => new window.DOMParser().parseFromString(str, "text/xml"))
        .then(data => {
            const doodlesData = [];
            const entries = data.querySelectorAll('entry');
    
            entries.forEach(entry => {
                const categoryNodes = entry.querySelectorAll('categories category');
                const hasTargetCategory = Array.from(categoryNodes).some(node => node.textContent.trim() === targetCategory);
    
                if (hasTargetCategory) {
                    const title = entry.querySelector('title')?.textContent || '无标题';
                    const url = entry.querySelector('url')?.textContent || '#';
                    let date = '';
                    // 1. 使用正则表达式匹配 URL 中的 YYYY/MM/DD 格式
                    const dateMatch = url.match(/(\d{4})\/(\d{2})\/(\d{2})/);
    
                    if (dateMatch) {
                        // 2. 如果匹配成功，则拼接成 YYYY-MM-DD 格式并创建日期对象
                        const dateString = `${dateMatch[1]}-${dateMatch[2]}-${dateMatch[3]}`;
                        date = new Date(dateString).toLocaleDateString('zh-CN', {
                            year: 'numeric',
                            month: 'long',
                            day: 'numeric'
                        });
                    } else {
                        // 3. [备用方案] 如果 URL 中没有日期，则尝试从 <updated> 或 <published> 标签获取
                        const dateElement = entry.querySelector('updated') || entry.querySelector('published');
                        if (dateElement) {
                            date = new Date(dateElement.textContent).toLocaleDateString('zh-CN', {
                                year: 'numeric',
                                month: 'long',
                                day: 'numeric'
                            });
                        }
                    }
                    // =======================================================
    
                    doodlesData.push({ title, url, date });
                }
            });
            
            resultsContainer.innerHTML = '';
    
            if (doodlesData.length === 0) {
                resultsContainer.innerHTML = '<p>暂无“' + targetCategory + '”分类下的零碎记录。</p>';
                return;
            }
    
            doodlesData.sort((a, b) => new Date(b.date) - new Date(a.date));
    
            const ul = document.createElement('ul');
            ul.className = 'doodles-list';
    
            doodlesData.forEach(item => {
                const li = document.createElement('li');
                
                const link = document.createElement('a');
                link.href = item.url;
                link.textContent = item.title;
    
                const dateSpan = document.createElement('span');
                dateSpan.className = 'doodle-date';
                dateSpan.textContent = item.date ? `${item.date}-` : '';
                li.appendChild(dateSpan); // 先添加日期
                li.appendChild(link);     // 再添加链接
                ul.appendChild(li);
            });
    
            resultsContainer.appendChild(ul);
        })
        .catch(error => {
            resultsContainer.innerHTML = `<p style="color: red;">加载零碎记录失败: ${error.message}</p>`;
            console.error("加载或解析 search.xml 时出错:", error);
        });
});
</script>