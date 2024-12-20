## 2024.12.21 0:31

### 【lstk】参数分析
【lstk】这个参数是全局线条。
在 `convert.py` 中，第 449 行，注销掉。
```python
# ops += f"ET q 1 0 0 1 {l.pts[0][0]:f} {l.pts[0][1]:f} cm [] 0 d 0 J {l.linewidth:f} w 0 0 m {l.pts[1][0] - l.pts[0][0]:f} {l.pts[1][1] - l.pts[0][1]:f} l S Q BT "
```
但是有一点问题，就是表格没有了第一横行的横线。
引用的格式没了，全堆一起了，实际上，引用可以不用翻译。

## 2024.12.20

### 调试记录
1. 在 Windows 环境下，直接双击安装 `setip.bat`，直接安装了 Python 的环境。安装完可以直接运行，会自行下载 YOLO 布局模型的权重。随后自己运行，那么使用以下命令：
   ```shell
   .\pdf2zh_dist\python.exe pdf2zh\pdf2zh.py -i
   ```
   即在下载的 Python 环境里运行 `pdf2zh.py` 主程序。

2. 打开的可视化界面，提供了完善的功能。这里，为了免费的翻译，选择 Bing，反应还可以。如果使用 DeepLX，会比较慢，而且需要下载代理挂在后台。API 会延时，一段时间后才能行。

3. 翻译后的 PDF，个别排版会有些重影，放的位置重合了，个别语句会被检测为不用翻译的部分。总体功能可用。

4. 在后面，突然出现了，文档翻译还可以，但是全部都有线条，好像是文字下划线，将所有文字下面连接到了一起，排版变得奇怪，第一次用不会这样。难说原因。

### 各个文件的功能
1. `pdf2zh.py`：主程序，解析参数。
2. `high_level.py`：主要 pipeline，`translate` 函数是整体流程。
   - `translate_stream` 是处理文档函数。
     1. 文档交叉引用表中，字体的更新。交叉引用表是 PDF 文件中用于管理和定位对象的关键数据结构。它不仅记录了对象的位置，还管理了对象的状态。
     2. `translate_patch`：
        1. `TranslateConverter` 类用于翻译。
        2. `PDFPageInterpreterEx` 类实例化：interpreter。
        3. 初始化后，先确定 pages 数量，随后解析 PDF：
           ```python
           parser = PDFParser(inf)
           doc = PDFDocument(parser)
           ```
           逐页进行解析：
           - 先取当前页的图像，给 model 进行推理，得出页面布局，布局结果里有类别：
             ```python
             page_layout = model.predict(image, imgsz=int(pix.height / 32) * 32)[0]
             vcls = ["abandon", "figure", "table", "isolate_formula", "formula_caption"]
             ```
             根据推理出来的类别，对 box 分别进行操作：
             ```python
             page_layout.names[int(d.cls)] in vcls:
             ```
           - 处理后，新建一个 xref 存放新指令流：
             ```python
             page.page_xref = doc_zh.get_new_xref()  # hack 插入页面的新 xref
             doc_zh.update_object(page.page_xref, "<<>>")
             doc_zh.update_stream(page.page_xref, b"")
             doc_zh[page.pageno].set_contents(page.page_xref)
             ```
           - 解析器来进行解析。interpreter.process_page(page)

3. `pdfinterp.py`：pdf 解析器。继承自 pdfminer.pdfinterp 的函数，重构成 PDFPageInterpreterEx。
   - `process_page` 是主要函数，页面渲染，设置字体，可能用上页面旋转角度。
   - `render_contents` 方法渲染内容流。
   - `init_resources` 方法准备字体和 XObject 资源，并设置颜色空间。
   - `execute` 解析执行 pdf 中的内容流。根据关键字指令，与库中对照返回操作符。

   后面的函数好像没用。
   - `do_S` 方法处理路径描边，过滤非公式线条，`do_f`, `do_F`, `do_f_a`, `do_B`, `do_B_a` 方法处理路径填充和描边。
   - `do_SCN`, `do_scn`, `do_SC`, `do_sc` 方法设置描边和非描边操作的颜色。
   - `do_Do` 方法重载 XObject 的 obj_patch。

   `execute` 方法解析并执行 PDF 内容流中的操作符。

