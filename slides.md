---
theme: seriph
background: https://cover.sli.dev
class: text-center
highlighter: shiki
transition: slide-left
routerMode: 'hash'
lineNumbers: true
info: |
  ## 深入理解patch过程

drawings:
  persist: false
css: unocss
title: 深入理解patch过程
---

# **Bug案例原因分享**

<div
v-motion
:initial="{ x: -80, opacity: 0}"
:enter="{ x: 0, opacity: 1,  scale: 1.5, transition: { delay: 100, duration: 2500 } }"
>
  <span class="color-orange text-xl">
    深入理解patch过程
  </span>
</div>

---
src: ./pages/share/bug展示.md
---

---
src: ./pages/share/bug1.md
---

---
src: ./pages/share/bug2.md
---
