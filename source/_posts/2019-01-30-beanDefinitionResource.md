---
title: 读书笔记：beanDefinition的resource定位
date: 2019-01-30 09:13:05
tags: 
- framework
- spring ioc
comments: true
categories: 
- Java架构
- 源码
---

spring ioc 的原理将beanDefinition 的定位，载入，解析分为三部分。本篇选取FileSystemXmlApplicationContext 的源码实现来分析下如何实现beanDefinition的定位。
<!--more-->
spring ioc 的设计分为两部分。一部分是基于interface （代表接口：BeanFactory,ApplicationContext），规范了IOC容器应该具备的功能。另一部分是基于Abstract (代表抽象类：AbstractBeanFactory,AbstractApplicationContext)。
抽象类会实现interface 的部分方法,然后还会调用interface的部分方法，这也是一种模板模式。FileSystemXmlApplicationContext 是针对使用场景，对抽象类的具体扩展。源码中除了构造器，就是两个特有的方法，如下：


{% codeblock FileSystemXmlApplicationContext  lang:java %}
    public FileSystemXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {
        
		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
		    //调用了父类AbstractApplicationContext 的refresh()方法，目的是刷新容器中的bean重新定位BeanDefinition
			refresh();
		}
	}
	@Override
    	protected Resource getResourceByPath(String path) {
    		if (path.startsWith("/")) {
    			path = path.substring(1);
    		}
    		return new FileSystemResource(path);
    	}

{% endcodeblock %}
{% codeblock refresh()沿着方法栈往下走  lang:java %}
    public void refresh() throws BeansException, IllegalStateException {
        ...
        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        ...
    }
    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        refreshBeanFactory();
        ...
    }
{% endcodeblock %}

refreshBeanFactory 是 AbtractApplicationContext 的抽象方法，要知道其实现类是什么必须借助调用refresh()方法的<font color="#FF0000">FileSystemXmlApplicationContext类的继承树来确定</font>。
{% asset_img 1.png extends tree %}

{% codeblock AbstractRefreshableApplicationContext.refreshBeanFactory()  lang:java %}
    @Override
	protected final void refreshBeanFactory() throws BeansException {
	    ...
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
		...
	}
	@Override
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        // Create a new XmlBeanDefinitionReader for the given BeanFactory.
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

        // Configure the bean definition reader with this context's
        // resource loading environment.
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        //注意这个this 指的是继承树的DefaultResourceLoader
        beanDefinitionReader.setResourceLoader(this);
        ...
        loadBeanDefinitions(beanDefinitionReader);
    }
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
	    //返回Resource[] 的是本类方法，返回的是null
        Resource[] configResources = getConfigResources();
        if (configResources != null) {
            reader.loadBeanDefinitions(configResources);
        }
        //
        String[] configLocations = getConfigLocations();
        if (configLocations != null) {
            reader.loadBeanDefinitions(configLocations);
        }
    }
    @Nullable
    protected Resource[] getConfigResources() {
        return null;
    }
    //继续跟踪loadBeanDefinitions 
	public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
	    //这个是DefaultResourceLoader
        ResourceLoader resourceLoader = getResourceLoader();
        if (resourceLoader == null) {
            throw new BeanDefinitionStoreException(
                    "Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
        }
        if (resourceLoader instanceof ResourcePatternResolver) {
            ...
        }else {
            // Can only load single resources by absolute URL.
            Resource resource = resourceLoader.getResource(location);
            int loadCount = loadBeanDefinitions(resource);
           ...
        }
    }
    //看看DefaultListableBeanFactory.getResource()  
    @Override
    public Resource getResource(String location) {
        ...
        if (location.startsWith("/")) {
            //这个使用的是模板模式。DefaultListableBeanFactory中的getResourceByPath方法根据继承树，已经被FileSystemXmlApplicationContext重写
            return getResourceByPath(location);
        }
        else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
            return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
        }
        else {
            try {
                // Try to parse the location as a URL...
                URL url = new URL(location);
                return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
            }
            catch (MalformedURLException ex) {
                // No URL -> resolve as resource path.
                return getResourceByPath(location);
            }
        }
    }
    //最后回到了FileSystemXmlApplicationContext.getResourceByPath()
    @Override
    protected Resource getResourceByPath(String path) {
        if (path.startsWith("/")) {
            path = path.substring(1);
        }
        return new FileSystemResource(path);
    }
{% endcodeblock %}

是的。我们分析源码就知道实际场景采用的类，是基于Abstract 类 针对实际场景的扩展。
END