4. `doclayout.py`：pdf 布局解析，主要用在 `pdf2zh.py` 中。
5. `convert.py`：`TranslateConverter` 翻译后布局的解析。之前还先重构一个 `PDFConverterEx`，来自原始库的重构。
   - `receive_layout` 主要处理函数，解析布局，翻译处理。
   - 主要过程：原文档解析，翻译，新文档排版。
   - 主要区分：文字、公式、线条、图表。用了几个栈，分别存储文字，公式，判断是否为公式，并使用 `vflag` 函数匹配公式字体。

   以及三个函数：
   - 【lstk】全局线条，可能是这个出的问题。

6. `translate.py`：翻译，调用 api。
7. `backend.py`：是 flask 的后端。
8. `cache.py`：ui 界面的后端。

s_mono, s_dual = translate_stream(s_raw, **locals())
locals 获取当前作用域内所有局部变量。

<div align="center">

English | [简体中文](docs/README_zh-CN.md) | [日本語](docs/README_ja-JP.md)

<img src="./docs/images/banner.png" width="320px"  alt="PDF2ZH"/>

<h2 id="title">PDFMathTranslate</h2>

<p>
  <!-- PyPI -->
  <a href="https://pypi.org/project/pdf2zh/">
    <img src="https://img.shields.io/pypi/v/pdf2zh"></a>
  <a href="https://pepy.tech/projects/pdf2zh">
    <img src="https://static.pepy.tech/badge/pdf2zh"></a>
  <a href="https://hub.docker.com/repository/docker/byaidu/pdf2zh">
    <img src="https://img.shields.io/docker/pulls/byaidu/pdf2zh"></a>
  <a href="https://gitcode.com/Byaidu/PDFMathTranslate/overview">
    <img src="https://gitcode.com/Byaidu/PDFMathTranslate/star/badge.svg"></a>
  <a href="https://huggingface.co/spaces/reycn/PDFMathTranslate-Docker">
    <img src="https://img.shields.io/badge/%F0%9F%A4%97-Online%20Demo-FF9E0D"></a>
  <a href="https://www.modelscope.cn/studios/AI-ModelScope/PDFMathTranslate">
    <img src="https://img.shields.io/badge/ModelScope-Demo-blue"></a>
  <a href="https://github.com/Byaidu/PDFMathTranslate/pulls">
    <img src="https://img.shields.io/badge/contributions-welcome-green"></a>
  <a href="https://t.me/+Z9_SgnxmsmA5NzBl">
    <img src="https://img.shields.io/badge/Telegram-2CA5E0?style=flat-squeare&logo=telegram&logoColor=white"></a>
  <!-- License -->
  <a href="./LICENSE">
    <img src="https://img.shields.io/github/license/Byaidu/PDFMathTranslate"></a>
</p>

<a href="https://trendshift.io/repositories/12424" target="_blank"><img src="https://trendshift.io/api/badge/repositories/12424" alt="Byaidu%2FPDFMathTranslate | Trendshift" style="width: 250px; height: 55px;" width="250" height="55"/></a>

</div>

PDF scientific paper translation and bilingual comparison.

