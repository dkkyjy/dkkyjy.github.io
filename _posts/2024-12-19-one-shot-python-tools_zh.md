# 使用 uv run 和 Claude Projects 通过一次性提示构建 Python 工具

2024年12月19日

我写了很多关于我如何使用 Claude 通过 Claude Artifacts 构建一次性 HTML+JavaScript 应用程序的文章。我最近开始使用类似的模式来创建一次性 Python 实用程序，使用自定义的 Claude Project 并结合 uv 的依赖管理功能。
（在 LLM 术语中，“一次性”提示是指能在第一次尝试时就产生完整预期结果的提示。令人困惑的是，它有时也指包含单个期望输出格式示例的提示。这里我使用的是这两个定义中的第一个。）
我将从一个用这种方式构建的工具示例开始。
我今天又与 Amazon S3 进行了一轮较量，试图弄清楚为什么我某个存储桶中的一个文件无法通过公共 URL 访问。
出于沮丧，我用以下内容的变体提示了 Claude（完整对话记录在这里）：
我无法访问 EXAMPLE_S3_URL 处的文件。请使用 Click 和 boto3 为我编写一个 Python CLI 工具，该工具接受该格式的 URL，然后使用 boto3 中的所有技巧来尝试调试为什么该文件返回 404 错误。
它为我编写了这个脚本，这正是我所需要的。我像这样运行它：
uv run debug_s3_access.py \
  https://test-public-bucket-simonw.s3.us-east-1.amazonaws.com/0f550b7b28264d7ea2b3d360e3381a95.jpg
终端截图显示 S3 访问分析结果。命令：'$ uv run http://tools.simonwillison.net/python/debug_s3_access.py url-to-image'，后面是详细输出，显示存储桶存在（是）、区域（默认）、键存在（是）、存储桶策略（AllowAllGetObject）、存储桶所有者（swillison）、版本控制（未启用）、内容类型（image/jpeg）、大小（71683 字节）、最后修改时间（2024-12-19 03:43:30+00:00）和公共访问设置（全部为 False）
你可以在这里看到文本输出。
内联依赖和 uv run
关键的是，我不需要采取任何额外步骤来安装脚本所需的任何依赖项。这是因为脚本以这个神奇的注释开头：
```script
requires-python = ">=3.12"
dependencies = [
    "click",
    "boto3",
    "urllib3",
    "rich",
]
```
这是内联脚本依赖项的一个例子，这是 PEP 723 中描述并由 uv run 实现的一个功能。运行脚本会导致 uv 创建一个安装了这些依赖项的临时虚拟环境，一旦 uv 缓存被填充，这个过程只需要几毫秒。
即使脚本是通过 URL 指定的，这也能工作！任何安装了 uv 的人都可以运行以下命令（前提是你相信我还没有用恶意脚本替换它）来调试他们自己的 S3 存储桶：
uv run http://tools.simonwillison.net/python/debug_s3_access.py \
  https://test-public-bucket-simonw.s3.us-east-1.amazonaws.com/0f550b7b28264d7ea2b3d360e3381a95.jpg
在 Claude Project 的帮助下编写这些脚本
我现在能够一次性生成这样的脚本，是因为我设置了一个名为“Python app”的 Claude Project。项目可以有自定义指令，我利用这些指令来“教导”Claude 如何利用内联脚本依赖项：
您将 Python 工具编写为单个文件。它们总是以这个注释开头：
```script
requires-python = ">=3.12"
```
这些文件可以包含对库（如 Click）的依赖。如果包含，这些依赖项会列在同一个注释中的一个列表中（这里显示了两个依赖项）：
```script
requires-python = ">=3.12"
dependencies = [
    "click",
    "sqlite-utils",
]
```
这就是 Claude 可靠地生成功能齐全的 Python 工具（作为单个脚本）所需的一切，这些脚本可以直接运行，使用 Claude 选择包含的任何依赖项。
我之前并没有建议 Claude 在 debug_s3_access.py 脚本中使用 rich，但它还是决定使用了它！
我最近才开始试验这种模式，但它似乎效果很好。这是另一个例子——我的提示是：
创建一个 Starlette Web 应用程序，提供一个 API，您传入 ?url= 参数，它会去除所有 HTML 标签并仅返回文本，使用 beautifulsoup。
这是聊天记录和它生成的原始代码。您可以直接在您的机器上运行该服务器（它使用端口 8000），像这样：
uv run https://gist.githubusercontent.com/simonw/08957a1490ebde1ea38b4a8374989cf8/raw/143ee24dc65ca109b094b72e8b8c494369e763d6/strip_html.py
然后访问 http://127.0.0.1:8000/?url=https://simonwillison.net/ 查看其运行情况。
自定义指令
这里最让我感兴趣的模式是使用自定义指令或系统提示来向 LLM 展示如何实现其训练数据中可能不存在的新模式。uv run 问世还不到一年，但只需提供一个简短的示例就足以让模型编写出利用其功能的代码。
我有一套类似的自定义指令，用于创建单页 HTML 和 JavaScript 工具，同样在 Claude Project 中运行：
在制品中绝不使用 React——始终使用纯 HTML、原生 JavaScript 和 CSS，并尽量减少依赖。
CSS 应使用两个空格缩进，并应如下开始：
<style>
* {
  box-sizing: border-box;
}
输入框和文本区域字体大小应为 16px。字体应优先使用 Helvetica。
JavaScript 应使用两个空格缩进，并如下开始：
<script type="module">
// 这里的代码在第一级不应缩进
我 tools.simonwillison.net 网站上的大多数工具都是使用此自定义指令提示的各个版本创建的。
