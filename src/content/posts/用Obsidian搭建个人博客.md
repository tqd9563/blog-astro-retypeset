---
title: Obsidian搭建个人博客
published: 2026-01-18
created_time: 2026-01-18 20:57:34
description:
tags:
  - 文章
  - astro
  - zeabur
  - blog
category:
  - Articles
toc: true
share: true
abbrlink: blog-built-with-obsidian
---
我的这个个人博客是用obsidian +astro +zeabur搭建的，其中obsidian负责文章的编写，astro负责静态网站的编译生成，zeabur负责网站的托管运维。核心最主要用到obsidian中的两个插件：
- Image Converter：负责obsidian客户端侧的图片转化、压缩
- Enveloppe：方便地将笔记发布到指定的github仓库中，并支持个性化的自定义配置

本文主要是记录一下搭建博客过程中踩过的坑，希望对你有所帮助

一些参考文档：
- [Enveloppe 插件适配 Hugo 的配置](https://www.printlove.cn/github-publisher-hugo/)

# 选定一个主题
首先需要选择一个喜欢的博客主题，这里我用的是[retypeset](https://github.com/radishzzz/astro-theme-retypeset)这个主题。跟着相应的Getting Started流程走就行了，安装node.js，pnpm，以及相关依赖，最后本地能够成功启动即可。

# 用Zeabur托管网站
[Zeabur](https://zeabur.cn/)是一个托管平台，类似的还有Vercel, Netlify等等。选择zeabur纯粹是因为相对较新，域名还没有被严重污染。
Astro官方提供了各式各样的云托管平台[部署指南](https://docs.astro.build/en/guides/deploy/)，在里面找到Zeabur的部分，照着做就好了。
> [!tip]
> 一般来说，服务端渲染适配器是不需要的安装的，除非你的博客中有很多需要服务器参与计算的 ，类似于实时点赞之类的功能。

流程走到最后，用zeabur给你绑定的域名访问，能够看到下面的样子就说明成功了（你使用的主题的初始模板）
![](../../../src/assets/用Obsidian搭建个人博客/image.webp)

# 设置你的Enveloppe
Enveloppe插件是这套博客搭建流程中的核心部分，它将博客的编写和编译渲染完全解耦，你只需要在你的Obsidian客户端中沉浸式地写你的博客文章，然后通过Enveloppe插件，就可以将你的Markdown文件以及一些附件资源（如图片、音频等）一键同步到你的Astro主题所在的Github仓库，这个仓库存的是HTML、CSS以及一些博客站点的配置文件，**其结构可以和你的Obsidian Vault完全不同！**

首先需要在Github点击头像，开发者设置中生成一个classic的token，权限将repo全部勾选上。然后复制这个token，在Enveloppe插件设置中配置下面这些内容
## GitHub config
只需要修改你的Github用户名，以及对应的Astro主题仓库的名字即可
![](../../../src/assets/用Obsidian搭建个人博客/image-2.webp)
## File paths
这里设置的是你的Obsidian Vault中的markdown文件（注意，仅针对markdown文件），将被推送到目标git仓库中的什么目录下。
由于Astro博客的组织形式是固定的，也就是说你的博客文章必须放在指定的目录下，网站才会渲染成功。因此这里只需要在Default folder配置项中填写你的.md文件的目录。以我的主题为例，我配置的是`src/content/posts`

![](../../../src/assets/用Obsidian搭建个人博客/image-3.webp)

## Content
这里设置的是你的.md文件中的具体内容，在推送至Astro主题仓库时候会做的一些转换（但这不会修改在Obsidian Vault中的.md文件样式）
**这里我们一定需要把Wikilinks to MDlinks勾选上**，因为在Obsidian里往md文件复制一张图片时，默认是用双链的形式，即`![[image_name]]`的形式。如果不勾选这个，Astro那边看到的也是这一串代码，会导致无法准确读取对应的图片资源（因为这是Obsidian的专属语法）
打开勾选后，会强制将`![[image_name]]`转换成标准的md图片语法`![name](url)`

![](../../../src/assets/用Obsidian搭建个人博客/image-4.webp)

## Attachment & embeds
>[!important]
>配置的重点来了！！！

这里配置的是除.md以外的其他文件（如图片、音频、视频等等），将会被推送到Astro主题仓库中的哪个路径下。

这里很关键的一点就是，**你的博客里的图片资源是图床，还是本地文件**。如果是前者，这个配置可以先跳过，因为图床本身是一个url，obsidian不会识别到任何的.png或是.jpg的图片文件，也就不存在推送这一行为了。但是图床可能存在的问题是：
- 图片提及过大，加载速度缓慢
- 访问延迟高，加载速度慢

之前我的博客是用的Picgo +Github的图床管理图片的，图片一多起来，整个博客就卡卡的，加载图片需要好久，浏览体验极差。。于是这次痛定思痛，准备改成本地加载图片的方式（直接复制图片到md文件中）

首先，**Astro有一个图像优化的特性**，它默认会对图片开启懒加载`loading="lazy"`，这样即使你的页面有 100 张图，用户打开页面时，浏览器也只会加载 **首屏（Viewport）** 可见的那 1-2 张图，其余的图片会等到用户滚动到那里时才加载。同时它还会对图片尺寸和格式做优化，转为webp，针对手机生成小图，针对4K屏生成大图。

但是，Astro的上述特性实现基于它能够成功扫描到图片资源，具体逻辑如下：
- 首先扫描markdown文件，捕捉形如`![]()`的代码段
- 然后判断这是一个相对路径还是绝对路径：
	- 如果是相对路径：Astro会去读取这个文件，获取宽高，转换格式，最终生成优化后的`<img>`标签（带有懒加载的）
	- 如果是绝对路径：Astro会认为这是一个外部文件，原样输出，结果是没有懒加载和尺寸类型转换！！！

因此，首先我们需要保证，**你的Obsidian的markdown文件中，引用图片的语法必须是相对路径引用**。要实现这一点，我们需要用到Image Converter插件，后文会有提及如何配置。

除此以外，我们还需要保证的是，**在Obsidian Vault中，md文件与图片文件的相对层级关系，和Astro Repo中md文件与图片文件的相对层级关系完全一致！！** 这一点至关重要，否则会导致Astro找不到图片资源，整个网站挂掉。

这里先放上我的配置，后面会具体讲这么配的含义以及为什么这么配：
选择Override attachments path配置项，新建：
- Path or extension：`/^src\/assets\/(.*)\/(.*)\.webp$/`
- Destination：`src/assets/$1/$2.webp`


捋一捋整个流程的顺序就能弄明白为什么需要这一点
1. 在本地vault中，md文件通过相对路径引用img图片：`![name](../images/imagename.png)`
2. 用Enveloppe插件同步后，md文件被传递到了`src/content/posts/article.md`，并且此时md文件中引用图片的代码仍然保持不变：`![name](../images/imagename.png)
3. 倒推一下，如果想要图片成功识别到，那这张图片应该被推送到`src/content/images/imagename.png`这个路径下。


# 设置你的Image Converter
> [!note]
> 这一步非必需，如果你用自己的图床管理图片资源，就不需要安装这个插件。只有图片资源全部放在obsidian本地的才需要。

Image Converter插件可以在往markdown文件里复制一张图片时，自动进行压缩裁剪，并修改其保存的路径和文件名。
这里我们需要改的配置如下：
## Folder
更改图片处理后保存的路径，选择+ Add New，Location选择Custom自定义，自定义路径写`src/assets/{notename}`。这样配的好处是，同一个博客下的图片资源，会被归类在同一个目录下，便于后续管理维护（`{notename}`是插件预定义好的一个变量，表示当前的文件名）

例如我的这篇博客中的图片文件都会在转换后保存在`src/assets/用 Obsidian+Astro+Zeabur搭建个人博客/`目录下。

![](../../../src/assets/用Obsidian搭建个人博客/image-1.webp)

## Filename
选择自定义名字，注意If an output file already exists选项改为添加数字后缀，防止图片被覆盖。

![](../../../src/assets/用Obsidian搭建个人博客/image-5.webp)

此时注意，经过Folder和Filename的配置后，我们在md文件中添加的图片，将会被保存为`src/assets/{notename}/image.webp`。

## Conversion
这里可以设置对图片进行的转换操作，包括转换精度（通常75-80范围内，肉眼和原图看不出什么区别），转换格式（推荐webp），裁剪（宽度1920）

## Link format
注意，这里需要修改图片链接格式为Markdown，路径模式为相对路径，具体原因在前面已经讲过了，目的是为了Astro的懒加载和图片性能优化特性能够顺利执行。

![](../../../src/assets/用Obsidian搭建个人博客/image-7.webp)


看到这里，再回过头去看Enveloppe插件的Attachment & embeds配置，其中的Override attachments path配置项，我的配置是这样的：
- Path or extension：`/^src\/assets\/(.*)\/(.*)\.webp$/`
- Destination：`src/assets/$1/$2.webp`

这个配置的含义是：将Obsidian Vault中所有的`/src/assets\任意文件夹名\任意图片名.webp`这样路径下的图片文件，全都推送到Astro Repo中的`src/assets/任意文件夹名/任意图片名.webp`路径下。

这样配置后，对于一篇`{notename}.md`笔记文件，如果我们往里面添加一张图片，会发生什么？
1. 首先图片经过Image Converter插件的转换、裁剪，保存到了路径`src/assets/{notename}/image.webp`下
2. 由于在Link format配置中配置了链接格式为markdown，路径格式为相对路径，因此在`{notename}.md`中，对应的图片引用代码为：`![](某相对路径/src/assets/{notename}/image.webp`
3. 经过Enveloppe插件同步，在Astro Repo中，`{notename}.md`笔记文件的绝对路径为`src/content/posts/{notename.md}`
4. 经过Enveloppe插件同步，在Astro Repo中，`src/assets/{notename}/image.webp`这个图片的绝对路径为`src/assets/{notename}/image.webp`

现在，由于Astro Repo中的md和webp文件路路径都已固定，因此可以推导出能够解析图片成功的相对路径代码为：
`../../../src/assets/{notename}/image.webp`
因此，只需要保证：我们的md文件与图片文件，相较于根目录的深度是一致的，并且距离根目录有3层目录即可。

最终我们要做的是，保证我们的Obsidian Vault的结构层级是三层深，即`layer1/layer2/layer3/title.md`，这样就可以使得图片完美被渲染。