- 📊 Preserve formulas, charts, table of contents, and annotations _([preview](#preview))_.
- 🌐 Support [multiple languages](#language), and diverse [translation services](#services).
- 🤖 Provides [commandline tool](#usage), [interactive user interface](#gui), and [Docker](#docker)

Feel free to provide feedback in [GitHub Issues](https://github.com/Byaidu/PDFMathTranslate/issues), [Telegram Group](https://t.me/+Z9_SgnxmsmA5NzBl) or [QQ Group](https://qm.qq.com/q/DixZCxQej0).

<h2 id="updates">Updates</h2>

- [Dec. 19 2024] Non-PDF/A documents are now supported using `-cp` _(by [@reycn](https://github.com/reycn))_
- [Dec. 13 2024] Additional support for backend by _(by [@YadominJinta](https://github.com/YadominJinta))_
- [Dec. 10 2024] The translator now supports OpenAI models on Azure _(by [@yidasanqian](https://github.com/yidasanqian))_

<h2 id="preview">Preview</h2>

<div align="center">
<img src="./docs/images/preview.gif" width="80%"/>
</div>

<h2 id="demo">Online Service 🌟</h2>

You can try our application out using either of the following demos:

- [Public free service](https://pdf2zh.com/) online without installation _(recommended)_.
- [Demo hosted on HuggingFace](https://huggingface.co/spaces/reycn/PDFMathTranslate-Docker)
- [Demo hosted on ModelScope](https://www.modelscope.cn/studios/AI-ModelScope/PDFMathTranslate) without installation.

Note that the computing resources of the demo are limited, so please avoid abusing them.

<h2 id="install">Installation and Usage</h2>

### Methods

For different use cases, we provide four distinct methods to use our program:

<details open>
  <summary>1. Commandline</summary>

1. Python installed (3.8 <= version <= 3.12)
2. Install our package:

   ```bash
   pip install pdf2zh
   ```

3. Execute translation, files generated in [current working directory](https://chatgpt.com/share/6745ed36-9acc-800e-8a90-59204bd13444):

   ```bash
   pdf2zh document.pdf
   ```

</details>

<details>
  <summary>2. Portable (w/o Python installed)</summary>

1. Download [setup.bat](https://raw.githubusercontent.com/Byaidu/PDFMathTranslate/refs/heads/main/script/setup.bat)

2. Double-click to run.

</details>

<details>
  <summary>3. Graphic user interface</summary>
1. Python installed (3.8 <= version <= 3.12)
2. Install our package:

```bash
pip install pdf2zh
```

3. Start using in browser:

   ```bash
   pdf2zh -i
   ```

4. If your browswer has not been started automatically, goto

   ```bash
   http://localhost:7860/
   ```

   <img src="./docs/images/gui.gif" width="500"/>

See [documentation for GUI](./docs/README_GUI.md) for more details.

</details>

<details>
  <summary>4. Docker</summary>

1. Pull and run:

   ```bash
   docker pull byaidu/pdf2zh
   docker run -d -p 7860:7860 byaidu/pdf2zh
   ```

2. Open in browser:

   ```
   http://localhost:7860/
   ```

For docker deployment on cloud service:

<div>
<a href="https://www.heroku.com/deploy?template=https://github.com/Byaidu/PDFMathTranslate">
  <img src="https://www.herokucdn.com/deploy/button.svg" alt="Deploy" height="26"></a>
<a href="https://render.com/deploy">
  <img src="https://render.com/images/deploy-to-render-button.svg" alt="Deploy to Koyeb" height="26"></a>
<a href="https://zeabur.com/templates/5FQIGX?referralCode=reycn">
  <img src="https://zeabur.com/button.svg" alt="Deploy on Zeabur" height="26"></a>
<a href="https://app.koyeb.com/deploy?type=git&builder=buildpack&repository=github.com/Byaidu/PDFMathTranslate&branch=main&name=pdf-math-translate">
  <img src="https://www.koyeb.com/static/images/deploy/button.svg" alt="Deploy to Koyeb" height="26"></a>
</div>

</details>

### Unable to install?

The present program needs an AI model(`wybxc/DocLayout-YOLO-DocStructBench-onnx`) before working and some users are not able to download due to network issues. If you have a problem with downloading this model, we provide a workaround using the following environment variable:

```shell
set HF_ENDPOINT=https://hf-mirror.com
```

If the solution does not work to you / you encountered other issues, please refer to [frequently asked questions](https://github.com/Byaidu/PDFMathTranslate/wiki#-faq--%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98).

<h2 id="usage">Advanced Options</h2>

Execute the translation command in the command line to generate the translated document `example-mono.pdf` and the bilingual document `example-dual.pdf` in the current working directory. Use Google as the default translation service.

<img src="./docs/images/cmd.explained.png" width="580px"  alt="cmd"/>

In the following table, we list all advanced options for reference:

| Option         | Function                                                                                                      | Example                                        |
| -------------- | ------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| files          | Local files                                                                                                   | `pdf2zh ~/local.pdf`                           |
| links          | Online files                                                                                                  | `pdf2zh http://arxiv.org/paper.pdf`            |
| `-i`           | [Enter GUI](#gui)                                                                                             | `pdf2zh -i`                                    |
| `-p`           | [Partial document translation](https://github.com/Byaidu/PDFMathTranslate/blob/main/docs/ADVANCED.md#partial) | `pdf2zh example.pdf -p 1`                      |
| `-li`          | [Source language](https://github.com/Byaidu/PDFMathTranslate/blob/main/docs/ADVANCED.md#languages)            | `pdf2zh example.pdf -li en`                    |
| `-lo`          | [Target language](https://github.com/Byaidu/PDFMathTranslate/blob/main/docs/ADVANCED.md#languages)            | `pdf2zh example.pdf -lo zh`                    |
| `-s`           | [Translation service](https://github.com/Byaidu/PDFMathTranslate/blob/main/docs/ADVANCED.md#services)         | `pdf2zh example.pdf -s deepl`                  |
| `-t`           | [Multi-threads](https://github.com/Byaidu/PDFMathTranslate/blob/main/docs/ADVANCED.md#threads)                | `pdf2zh example.pdf -t 1`                      |
| `-o`           | Output dir                                                                                                    | `pdf2zh example.pdf -o output`                 |
| `-f`, `-c`     | [Exceptions](https://github.com/Byaidu/PDFMathTranslate/blob/main/docs/ADVANCED.md#exceptions)                | `pdf2zh example.pdf -f "(MS.*)"`               |
| `-cp`          | Compatibility Mode                                                                                            | `pdf2zh example.pdf --compatible`              |
| `--share`      | Public link                                                                                                   | `pdf2zh -i --share`                            |
| `--authorized` | Authorization                                                                                                 | `pdf2zh -i --authorized users.txt [auth.html]` |
| `--prompt`     | [Custom Prompt](https://github.com/Byaidu/PDFMathTranslate/blob/main/docs/ADVANCED.md#prompt)                 | `pdf2zh --prompt [prompt.txt]`                 |

For detailed explanations, please refer to our document about [Advanced Usage](./docs/ADVANCED.md) for a full list of each option.

<h2 id="downstream">Secondary Development (APIs)</h2>

For downstream applications, please refer to our document about [API Details](./docs/APIS.md) for futher information about:

- [Python API](./docs/APIS.md#api-python), how to use the program in other Python programs
- [HTTP API](./docs/APIS.md#api-http), how to communicate with a server with the program installed

<h2 id="todo">TODOs</h2>

- [ ] Parse layout with DocLayNet based models, [PaddleX](https://github.com/PaddlePaddle/PaddleX/blob/17cc27ac3842e7880ca4aad92358d3ef8555429a/paddlex/repo_apis/PaddleDetection_api/object_det/official_categories.py#L81), [PaperMage](https://github.com/allenai/papermage/blob/9cd4bb48cbedab45d0f7a455711438f1632abebe/README.md?plain=1#L102), [SAM2](https://github.com/facebookresearch/sam2)

- [ ] Fix page rotation, table of contents, format of lists

- [ ] Fix pixel formula in old papers

- [ ] Async retry except KeyboardInterrupt

- [ ] Knuth–Plass algorithm for western languages

- [ ] Support non-PDF/A files

- [ ] Plugins of [Zotero](https://github.com/zotero/zotero) and [Obsidian](https://github.com/obsidianmd/obsidian-releases)

<h2 id="acknowledgement">Acknowledgements</h2>

- Document merging: [PyMuPDF](https://github.com/pymupdf/PyMuPDF)

- Document parsing: [Pdfminer.six](https://github.com/pdfminer/pdfminer.six)

- Document extraction: [MinerU](https://github.com/opendatalab/MinerU)

- Multi-threaded translation: [MathTranslate](https://github.com/SUSYUSTC/MathTranslate)

- Layout parsing: [DocLayout-YOLO](https://github.com/opendatalab/DocLayout-YOLO)

- Document standard: [PDF Explained](https://zxyle.github.io/PDF-Explained/), [PDF Cheat Sheets](https://pdfa.org/resource/pdf-cheat-sheets/)

- Multilingual Font: [Go Noto Universal](https://github.com/satbyy/go-noto-universal)

<h2 id="contrib">Contributors</h2>

<a href="https://github.com/Byaidu/PDFMathTranslate/graphs/contributors">
  <img src="https://opencollective.com/PDFMathTranslate/contributors.svg?width=890&button=false" />
</a>

![Alt](https://repobeats.axiom.co/api/embed/dfa7583da5332a11468d686fbd29b92320a6a869.svg "Repobeats analytics image")

<h2 id="star_hist">Star History</h2>

<a href="https://star-history.com/#Byaidu/PDFMathTranslate&Date">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=Byaidu/PDFMathTranslate&type=Date&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=Byaidu/PDFMathTranslate&type=Date" />
   <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=Byaidu/PDFMathTranslate&type=Date"/>
 </picture>
</a>
