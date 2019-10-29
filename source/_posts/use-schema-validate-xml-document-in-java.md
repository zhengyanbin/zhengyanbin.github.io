---
title: 在Java中使用xsd校验xml
path: use-schema-validate-xml-document-in-java
date: 2017-04-17 17:00:33
updated: 2017-04-17 17:00:33
categories:
- Java
tags:
- java
- xsd
---

最近项目需要使用xsd对xml进行预校验，于是封装了一个工具类，来完成校验工作。<!--more-->
完整代码如下：
    
``` java
import java.io.File;
import java.io.IOException;
import java.io.StringReader;
import java.util.ArrayList;
import java.util.List;
import java.util.Locale;

import javax.xml.XMLConstants;
import javax.xml.transform.stream.StreamSource;
import javax.xml.validation.Schema;
import javax.xml.validation.SchemaFactory;
import javax.xml.validation.Validator;

import org.xml.sax.ErrorHandler;
import org.xml.sax.SAXException;
import org.xml.sax.SAXParseException;

public class MultiSchemaValidator {
	
//	private static final Logger logger = LoggerFactory.getLogger(MultiSchemaValidator.class);

	static{
		System.setProperty("jdk.xml.maxOccurLimit", "9999");    //默认的maxOccur为5000，而我们项目中要求9999
		Locale.setDefault(Locale.CHINA);    //如果项目不考虑国际化的话
	}
		
	public static List<SAXParseException> validateXMLSchema(String xsdPath, String xml){
		final List<SAXParseException> errors = new ArrayList<>();
        try {
            SchemaFactory factory = 
                    SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
            String path = Thread.currentThread().getContextClassLoader().getResource("").getPath();
            Schema schema = factory.newSchema(new File(path + xsdPath));
            Validator validator = schema.newValidator();
            validator.setErrorHandler(new ErrorHandler() {
				@Override
				public void warning(SAXParseException exception) throws SAXException {
//					logger.debug("warning ex", exception);
				}
				@Override
				public void fatalError(SAXParseException exception) throws SAXException {
//					logger.debug("fatalError ex", exception);
				}
				
				@Override
				public void error(SAXParseException exception) throws SAXException {
//					logger.debug("error ex", exception);
					errors.add(exception);
				}
			});
            validator.validate(new StreamSource(new StringReader(xml)));
        } catch (IOException | SAXException e) {
            System.out.println("Exception: "+e.getMessage());
        }
        return errors;
    }
    //测试代码
	public static void main(String[] args) throws Exception {
		String schemaURI = "xsd/Manifest.xsd";
        String xml = "";

        List<SAXParseException> errors = validateXMLSchema(schemaURI, xml);
		
		for(SAXParseException ex : errors){
			System.out.println(ex.getLineNumber() + "行," + ex.getColumnNumber() + "列," + ex.getMessage());
		} 
    }
}
```



   > 该代码应该可以完成一般需求。不过需要注意以下问题：
   > 1. xsd中使用`<xs:import>` `<xs:include>` 引入其他xsd文件时，不要将xsd打包到jar中，这种方式不支持jar!的方式访问import文件。
   > 2. jdk有xml-apis及其实现，但是尝试覆盖其`XMLSchemaMessages.properties`以便自定义提示语句时出现问题，便引用了 `xml-apis` 及 `xercesImpl`,覆盖了`org.apache.xerces.impl.msg`包下的properties文件。
   > 3. 上述代码可以完成多schema文件的校验，需保证xsd都在相同路径。若不在同一位置，可参考链接中博客的方式，实现[SchemaFactory解析shcema][1]的处理操作。




[1]: http://blog.csdn.net/hld_hepeng/article/details/6318663