---
permalink: /
title: "About Me"
author_profile: true
redirect_from:
  - /about/
  - /about.html
---

{% comment %}
  Still missing (add when available):
    - City: author.location in _config.yml
    - Microsoft start month (Experience)

  Publication format (newest first); leave out anything still under anonymous review:
    - <span class="venue">Venue Year</span> [Paper Title](link)  
      Author A, **Yong Hu**, Author C

  News format (newest first, one item per line, no bullets; end every line
  except the last with <br>):
    <span class="news-date">Mon. YYYY</span> ...<br>
    <span class="news-date">Mon. YYYY</span> ...

  Older items can be folded:
    <details markdown="1">
    <summary>Show more news</summary>

    <span class="news-date">Mon. YYYY</span> ...

    </details>

  The Blog section below News is generated: it lists the 5 newest posts in
  _posts/ (and links to /blog/ once there are more). Nothing to edit here.
{% endcomment %}

I'm Yong Hu, an Applied Scientist II at [Microsoft](https://www.microsoft.com/), responsible for the Model Evolver of [Sico](https://github.com/microsoft/Sico). My current research focuses on vision-language models (VLMs), data synthesis, and embodied AI. I received my Master's degree in Software Engineering from [Beijing University of Posts and Telecommunications (BUPT)](https://www.bupt.edu.cn/) in 2025.

## News 🌟

<span class="news-date">Jul. 2026</span> We released [GELab-Zero-4B-preview-Sico-Evolution](https://huggingface.co/microsoft/GELab-Zero-4B-preview-Sico-Evolution), the first open model evolved with [Sico](https://github.com/microsoft/Sico)'s model evolution pipeline: task success rate from 39.8% to 82.9%!

{% if site.posts.size > 0 %}
## Blog ✍️

{% for post in site.posts limit: 5 %}{% assign mon = post.date | date: "%b" %}<span class="news-date">{{ mon }}{% if mon != "May" %}.{% endif %} {{ post.date | date: "%Y" }}</span> [{{ post.title }}]({{ post.url | relative_url }}){% unless forloop.last %}<br>{% endunless %}
{% endfor %}{% if site.posts.size > 5 %}<br>[All posts →]({{ "/blog/" | relative_url }}){% endif %}
{% endif %}

## Projects 🧰

- **[Sico](https://github.com/microsoft/Sico)**: an open-source Digital Worker platform for reliable execution, continuous co-evolution, and building enterprise AI assets. I work on its Model Evolver.
- **[GELab-Zero-4B-preview-Sico-Evolution](https://huggingface.co/microsoft/GELab-Zero-4B-preview-Sico-Evolution)**: a 4B GUI agent fine-tuned (LoRA) from [GELab-Zero-4B-preview](https://huggingface.co/stepfun-ai/GELab-Zero-4B-preview) on Microsoft Edge and Copilot UI trajectories with Sico's model evolution pipeline.

## Publications 📚

- <span class="venue">IJCNN 2026</span> [MSDDS: Multivariate Time Series Anomaly Detection via Multi-Scale Decomposition with Fusion and Dual-Stream Channel Transformer](https://ieeexplore.ieee.org/document/11729857)  
  Yuying Ma, Jiong Yu, Shu Li, **Yong Hu**

## Education 🎓

- **Beijing University of Posts and Telecommunications (BUPT)** <span class="cv-date">Sep. 2022 – Jun. 2025</span>  
  Master's in Software Engineering, School of Computer Science (National Pilot Software Engineering School)
- **China University of Geosciences (Wuhan)** <span class="cv-date">Sep. 2018 – Jun. 2022</span>  
  Bachelor's in Software Engineering

## Experience 💼

- **[Microsoft](https://www.microsoft.com/)** <span class="cv-date">2025 – Present</span>  
  Applied Scientist II, responsible for the Model Evolver of [Sico](https://github.com/microsoft/Sico)
- **[Microsoft](https://www.microsoft.com/)** <span class="cv-date">Nov. 2023 – Jun. 2025</span>  
  Software Engineer Intern, Azure Data  
  Built features for Microsoft Fabric Eventstream, including a visual event-processing editor that replaces hand-written SQL.
- **[Kuaishou](https://www.kuaishou.com/)** <span class="cv-date">May 2023 – Sep. 2023</span>  
  Software Engineer Intern  
  Built a Node.js toolkit that prefetches API requests to cut page latency, and helped build a micro-frontend platform.
- **[ByteDance](https://www.bytedance.com/)** <span class="cv-date">Jul. 2021 – Jan. 2022</span>  
  Software Engineer Intern  
  Built article submission via Lark for ByteDance's internal tech community, and sped up its page loads.
