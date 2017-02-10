---
layout: post
title: 스프링의 Servlet 3.0 지원
subtitle: web.xml을 대체하는 org.springframework.web.WebApplicationInitializer
category: spring
tags: [spring,javaee]
---
## web.xml이 사라진 Servlet 3.0
Java EE 7에서부터 Servlet 3.1이 발표되었고 현재 Servlet 4.0 스펙까지 업그레이드된 마당에 나는 Servlet 3.0 얘기를 하려고 한다. Servlet 3.0에서는 web.xml을 대체하기 위해 [javax.servlet.ServletContainerInitializer][ServletContainerInitializer]{:target="blank"} 인터페이스를 제공한다. web.xml이 사라짐으로 해서 서블릿 추가를 xml에 등록하지 않고
{% highlight java %}
@WebServlet(urlPatterns = "/hello")
public class HelloServlet extends HttpServlet {
....
{% endhighlight %}
와 같이 편하게 서블릿을 추가할 수 있는 대신 web.xml을 대신하기 위해 [javax.servlet.ServletContainerInitializer][ServletContainerInitializer]{:target="blank"} 인터페이스를 구현하기가 쉽지가 않다. 그래서 이번 포스트에서는 스프링에서 제공해주는 [javax.servlet.ServletContainerInitializer][ServletContainerInitializer]{:target="blank"} 인터페이스 구현체에 대해서 얘기해 보려고 한다.

## 스프링의 [javax.servlet.ServletContainerInitializer][ServletContainerInitializer]{:target="blank"} 구현체
일단 스프링이 제공하는 구현체는 [org.springframework.web.WebApplicationInitializer][WebApplicationInitializer]{:target="blank"}이다. Java EE의 [javax.servlet.ServletContainerInitializer][ServletContainerInitializer]{:target="blank"}는 SPI 방식으로 /META-INF/services/javax.servlet.ServletContainerInitializer 파일을 만들어야 하는데 스프링은 우리 대신 해당 인터페이스의 구현체인 [SpringServletContainerInitializer] 클래스를 제공해주며, 이 클래스를 통해 컨테이너에 의해 자동으로 SPI 구현체를 감지한다.([WebApplicationInitializer]{:target="blank"} 구현체를 찾는다.) [SpringServletContainerInitializer] 클래스에 의해 초기화 된 후 만들어진 ServletContext는 [WebApplicationInitializer]{:target="blank"}에게 위임되고 여기에 바로 우리가 원하는 초기화 코드를 작성하면 된다. 이전에 아래와 같이 web.xml을 작성했다면

{% highlight xml %}
 <servlet>
   <servlet-name>dispatcher</servlet-name>
   <servlet-class>
     org.springframework.web.servlet.DispatcherServlet
   </servlet-class>
   <init-param>
     <param-name>contextConfigLocation</param-name>
     <param-value>/WEB-INF/spring/dispatcher-config.xml</param-value>
   </init-param>
   <load-on-startup>1</load-on-startup>
 </servlet>

 <servlet-mapping>
   <servlet-name>dispatcher</servlet-name>
   <url-pattern>/</url-pattern>
 </servlet-mapping>
{% endhighlight %}

이에 해당하는 [WebApplicationInitializer]{:target="blank"} 구현체는 아래와 같다.
{% highlight java %}
public class MyWebAppInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
      XmlWebApplicationContext appContext = new XmlWebApplicationContext();
      appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

      ServletRegistration.Dynamic dispatcher =
        container.addServlet("dispatcher", new DispatcherServlet(appContext));
      dispatcher.setLoadOnStartup(1);
      dispatcher.addMapping("/");
    }
}
{% endhighlight %}

#### SPI에 대해서
From Effective Java 2nd Edition:  

>A service provider framework is a system in which multiple service providers implement a service, and the system makes the implementations available to its clients, decoupling them from the implementations.
>
>There are three essential components of a service provider framework: a service interface, which providers implement; a provider registration API, which the system uses to register implementations, giving clients access to them; and a service access API, which clients use to obtain an instance of the service. The service access API typically allows but does not require the client to specify some criteria for choosing a provider. In the absence of such a specification, the API returns an instance of a default implementation. The service access API is the “flexible static factory” that forms the basis of the service provider framework.
>
>An optional fourth component of a service provider framework is a service provider interface, which providers implement to create instances of their service implementation. In the absence of a service provider interface, implementations are registered by class name and instantiated reflectively (Item 53). In the case of JDBC, Connection plays the part of the service interface, DriverManager.registerDriver is the provider registration API, DriverManager.getConnection is the service access API, and Driver is the service provider interface.
>
>There are numerous variants of the service provider framework pattern. For example, the service access API can return a richer service interface than the one required of the provider, using the Adapter pattern [Gamma95, p. 139]. Here is a simple implementation with a service provider interface and a default provider:

