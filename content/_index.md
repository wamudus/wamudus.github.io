---
title: "K8s 学习笔记"
---

<div style="max-width: 800px; margin: 0 auto; padding: 2rem 1rem;">

  <!-- 头部 -->
  <div style="text-align: center; margin-bottom: 3rem;">
    <h1 style="font-size: 2rem; margin-bottom: 0.5rem;">K8s 学习笔记</h1>
    <p style="color: #666; font-size: 1rem;">备考 CKA，记录学习过程</p>
    
    <div style="margin-top: 1rem; font-size: 0.9rem;">
      <span style="background: #e8f5e9; color: #2e7d32; padding: 0.2rem 0.6rem; border-radius: 4px; margin: 0 0.2rem;">RHCA ✓</span>
      <span style="background: #fff3e0; color: #ef6c00; padding: 0.2rem 0.6rem; border-radius: 4px; margin: 0 0.2rem;">CKA 备考中</span>
    </div>
  </div>

  <!-- 文章列表 -->
  <h2 style="border-bottom: 2px solid #eee; padding-bottom: 0.5rem; font-size: 1.2rem; color: #333;">学习日志</h2>

  {{ range .Site.RegularPages.ByDate.Reverse }}
  <article style="margin-bottom: 2rem; padding-bottom: 1.5rem; border-bottom: 1px solid #f0f0f0;">
    <div style="display: flex; justify-content: space-between; align-items: baseline; margin-bottom: 0.5rem;">
      <a href="{{ .RelPermalink }}" style="font-size: 1.1rem; font-weight: 600; color: #1a1a1a; text-decoration: none;">
        {{ .Title }}
      </a>
      <time style="font-size: 0.85rem; color: #999; white-space: nowrap;">{{ .Date.Format "2006-01-02" }}</time>
    </div>
    <p style="margin: 0; color: #555; font-size: 0.95rem; line-height: 1.5;">
      {{ .Summary | truncate 100 }}
    </p>
    {{ if .Params.tags }}
    <div style="margin-top: 0.5rem;">
      {{ range .Params.tags | first 3 }}
        <span style="font-size: 0.8rem; color: #666; background: #f5f5f5; padding: 0.1rem 0.5rem; border-radius: 3px; margin-right: 0.3rem;">{{ . }}</span>
      {{ end }}
    </div>
    {{ end }}
  </article>
  {{ end }}

</div>