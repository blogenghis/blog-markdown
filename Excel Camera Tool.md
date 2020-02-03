# Excel的数据快照特性

## 问题

问题来源于项目中一个蹩脚的需求。客户原先有一份Excel模版，其中跟踪了原料的库存量和消耗速度，通过复杂的公式实现报表实时刷新。客户希望将报表呈现在网页上，同时可供下载保存。并且已经验证了通过Excel的快照功能可以实现将一个范围内的单元格映射为图片，该图片能够随数据变化而更新，因此希望我们动态修改数据后直接在网页上获取该图片。

> 相关的概念和操作可以参考[这个链接](https://excelchamps.com/blog/camera-tool/)。

## 分析

用户的操作均在桌面UI下完成，为了在后端无UI的情况下实现相同功能，我们选择Apache POI库操作Excel。

一般情况下，Excel所使用的Open XML格式将图片文件单独存储，但是对于这种快照图片，不清楚是否也有对应图片文件。

另一方面，数据变动后需要重新渲染图片，这个渲染过程是否能够在无UI界面的情况下完成还未知。

这里将问题分为两个关键点：
1. 从Excel中获取该快照图片；
2. 修改数据后刷新该图片。

## 尝试

首先，解压Excel，在xl/media/目录下找到了emf图片文件。其内容与快照图片一致，因此可以确认Excel快照也会动态生成对应图片文件。如此，获取图片的方式即与普通图片相同。

POI库有直接获取图片的API——[Workbook#getAllPictures](https://poi.apache.org/apidocs/dev/org/apache/poi/ss/usermodel/Workbook.html#getAllPictures--)。但是这个API直接获取图片文件的内容，如果存在多张图片就无法确认是不是所需要的图片。

换一种方式，通常图片和图形都是使用Drawing对象存储，因此可以通过获取Drawing对象，并对比其Name字段与Excel名称框而准确找到该图片。以下是相关代码：

```java
XSSFSheet imageSheet = workbook.getSheet("Sheet1");  
for (POIXMLDocumentPart documentPart : imageSheet.getRelations()) {  
    if (documentPart instanceof XSSFDrawing) {  
        XSSFDrawing drawing = (XSSFDrawing) documentPart;  
        for (XSSFShape shape : drawing.getShapes()) {  
            if (shape instanceof XSSFPicture && shape.getShapeName().equals(pictureName)) {  
                XSSFPicture picture = (XSSFPicture) shape;
                XSSFPictureData pictureData = picture.getPictureData();
                ...
            }
        }
    }
}
```

通过这种方式定位到的图片对象，其结构与Excel中的drawingX.xml一致。

![drawingX.xml内容](https://raw.githubusercontent.com/progenghis/p/res/excel-camera-tool/drawing.png)

甚至可以通过如下代码，找到该快照引用的单元格范围。并且可以看出代码结构也与xml结构一致。

```java
CTPicture ctPicture = picture.getCTPicture();  
CTNonVisualPictureProperties cNvPicPr = ctPicture.getNvPicPr().getCNvPicPr();  
CTOfficeArtExtension extLst = cNvPicPr.getExtLst().getExtArray(0);  
System.out.println(extLst.toString());
```

至此，第一个问题已经解决。紧接着尝试动态渲染：

1. 直接调用[FormulaEvaluator#evaluateAll](http://poi.apache.org/apidocs/dev/org/apache/poi/ss/usermodel/FormulaEvaluator.html#evaluateAll--)，计算所有公式；
2. 通过[CTOfficeArtExtension#set]()重设快照取值范围；
3. 重新保存文件；
4. ……

这些方法无一例外都失败了。

从快照对应的drawing.xml中，我们可以看出，快照是一个名叫CameraTool的扩展模块。通过查找[Microsoft官方文档](https://msdn.microsoft.com/en-us/library/documentformat.openxml.office2010.drawing.cameratool.aspx?cs-save-lang=1&cs-lang=csharp#code-snippet-1)可以确定这是一个较新的功能，而POI的网站和API中均未查找到对于该特性的支持。

## 方案

面对该问题的实现难度，客户表示资金充裕，是否可以使用商业Excel库实现该功能，于是尝试了Aspose.Cells。通过简单了解，尝试性的写出如下代码：

```java
com.aspose.cells.Workbook workbook = new com.aspose.cells.Workbook(file.getAbsolutePath());workbook.calculateFormula();  
Worksheet worksheet = workbook.getWorksheets().get("Sheet1");  
ShapeCollection shapes = worksheet.getShapes();  
for (int i = 0; i < shapes.getCount(); i++) {  
    Shape shape = shapes.get(i);  
    if (pictureName.equals(shape.getName())) {   
        shape.updateSelectedValue();
        ...
    }
}
```

令人难以置信的是，仅仅在对应的shape上调用updateSelectedValue()，快照被刷新了。甚至直接调用shape的toImage()方法就可以将图片输出到文件或流。

相比POI的繁复，Aspose.Cells在Excel操作上简洁而强大。不过一个单用户的商用License也需要$4000+，也只有一些出手阔绰的客户才会考虑吧。

## 遗留

最终尝试了使用Aspose.Cells导出PNG图片或EMF图片，其效果都还有缺陷，但由于用户变更了需求价值和范围，这个SPIKE就终止掉了。两个问题遗留下来：
1. 导出PNG时，一些图标会有黑背景，这应该是PNG的alpha通道没有处理好导致的。并且文字的边缘都没有做平滑处理。不知道Aspose.Cells是否能够通过设置解决这些绘图问题。
2. 导出EMF时，图片效果与快照效果一致，但是中文都无法显示。看起来多半是字体问题，限于手头没有Windows机器验证，只能作罢。

## 结论

在看到POI XSLF包下的[PPTX2PNG](https://poi.apache.org/apidocs/dev/org/apache/poi/xslf/util/PPTX2PNG.htmll)类时，觉得自己也一定可以实现类似的快照功能，但是所需要的时间成本是用户无法接受的。而这些商业性的工具，恰恰能够满足客户。这也很好的解释了为什么它们那么贵还能生存。

> Written with [StackEdit](https://stackedit.io/) at Dec 1, 2018.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTYzMTMxNDgwMF19
-->