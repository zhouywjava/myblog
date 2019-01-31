---
title: 读书笔记：beanDefinition的载入
date: 2019-01-31 15:08:50
tags: 
- framework
- spring ioc
comments: true
categories: 
- Java架构
- 源码
---

上次我们分析了beanDefinition的定位。今天我们来分析beanDefinition 的载入
<!--more-->
载入的过程其实就是xml 的解析。从AbstractBeanDefinitionReader.loadBeanDefinitions 中的loadBeanDefinitions(Resource resource) 出发
{% codeblock AbstractBeanDefinitionReader.loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) lang:java %}
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    ResourceLoader resourceLoader = getResourceLoader();
    ...
    // Can only load single resources by absolute URL.
    //这步是定位
    Resource resource = resourceLoader.getResource(location);
    //这步就是载入。这个也是模板模式的一种表现形式，除了组合模式下的模板模式，这边采用的是Abstract implements interface 后直接调用interface 中的方法
    //那么具体实现，就是看loadBeanDefinitions 的引用类型了
    int loadCount = loadBeanDefinitions(resource);
    ...
}
{% endcodeblock %}
XmlBeanDefinitionReader.loadBeanDefinitions()
XmlBeanDefinitionReader.doLoadBeanDefinitions()
XmlBeanDefinitionReader.registerBeanDefinitions(Document doc, Resource resource)
DefaultBeanDefinitionDocumentReader.doRegisterBeanDefinitions(Element root)
{% codeblock DefaultBeanDefinitionDocumentReader.doRegisterBeanDefinitions(Element root) lang:java %}
    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
        this.readerContext = readerContext;
        logger.debug("Loading bean definitions");
        Element root = doc.getDocumentElement();
        doRegisterBeanDefinitions(root);
    }
    protected void doRegisterBeanDefinitions(Element root) {
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = createDelegate(getReaderContext(), root, parent);
        ...
        preProcessXml(root);
        parseBeanDefinitions(root, this.delegate);
        postProcessXml(root);
        
        this.delegate = parent;
    }
    protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        ...
            parseDefaultElement(ele, delegate);
        ...
    }
    private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
      ...
        else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            processBeanDefinition(ele, delegate);
        }
     ...
    }
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        //xml 解析后是保存到BeanDefinitionHolder 类中 真正执行解析的是BeanDefinitionParserDelegate 类
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        ...
    }
{% endcodeblock %}
{% codeblock BeanDefinitionParserDelegate.parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) lang:java %}
    ...
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
    ...
    public AbstractBeanDefinition parseBeanDefinitionElement(
    			Element ele, String beanName, @Nullable BeanDefinition containingBean) {
        ...
        try {
            AbstractBeanDefinition bd = createBeanDefinition(className, parent);

            parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
            bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

            parseMetaElements(ele, bd);
            parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
            parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

            parseConstructorArgElements(ele, bd);
            parsePropertyElements(ele, bd);
            parseQualifierElements(ele, bd);

            bd.setResource(this.readerContext.getResource());
            bd.setSource(extractSource(ele));

            return bd;
        }
        ...
    }
    public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
        NodeList nl = beanEle.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
                parsePropertyElement((Element) node, bd);
            }
        }
    }
    public void parsePropertyElement(Element ele, BeanDefinition bd) {
        String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
        ...
            Object val = parsePropertyValue(ele, bd, propertyName);
            PropertyValue pv = new PropertyValue(propertyName, val);
            parseMetaElements(ele, pv);
            pv.setSource(extractSource(ele));
            bd.getPropertyValues().addPropertyValue(pv);
        ...
    }
    public Object parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName) {
        ...
        boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
        boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
        if ((hasRefAttribute && hasValueAttribute) ||
                ((hasRefAttribute || hasValueAttribute) && subElement != null)) {
            error(elementName +
                    " is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
        }

        if (hasRefAttribute) {
            String refName = ele.getAttribute(REF_ATTRIBUTE);
            if (!StringUtils.hasText(refName)) {
                error(elementName + " contains empty 'ref' attribute", ele);
            }
            RuntimeBeanReference ref = new RuntimeBeanReference(refName);
            ref.setSource(extractSource(ele));
            return ref;
        }
        else if (hasValueAttribute) {
            TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
            valueHolder.setSource(extractSource(ele));
            return valueHolder;
        }
        else if (subElement != null) {
            return parsePropertySubElement(subElement, bd);
        }
        ...
    }
    public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd, @Nullable String defaultValueType) {
        ...
        else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
            return parseIdRefElement(ele);
        }
        else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
            return parseValueElement(ele, defaultValueType);
        }
        else if (nodeNameEquals(ele, NULL_ELEMENT)) {
            // It's a distinguished null value. Let's wrap it in a TypedStringValue
            // object in order to preserve the source location.
            TypedStringValue nullHolder = new TypedStringValue(null);
            nullHolder.setSource(extractSource(ele));
            return nullHolder;
        }
        else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
            return parseArrayElement(ele, bd);
        }
        else if (nodeNameEquals(ele, LIST_ELEMENT)) {
            return parseListElement(ele, bd);
        }
        else if (nodeNameEquals(ele, SET_ELEMENT)) {
            return parseSetElement(ele, bd);
        }
        else if (nodeNameEquals(ele, MAP_ELEMENT)) {
            return parseMapElement(ele, bd);
        }
        else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
            return parsePropsElement(ele);
        }
        ...
    }
{% endcodeblock %}
以上就是 xml 的解析过程，beanDefinition 的载入过程。 BeanDefinitionParserDelegate 具体完成了解析的动作，解析的结果是AbstractBeanDefinition 类，外部封装一层return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);后返回。
END