## [AbstractAnnotationConfigDispatcherServletInitializer]{:target="blank"}
[AbstractAnnotationConfigDispatcherServletInitializer]{:target="blank"} 클래스는 스프링 3.2에서 추가된 [WebApplicationInitializer]{:target="blank"} 인터페이스의 구현체이다. 물론 마찬가지로 Servlet 3.0 컨테이너에 배포되면 자동으로 감지되어 ServletContext 설정에 사용된다. 대표적인 메소드는 geRootConfigClasses()와 getServletConfigClasses() 메소드인데 공식 문서를 인용하면 아래와 같다.

| Method | Description |
|---|---|
| protected abstract Class<?>[]	getRootConfigClasses() | Specify @Configuration and/or @Component classes to be provided to the root application context. |
| protected abstract Class<?>[]	getServletConfigClasses() | Specify @Configuration and/or @Component classes to be provided to the dispatcher servlet application context. |

한마디로 말하면 getRootConfigClasses()는 xml의 빈 설정 대신 @Configuration과 @Component 어노테이션으로 설정된 클래스들을 root 어플리케이션 컨텍스트에 등록하고 getServletConfigClasses()는 dispatcher 서블릿 컨텍스트에 등록하는 메소드이다. [WebApplicationInitializer]을 이용한다면 우리가 직접 [AnnotationConfigApplicationContext] 클래스에 JavaConfig 클래스들을 로드하는 코드를 작성해야 한다. 위 예제에서 MyWebAppInitializer 클래스에서 [AnnotationConfigApplicationContext] 클래스 대신 [XmlWebApplicationContext]을 이용하여 xml 빈 설정 파일을 로드하고 있음을 알 수 있다. 아래 예제는 [AbstractAnnotationConfigDispatcherServletInitializer] 클래스를 이용한 컨텍스트 등록 코드이다.

{% highlight java %}
public class ExampleWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
  @Override
  protected String[] getServletMappings() {
    return new String[] { "/" };
  }
  
  @Override
  protected Class<?>[] getRootConfigClasses() {
    return new Class<?>[] { RootConfig.class };
  }

  @Override
  protected Class<?>[] getServletConfigClasses() {
    return new Class<?>[] { WebConfig.class };
  }
}
{% endhighlight %}

위 코드에서 getServletMappings()는 DispatcherServlet의 UrlMapping을 하는 설정으로 [AbstractAnnotationConfigDispatcherServletInitializer]{:target="blank"} 클래스의 상위 클래스인 [AbstractDispatcherServletInitializer]에서 제공한다. 전체 상속구조는 아래 클래스 다이어그램을 참조한다.

![uml]

위 설정을 xml 버전으로 하면 아래와 같다.
{% highlight xml %}
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

<servlet>
  <servlet-name>example</servlet-name>
  <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
  <servlet-name>example</servlet-name>
  <url-pattern>/</url-pattern>
</servlet-mapping>
{% endhighlight %}

본 포스트에서는 Servlet 3.0 스펙에서 web.xml을 대체하는 부분을 스프링에서 어떻게 지원하고 있는지에 대해서 설명하였다. Servlet 3.0은 2009년 12월에 확정되어 포스트를 작성하는 시점보다 햇수로 8년이나 된 오래된 스펙이다. 널리 사용되는 서블릿 컨테이너인 Apache Tomcat은 7버전에서 Servlet 3.0을 지원하고 있기 때문에 이전 버전을 사용할 경우 [WebApplicationInitializer] 구현체를 탐지할 수 없다. (현재는 Servlet 4.0과 JSP 2.3 스펙을 구현한 Tomcat 9.0까지 나와있다.) 글쓰는게 어렵구만~;;

[uml]: http://www.plantuml.com/plantuml/png/bP0z3i8m34Rtd28NQ4x0Ki62nCO9tE26gCMER4CHFtS7gGi8HClizpqzkK3i8A5dIK6BP4gjm047bYuCs0H5EVLeGO-bi9Y_EkzZ3wg-RjG4ejL4R62PQSdKvhJAMi3Y7cKxRjUKBKEVBoWVuv-mkpjN9leYa-7vMzTol6mOTYYlsgTrYl6BMrNDYvm3lUl-UTW3
[ServletContainerInitializer]: http://docs.oracle.com/javaee/6/api/javax/servlet/ServletContainerInitializer.html
[WebApplicationInitializer]: http://docs.spring.io/spring/docs/3.1.0.M2/javadoc-api/org/springframework/web/WebApplicationInitializer.html 
[SpringServletContainerInitializer]: http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/SpringServletContainerInitializer.html
[AbstractAnnotationConfigDispatcherServletInitializer]: http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/support/AbstractAnnotationConfigDispatcherServletInitializer.html
[AbstractDispatcherServletInitializer]: http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/support/AbstractDispatcherServletInitializer.html
[AnnotationConfigApplicationContext]: http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/AnnotationConfigApplicationContext.html
[XmlWebApplicationContext]: http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/context/support/XmlWebApplicationContext.html