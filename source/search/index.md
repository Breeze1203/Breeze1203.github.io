---
title: Search
layout: page
---

<style>
/* 同样的样式，让搜索框和结果更美观 */
.search-input {
    width: 100%;
    padding: 12px 15px;
    font-size: 1em;
    border: 1px solid #ddd;
    border-radius: 4px;
    box-sizing: border-box;
    margin-bottom: 20px;
}
.search-results-item {
    margin-bottom: 20px;
    padding-bottom: 20px;
    border-bottom: 1px solid #eee;
}
.search-results-item a {
    text-decoration: none;
    color: #333;
    display: block;
}
.search-results-item a:hover h3 {
    color: #007bff;
}
.search-results-item h3 {
    margin-top: 0;
    margin-bottom: 10px;
}
.search-results-item p {
    margin: 0;
    color: #666;
}
#no-results {
    color: #888;
}
</style>

<div id="search-container">
    <input type="text" id="search-input" class="search-input" placeholder="输入关键词搜索...">
    <div id="search-results"></div>
</div>

<script>
    window.addEventListener('DOMContentLoaded', function() {
        const searchInput = document.getElementById('search-input');
        const resultsContainer = document.getElementById('search-results');
        let searchData = []; // 用来存储从 XML 解析出的文章数据

        // 1. 获取并解析 XML 数据
        fetch('/search.xml') // <-- 核心改动：获取 search.xml 文件
            .then(response => response.text())
            .then(str => new window.DOMParser().parseFromString(str, "text/xml"))
            .then(data => {
                // 将 XML 节点转换为 JavaScript 对象数组，方便后续处理
                const entries = data.querySelectorAll('entry');
                searchData = Array.from(entries).map(entry => {
                    const title = entry.querySelector('title').textContent;
                    const url = entry.querySelector('url').textContent;
                    const content = entry.querySelector('content').textContent;
                    // 移除 HTML 标签以便于创建摘要
                    const plainContent = content.replace(/<[^>]+>/g, ""); 
                    
                    return {
                        title: title,
                        url: url,
                        content: plainContent,
                        excerpt: plainContent.substring(0, 150) + '...' // 创建一个150字符的摘要
                    };
                });
            })
            .catch(error => {
                console.error('Error fetching or parsing search data:', error);
                resultsContainer.innerHTML = '<p>加载搜索索引失败，请稍后重试。</p>';
            });

        // 2. 监听输入框的输入事件
        searchInput.addEventListener('input', function(e) {
            const query = e.target.value.trim().toLowerCase();
            
            if (!query) {
                resultsContainer.innerHTML = '';
                return;
            }

            // 3. 在已加载的数据中进行搜索
            const results = searchData.filter(item => {
                const titleMatch = item.title.toLowerCase().includes(query);
                const contentMatch = item.content.toLowerCase().includes(query);
                return titleMatch || contentMatch;
            });

            // 4. 渲染搜索结果
            renderResults(results);
        });

        function renderResults(results) {
            if (results.length === 0) {
                resultsContainer.innerHTML = '<p id="no-results">没有找到相关结果</p>';
                return;
            }

            let resultHTML = '';
            results.slice(0, 10).forEach(result => { // 最多显示10条结果
                resultHTML += `
                    <div class="search-results-item">
                        <a href="${result.url}">
                            <h3>${result.title}</h3>
                            <p>${result.excerpt}</p>
                        </a>
                    </div>
                `;
            });
            resultsContainer.innerHTML = resultHTML;
        }
    });
</script>
