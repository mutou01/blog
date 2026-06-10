效果：
在你想要爬取的网页，使用Obsidian Web Clipper插件，将网页内容以.md文档格式，保存到本地Obsidian。
![[Pasted image 20260427162759.png]]

![[Pasted image 20260427162730.png]]
背景：
Agent由于权限、渲染、安全性等因素无法获取部分网页内容，即与人的信息源不统一，容易读取旧内容，并基于错误信息生成错误代码。
解决方案：
github上有非常好的全自动爬虫工具，[Firecrawl - Search, Scrape, and Interact with the Web for AI](https://www.firecrawl.dev/)，但不完全免费。
因此基于日常开发用量小，要求完全免费的核心需求，推荐一套‘基于Obsidian Web Clipper实现半自动爬取网页内容’的方案。
前置资源：
[Obsidian - Sharpen your thinking](https://obsidian.md/)
安装步骤：
1.安装Obsidian；
2.根据你的浏览器，从对应的官方商店安装 "Obsidian Web Clipper"；

- **Chrome/Chromium**: [Chrome 网上应用店](https://chromewebstore.google.com/detail/obsidian-web-clipper/cnjifjpddelmedmihgijeibhnjfabmlf)[](https://awesome.ecosyste.ms/projects/github.com%2Fobsidianmd%2Fobsidian-clipper)
    
- **Firefox** (含移动版): [Firefox 附加组件](https://addons.mozilla.org/zh-CN/firefox/addon/web-clipper-obsidian/)[](https://awesome.ecosyste.ms/projects/github.com%2Fobsidianmd%2Fobsidian-clipper)
    
- **Safari** (macOS/iOS/iPadOS): [App Store](https://apps.apple.com/us/app/obsidian-web-clipper/id6720708363)[](https://awesome.ecosyste.ms/projects/github.com%2Fobsidianmd%2Fobsidian-clipper)
    
- **Edge**: [Edge 加载项](https://microsoftedge.microsoft.com/addons/detail/obsidian-web-clipper/eigdjhmgnaaeaonimdklocfekkaanfme)
3.连接 Obsidian，安装后点击浏览器工具栏的扩展图标，它会提示你打开本地的 Obsidian 应用以完成授权连接；
更多模板与规则、接入AI模型的内容，有使用或者探索需求的推荐问AI，本文不过多介绍。
上述内容已实现免费爬取网页内容，将.md文档丢给Agent作为知识库的能力。