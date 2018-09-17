---
title: C#之XML反序列化的一些坑
date: 2018-09-17 12:59:14
categories: .net
tags: xml,serialize
keyworks: xml,serialize
permalink:
---

##### 背景

我们在.net平台反序列化一份xml文档的时候经常性的会遇到一些很奇怪的问题，要么在反序列化过程中抛出各种异常，要么没有发生异常，可结果却不是我们想要的。抛异常的情况倒是比较容易解决，根据异常信息自己找MSDN文档或者Google搜索，一般都能找到解决办法。麻烦的是没有异常结果却不理想的情况，这种情况往往需要一步步调试跟踪代码，但是微软的底层已经封装，调试起来很困难。

下面是我在反序列化的时候遇到过的一些坑。先上一段需要反序列化的代码。

##### 目标XML

    <?xml version='1.0' encoding='UTF-8'?>
    <soapenv:Envelope
        xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
        <soapenv:Body>
            <xsd:DestinationListResponse
                xmlns:xsd="http://response.example.com/xsd">
                <xsd:success>true</xsd:success>
                <xsd:country xsd1:id="1933312" xsd1:name="Afghanistan" xsd1:nameEn="Afghanistan" xsd:isoCode="AF"
                    xmlns:xsd1="http://types.example.com/xsd">
                    <xsd:city xsd1:id="1934162" xsd1:name="Kabul" xsd1:nameEn="Kabul" xsd:isoCode="KBL" xsd:latitude="34.53333282470703" xsd:longitude="69.16666412353516"/>
                </xsd:country>
                <xsd:country xsd1:id="1933313" xsd1:name="Albania" xsd1:nameEn="Albania" xsd:isoCode="AL"
                    xmlns:xsd1="http://types.example.com/xsd">
                    <xsd:city xsd1:id="31959649" xsd1:name="Durres" xsd1:nameEn="Durres" xsd:isoCode="TIA" xsd:latitude="41.31666564941406" xsd:longitude="19.450000762939453"/>
                    <xsd:city xsd1:id="50001732" xsd1:name="Pogradec" xsd1:nameEn="Pogradec" xsd:isoCode="" xsd:latitude="40.900001525878906" xsd:longitude="20.649999618530273"/>
                </xsd:country>
            </xsd:DestinationListResponse>
        </soapenv:Body>
    </soapenv:Envelope>

这一段XML，相信大家都能看懂。这里涉及到几个XML的概念如下：声明、名称空间、前缀、元素、属性。对这几个概念不了解的自行Google。了解这几个概念之后，构建C#实体类就很简单了，复杂元素构建成类，简单元素和属性作为类的字段。我创建了对应的实体类之后开始反序列化就遇到了一些问题。

##### 一、获取SOAP中Body里的内容报错或者为空

我这里需要获取`DestinationListResponse`元素反序列化成我建的对应的C#实体DestinationListResponse类的对象。代码如下：
     XmlDocument doc = new XmlDocument();
            doc.LoadXml(xml);
            var node = doc.SelectSingleNode("//Body");
            var innerXml = node.InnerXml;

这样没有报错，但是获取到`node`的值=null。然后我把`var node = doc.SelectSingleNode("//Body");`这句代码改成`var node = doc.SelectSingleNode("//soapenv:Body");`之后再调试。这句代码直接就抛出异常了，提示类似缺少名称空间之类的信息。然后我发现`SelectSingleNode`正好有个参数类型是管理名称空间的，我就在这句代码之前把所有名称空间全部加上，最后完整的代码就长这样：

    XmlDocument doc = new XmlDocument();
            doc.LoadXml(xml);
     XmlNamespaceManager nsmgr = new XmlNamespaceManager(doc.NameTable);
            nsmgr.AddNamespace("soapenv", "http://schemas.xmlsoap.org/soap/envelope/");
            nsmgr.AddNamespace("xsd", "http://response.example.com/xsd");
            nsmgr.AddNamespace("xsd1", "http://types.example.com/xsd");
            var node = doc.SelectSingleNode("//soapenv:Body");
            var innerXml = node.InnerXml;
这下，终于`node`终于有值了。然后我藏尸着不把名称空间全部加上，少一个或者两个，`SelectSingleNode`方法都会抛异常。我又尝试着把这句代码改成`var node = doc.SelectSingleNode("//Body");`,`node`的值还是null。

以上得出：

1. XML有名称空间时必须把所有名称空间通过`XmlNamespaceManager`管理起来传入方法`SelectSingleNode`中。
2. xml节点有前缀时，获取节点必须带前缀，如`doc.SelectSingleNode("//soapenv:Body")`而不能是`doc.SelectSingleNode("//Body")`。

##### 二、反序列化XML之后,集合元素对应的字段的值为空

如目标XML中city元素是多个一组的，我称之为集合元素。对应的实体类型的字段也应该是集合类型的，我首先想到的就是使用数组，给字段标记`XmlArray`特性，如下：

    [XmlArray]
    [XmlArrayItem(ElementName ="city")]
    public CityClass[] CityArr { get; set; }

但是，这样定义的字段在反序列化XML之后`CityArr`的值是`null`的，必须改成下面这样的才能正确的得到`city`元素的内容

    [XmlElement(ElementName = "city")]
    public CityClass[] CityArr { get; set; }

##### 三、反序列化XML之后,含前缀的元素的属性对应的字段的值为空

出现这种情况，大概率有2个原因。

1. 实体类没有指定名称空间
2. 实体类字段没有指定强制使用名称空间前缀

如目标XML所有元素都有前缀，而且前缀还不一样，有`xsd`和`xsd1`。前缀和名称空间是一一对应的，目标XML中`xsd`前缀对应`http://response.example.com/xsd`名称空间，`xsd1`前缀对应`http://types.example.com/xsd`名称空间。如`city`元素的前缀是`xsd`。那么它对应的实体就要像下面这样加上名称空间。

    [Serializable]
    [XmlType(Namespace = "http://response.example.com/xsd")]
    public class CityClass
    {
        [XmlAttribute(Form = XmlSchemaForm.Qualified)]
        public string isoCode { get; set; }
        ......
    }
这段代码中`isoCode`的`XmlAttribute`特性加了一句代码`Form = XmlSchemaForm.Qualified` 它的作用就是强制使用名称空间前缀的意思。如果xml文档的元素或者元素属性有前缀，那么加上这句就可以正确的反序列化了。不过，如果是元素的化，好像不加这句也没有什么影响。

### 以上所有提及的序列化反序列化均是指使用微软.net平台System.Xml.Serialization命名空间中的类实现的操作。