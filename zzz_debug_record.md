2024.12.20
调试记录
1、在window环境下，直接双击安装setip.bat,直接安装了python的环境。安装完可以直接运行，会自行下载yolo布局模型的权重。随后自己运行，那就 .\pdf2zh_dist\python.exe pdf2zh\pdf2zh.py -i
也就是在下载的python环境里运行pdf2zh.py主程序。
2、打开的可视化界面，提供了完善的功能。这里，为了免费的翻译，选择bing，反应还可以。如果用deepLX，会比较慢，而且需要下载代理挂在后台。api会延时，一段时间后才能行。
3、翻译后的pdf，个别排版会有些重影，放的位置重合了，个别语句会被检测为不用翻译的部分。总体功能可用。
4、在后面，突然出现了，文档翻译还可以，但是全部都有线条，好像是文字下划线，将所有文字下面连接到了一起，排版变得奇怪，第一次用不会这样。难说原因。
各个文件的功能
1、pdf2zh.py：主程序，解析参数

2、high_level.py：主要pipeline，translate函数是整体流程。
    translate_stream是处理文档函数
        1、文档交叉引用表中，字体的更新。交叉引用表是 PDF 文件中用于管理和定位对象的关键数据结构。它不仅记录了对象的位置，还管理了对象的状态。
        2、translate_patch  ：
            1、TranslateConverter 类 用于翻译。
            2、PDFPageInterpreterEx 类实例化：interpreter 
            3、初始化后，先确定pages数量。
                随后解析pdf：parser = PDFParser(inf)   doc = PDFDocument(parser)
                逐页进行解析：
                    先取当前页的图像，给model进行推理，得出页面布局,布局结果里有类别:page_layout = model.predict(image, imgsz=int(pix.height / 32) * 32)[0]
                    vcls = ["abandon", "figure", "table", "isolate_formula", "formula_caption"]
                    根据推理出来的类别，对box分别进行操作：page_layout.names[int(d.cls)] in vcls:
                处理后，新建一个 xref 存放新指令流
                    page.page_xref = doc_zh.get_new_xref()  # hack 插入页面的新 xref
                    doc_zh.update_object(page.page_xref, "<<>>")
                    doc_zh.update_stream(page.page_xref, b"")
                    doc_zh[page.pageno].set_contents(page.page_xref)
                解析器来进行解析。interpreter.process_page(page)

3、pdfinterp.py：pdf解析器。继承自pdfminer.pdfinterp 的函数，重构成PDFPageInterpreterEx
    process_page是主要函数，页面渲染，设置字体，可能用上页面旋转角度。
        render_contents 方法渲染内容流。    
        init_resources 方法准备字体和 XObject 资源，并设置颜色空间。
        execute 解析执行pdf中的内容流。根据关键字指令，与库中对照返回操作符。
    
    后面的函数好像没用。
    do_S 方法处理路径描边，过滤非公式线条，do_f, do_F, do_f_a, do_B, do_B_a 方法处理路径填充和描边。
    do_SCN, do_scn, do_SC, do_sc 方法设置描边和非描边操作的颜色。
    do_Do 方法重载 XObject的obj_patch
    
    execute 方法解析并执行 PDF 内容流中的操作符。

4、doclayout.py：pdf布局解析，主要用在pdf2zh.py中。
5、convert.py：TranslateConverter 翻译后布局的解析。之前还先重构一个PDFConverterEx，来自原始库的重构。
    receive_layout主要处理函数，解析布局，翻译处理。
    主要过程：原文档解析，翻译，新文档排版。
    主要区分：文字、公式、线条、图表。用了几个栈，分别存储文字，公式，
    判断是否为公式，并使用vflag函数匹配公式字体。
    
    以及三个函数：

【lstk】全局线条，可能是这个出的问题。

6、translate.py：翻译，调用api


backend.py：是flask的后端。
    cache.py：ui界面的后端

s_mono, s_dual = translate_stream(s_raw, **locals())
locals获取当前作用域内所有局部变量。

