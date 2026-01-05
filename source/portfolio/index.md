---
title: 项目展示
date: 2024-01-01
type: "portfolio"
---

<link rel="stylesheet" href="https://npm.elemecdn.com/swiper@8/swiper-bundle.min.css" />
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@fancyapps/ui@5.0/dist/fancybox/fancybox.css"/>

<style>
  /* 全局居中 */
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
  /* 轮播容器 */
  .my-swiper {
    width: 100%;
    height: 350px;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    margin-bottom: 20px;
    background: #f9f9f9;
  }
  .swiper-slide {
    height: 100%;
    width: 100%;
  }
  /* Fancybox 链接容器 */
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
    object-fit: cover; /* 如果不想裁切，改成 contain */
    display: block;
    transition: transform 0.3s;
  }
  .fancybox-link:hover img {
    transform: scale(1.03);
  }
  /* 底部控制栏 */
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
    line-height: 1.6;
  }
  /* 重写 Swiper 按钮 */
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
  .ctrl-bar .swiper-button-next::after {
    font-size: 20px;
  }
  .custom-pagination {
    margin-top: 10px;
    display: flex;
    justify-content: center;
  }
  .swiper-pagination-bullet-active {
    background: #2bbc8a !important;
  }
</style>

<div class="project-card">
  <div class="project-title">公司成果 </div>

  <div class="swiper my-swiper">
    <div class="swiper-wrapper">
      <div class="swiper-slide">
        <a href="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/1.png" class="fancybox-link" data-fancybox="gallery-project-1" data-caption="Jasset 首页总览">
          <img src="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/1.png" alt="首页">
        </a>
      </div>
      <div class="swiper-slide">
         <a href="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/2.png" class="fancybox-link" data-fancybox="gallery-project-1" data-caption="详情页">
          <img src="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/2.png" alt="图2">
         </a>
      </div>
      <div class="swiper-slide">
         <a href="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/3.png" class="fancybox-link" data-fancybox="gallery-project-1" data-caption="详情页">
          <img src="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/3.png" alt="图3">
         </a>
      </div>
      <div class="swiper-slide">
         <a href="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/4.png" class="fancybox-link" data-fancybox="gallery-project-1" data-caption="详情页">
          <img src="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/4.png" alt="图4">
         </a>
      </div>
      <div class="swiper-slide">
         <a href="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/5.png" class="fancybox-link" data-fancybox="gallery-project-1" data-caption="详情页">
          <img src="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/5.png" alt="图5">
         </a>
      </div>
      <div class="swiper-slide">
         <a href="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/6.png" class="fancybox-link" data-fancybox="gallery-project-1" data-caption="详情页">
          <img src="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/6.png" alt="图6">
         </a>
      </div>
    </div>
  </div>

  <div class="ctrl-bar">
    <div class="swiper-button-prev"></div>
    <p class="project-desc">点击图片可放大查看细节</p>
    <div class="swiper-button-next"></div>
  </div>
  <div class="custom-pagination"></div>
</div>


<div class="project-card">
  <div class="project-title">个人项目</div>

  <div class="swiper my-swiper">
    <div class="swiper-wrapper">
      <div class="swiper-slide">
        <a href="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/1.png" class="fancybox-link" data-fancybox="gallery-project-2" data-caption="项目2截图A">
          <img src="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/1.png" alt="图1">
        </a>
      </div>
      <div class="swiper-slide">
        <a href="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/1.png" class="fancybox-link" data-fancybox="gallery-project-2" data-caption="项目2截图B">
          <img src="https://cdn.jsdelivr.net/gh/Breeze1203/portfolio@main/bw/1.png" alt="图2">
        </a>
      </div>
    </div>
  </div>

  <div class="ctrl-bar">
    <div class="swiper-button-prev"></div>
    <p class="project-desc">点击图片可放大查看细节</p>
    <div class="swiper-button-next"></div>
  </div>
  <div class="custom-pagination"></div>
</div>


<script src="https://npm.elemecdn.com/swiper@8/swiper-bundle.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@fancyapps/ui@5.0/dist/fancybox/fancybox.umd.js"></script>

<script>
  // 初始化 Swiper 轮播
  document.querySelectorAll('.project-card').forEach(function(card) {
    var swiperEl = card.querySelector('.my-swiper');
    var nextBtn = card.querySelector('.swiper-button-next');
    var prevBtn = card.querySelector('.swiper-button-prev');
    var paginationEl = card.querySelector('.custom-pagination');

    new Swiper(swiperEl, {
      loop: true, // 如果图片只有1张，loop:true 可能会有bug，建议至少放2张图
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