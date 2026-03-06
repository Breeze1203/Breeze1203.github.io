---
title: 项目展示
date: 2024-01-01
type: "portfolio"
---

<link rel="stylesheet" href="/lib/swiper/swiper-bundle.min.css" />
<link rel="stylesheet" href="/lib/fancybox/fancybox.css"/>

<style>
  .project-card {
    margin-bottom: 60px;
    padding-bottom: 30px;
    border-bottom: 1px dashed #e0e0e0;
    text-align: center;
  }
  .project-title {
    font-size: 1.5rem;
    font-weight: bold;
    margin-bottom: 20px;
    color: #2bbc8a;
  }
  .my-swiper {
    width: 100%;
    height: 350px;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    margin-bottom: 20px;
    background: #f9f9f9;
  }
  .swiper-slide { height: 100%; width: 100%; }

  .fancybox-link {
    display: block;
    width: 100%;
    height: 100%;
    cursor: zoom-in;
    overflow: hidden;
    border-radius: 8px;
  }
  .fancybox-link img {
    width: 100%;
    height: 100%;
    object-fit: cover; 
    display: block;
    transition: transform 0.3s;
  }
  .fancybox-link:hover img { transform: scale(1.03); }

  .ctrl-bar {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 15px;
    margin-top: 15px;
    position: relative;
    padding: 0 10px;
  }
  .project-desc {
    margin: 0;
    max-width: 70%; 
    font-size: 0.95rem;
    color: #666;
  }
  .ctrl-bar .swiper-button-prev, 
  .ctrl-bar .swiper-button-next {
    position: static !important;
    margin-top: 0 !important;
    width: 30px;
    height: 30px;
    color: #2bbc8a;
    font-weight: bold;
    flex-shrink: 0;
    cursor: pointer;
  }
  .ctrl-bar .swiper-button-prev::after,
  .ctrl-bar .swiper-button-next::after { font-size: 20px; }
  .custom-pagination {
    margin-top: 10px;
    display: flex;
    justify-content: center;
  }
  .swiper-pagination-bullet-active { background: #2bbc8a !important; }
</style>

<div id="portfolio-container"></div>

<script src="/lib/swiper/swiper-bundle.min.js"></script>
<script src="/lib/fancybox/fancybox.umd.js"></script>

<script>
  const GITHUB_USER = "Breeze1203";
  const REPO_NAME = "portfolio";
  const BRANCH = "main";
  // 核心：项目列表配置
  const projects = [
    {
      id: "bw",                   
      folderName: "bw",          
      title: "信息管理系统",         
      imgCount: 5,                 
      desc: "点击图片可放大" 
    },
    {
      id: "xcx",
      folderName: "xcx",          
      title: "享乐享智家与手持机",
      imgCount: 6,                
      desc: "点击图片可放大"
    },
    {
      id: "yd",
      folderName: "yd",          
      title: "元点ai",
      imgCount: 7,                
      desc: "点击图片可放大"
    },
    {
      id: "person",
      folderName: "person",        
      title: "个人项目作品",
      imgCount: 15,                
      desc: "点击图片可放大"
    }
  ];
  // ============================================================

  // 1. 生成 HTML 函数
  const container = document.getElementById('portfolio-container');
  const baseUrl = `https://cdn.jsdelivr.net/gh/${GITHUB_USER}/${REPO_NAME}@${BRANCH}`;

  let finalHtml = "";

  projects.forEach(proj => {
    let slidesHtml = "";
    
    // 内层循环：生成图片 Slide
    for (let i = 1; i <= proj.imgCount; i++) {
      let imgUrl = `${baseUrl}/${proj.folderName}/${i}.png`; 
      
      slidesHtml += `
        <div class="swiper-slide">
          <a href="${imgUrl}" 
             class="fancybox-link" 
             data-fancybox="gallery-${proj.id}" 
             data-caption="${proj.title} - 图${i}">
            <img src="${imgUrl}" loading="lazy" alt="${proj.title} ${i}">
          </a>
        </div>
      `;
    }
    finalHtml += `
      <div class="project-card">
        <div class="project-title">${proj.title}</div>
        
        <div class="swiper my-swiper">
          <div class="swiper-wrapper">
            ${slidesHtml}
          </div>
        </div>
    
        <div class="ctrl-bar">
          <div class="swiper-button-prev"></div>
          <p class="project-desc">${proj.desc}</p>
          <div class="swiper-button-next"></div>
        </div>
        <div class="custom-pagination"></div>
      </div>
    `;
  });

  // 将生成的 HTML 插入页面
  container.innerHTML = finalHtml;
  document.querySelectorAll('.project-card').forEach(function(card) {
    var swiperEl = card.querySelector('.my-swiper');
    var nextBtn = card.querySelector('.swiper-button-next');
    var prevBtn = card.querySelector('.swiper-button-prev');
    var paginationEl = card.querySelector('.custom-pagination');

    new Swiper(swiperEl, {
      loop: true,
      // 懒加载优化，防止一次性加载太多图片卡顿
      lazy: true, 
      navigation: {
        nextEl: nextBtn,
        prevEl: prevBtn,
      },
      pagination: {
        el: paginationEl,
        clickable: true,
      }
    });
  });

  // 初始化 Fancybox
  Fancybox.bind("[data-fancybox]", {
    toolbar: "auto",
  });
</script>
