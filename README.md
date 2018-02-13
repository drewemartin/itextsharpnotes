# itextsharpnotes

https://rupertmaier.wordpress.com/2014/08/22/creating-a-pdf-with-an-image-in-itextsharp/

CREATING A PDF WITH AN IMAGE IN ITEXTSHARP
This article shows how to create a PDF file containing an image in iTextSharp. It covers the following features:

Image is stored as an embedded resource in a .NET class library
Content of the PDF is created from HTML
The iTextSharp library provides a way to create a PDF from HTML. But when the PDF should contain images that are not accessible via a public URL some adjustments are necessary. To demonstrate that I decided to embed the image to a .NET class library. This is just an example. The image could also be saved on a filesystem or even be stored as a base64 encoded string in a SQL database.

To be as close as possible to the HTML specification I use the img-tag to reference the image. The HTML markup is created via a XSL transformation.

Content of the xslt:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                              xmlns:msxsl="urn:schemas-microsoft-com:xslt"
                              exclude-result-prefixes="msxsl">
    <xsl:output method="xml" indent="yes"/>
 
    <xsl:template match="/">
      <html>
        <head>
          <title></title>
        </head>
        <body>
          <div style="min-width: 1237px;max-width: 1270px;position: relative;margin: 0 auto;">
            <img src="data:imagestream/EmbeddedImage.Resources.image.jpg" />
          </div>
          <div>It´s working!!</div>
        </body>
      </html>
    </xsl:template>
</xsl:stylesheet>
The src-attribute doesn´t contain a http-Url. Instead I use the prefix data:imagestream to identify the source type of the image. After the following slash the name of the resource in the manifest of the .NET library is listed. That´s the location where differenty source types could be defined.

Now I have to teach iTextSharp a different handling for the img-tag.

This method takes the HTML content and converts it to PDF.

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
public Stream CreateFromHtml(string html)
{
    var stream = new MemoryStream();
 
    using (var doc = new Document(PageSize.A4))
    {
        using (var ms = new MemoryStream())
        {
            using (var writer = PdfWriter.GetInstance(doc, ms))
            {
                writer.CloseStream = false;
                doc.Open();
 
                var tagProcessors = (DefaultTagProcessorFactory)Tags.GetHtmlTagProcessorFactory();
                tagProcessors.RemoveProcessor(HTML.Tag.IMG);
                tagProcessors.AddProcessor(HTML.Tag.IMG, new CustomImageTagProcessor()); 
 
                var cssFiles = new CssFilesImpl();
                cssFiles.Add(XMLWorkerHelper.GetInstance().GetDefaultCSS());
                var cssResolver = new StyleAttrCSSResolver(cssFiles);
 
                var charset = Encoding.UTF8;
                var context = new HtmlPipelineContext(new CssAppliersImpl(new XMLWorkerFontProvider()));
                context.SetAcceptUnknown(true).AutoBookmark(true).SetTagFactory(tagProcessors);
                var htmlPipeline = new HtmlPipeline(context, new PdfWriterPipeline(doc, writer));
                var cssPipeline = new CssResolverPipeline(cssResolver, htmlPipeline);
                var worker = new XMLWorker(cssPipeline, true);
                var xmlParser = new XMLParser(true, worker, charset);
 
                using (var sr = new StringReader(html))
                {
                    xmlParser.Parse(sr);
                    doc.Close();
                    ms.Position = 0;
                    ms.CopyTo(stream);
                    stream.Position = 0;
                }
            }
        }
    }
 
    return stream;
}
Here is where the magic happens. The default handling for img-tags is replaced by a custom tag processor.

1
2
3
var tagProcessors = (DefaultTagProcessorFactory)Tags.GetHtmlTagProcessorFactory();
tagProcessors.RemoveProcessor(HTML.Tag.IMG);
tagProcessors.AddProcessor(HTML.Tag.IMG, new CustomImageTagProcessor());
The custom tag processor only steps in with custom logic, when the src-tag starts with data:imagestream. That ensures that all other “legal” images are resolved by the logic of the default processor.

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
public class CustomImageTagProcessor : iTextSharp.tool.xml.html.Image
{
    public override IList<IElement> End(IWorkerContext ctx, Tag tag, IList<IElement> currentContent)
    {
        var src = String.Empty;
 
        if (!tag.Attributes.TryGetValue(HTML.Attribute.SRC, out src))
            return new List<IElement>(1);
 
        if (String.IsNullOrWhiteSpace(src))
            return new List<IElement>(1);
 
        if (src.StartsWith("data:imagestream/", StringComparison.InvariantCultureIgnoreCase))
        {
            var name = src.Substring(src.IndexOf("/", StringComparison.InvariantCultureIgnoreCase) + 1);
 
            using (var stream = Assembly.GetExecutingAssembly().GetManifestResourceStream(name))
            {
                return CreateElementList(ctx, tag, Image.GetInstance(stream));
            }
        }
 
        return base.End(ctx, tag, currentContent);
    }
 
    protected IList<IElement> CreateElementList(IWorkerContext ctx, Tag tag, Image image)
    {
        var htmlPipelineContext = GetHtmlPipelineContext(ctx);
        var result = new List<IElement>();
        var element = GetCssAppliers().Apply(new Chunk((Image) GetCssAppliers().Apply(image, tag, htmlPipelineContext), 0, 0, true), tag, htmlPipelineContext);
        result.Add(element);
 
        return result;
    }
}
The following sample code creates the HTML, converts it to the PDF and saves the result to a file.

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
class Program
{
    static void Main(string[] args)
    {
        string html;
 
        using (var xsltStream = Assembly.GetExecutingAssembly().GetManifestResourceStream("EmbeddedImage.Resources.Pdf.xslt"))
        {
            var reader = XmlReader.Create(xsltStream);
            var xslt = new XslCompiledTransform();
            xslt.Load(reader);
 
            using (var ms = new MemoryStream())
            {
                xslt.Transform(new XmlDocument(), new XsltArgumentList(), ms);
                ms.Position = 0;
                using (var r = new StreamReader(ms))
                {
                    html = r.ReadToEnd();
                }
            }
        }
 
        var pdfService = new TextSharpPdfService();
        using (var pdf = pdfService.CreateFromHtml(html))
        {
            using (var fs = new FileStream(@"c:\temp\embeddedimage.pdf", FileMode.Create))
            {
                pdf.CopyTo(fs);
            }
        }
    }
}
And here´s the result:

embedded_image_pdf

In my next post I will show how to use the same embedded resource image as header for a HTML mail.

