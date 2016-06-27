##Basic MVC Configuration
####Dispatcher Servlet
- Implementation of Front Controller Pattern (see Patterns of Enterprise Application Architecture)
- Single point of entry - handle all incomming requests
- Orchestrates request handling by delegating requests to additional components (@Controllers, Views, View Resolvers, handler mappers, ...)
- Can map directly to /, but then exception for handling static resources needs to be configured

####Basic Request handling process
- Client sends a request to a specific URL
- Dispatcher Servlet receives the request
- DS passes request to a specific controller depending on the URL requested
- Controller returns LOGICAL view name and model to DS
- DS consults view resolvers until actual View is determined to render the output
- DS contacts the chosen view (e.g. Thymeleaf, Freemarker, JSP) with model data and it renders the output depending on the model data
- Rendered output is returned to the client as response

####Application Contexts
- Two application contexts are created: Root And Web
- Root Application Context: Web independent; Services, Repositories, ...
- Web Application Context: Contains only web layer specific beans - Controllers, Views, Wier Resolvers,...
- Web Application Context is a child context of Root Application Context
- If a bean is not found in a context, it is searched in parent context (but not vice versa)

**Root webapp context**
- Need to declare org.springframework.web.context.ContextLoaderListener or only DispatcherServlet's context will be created
- If contextConfigLocation is not declared, it defaults to WEB-INF\applicationContext.xml
- ContextLoaderListener - Bootstrap listener to start up and shut down Spring's root WebApplicationContext. Optional.
```
<context-param>
  <param-name>contextConfigLocation</param-name>
  <!--Defaults to WEB-INF\applicationContext.xml if not specified-->
  <param-value>classpath:app-config.xml</param-value>
</context-param>
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
**Child webapp context**  
If contextConfigLocation is not declared, it defaults to WEB-INF\dispatcher-servlet.xml
```
<servlet>
  <servlet-name>dispatcher</servlet-name>
  <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  <init-param>
    <param-name>contextConfigLocation</param-name>
    <!--Defaults to WEB-INF\dispatcher-servlet.xml-->
    <param-value>classpath:mvc-config.xml</param-value>
  </init-param>
</servlet> 
```

####Minimal MVC Configuration
web.xml + WEB-INF/dispatcher-servlet.xml (for xml, annotation based alternative instead)

**web.xml**  
Declare DispatcherServlet as servlet and provide servlet-mapping
```
<servlet>
  <servlet-name>dispatcher</servlet-name>
  <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
</servlet>

<servlet-mapping>
  <servlet-name>dispatcher</servlet-name>
  <url-pattern>/*</url-pattern>
</servlet-mapping>
```
**WEB-INF/dispatcher-servlet.xml**  
In configuration file for Web Application Context declare a controller to serve requests (assuming controller returns directly content using @ResponseBody and no view resolving is required)  
```<bean id="myController" class="com.example.MyController" />```

####Configuration without web.xml 
- From Servlet 3.x web.xml is optional
- Can declare servlets and filters using annotations or implementing interface ServletContainerInitializer
- Needs Servlet 3.x compatible application server
- Servlet container looks for classes implementing ServletContainerInitializer (Spring provides SpringServletContainerInitializer)
- SpringServletContainerInitializer looks for classes implementing WebApplicationInitializer, which specify configuration instead of web.xml
- Spring provides two convenience implementations
    - AbstractContextLoaderInitializer - only Registers ContextLoaderListener
    - AbstractAnnotationConfigDispatcherServletInitializer - Registers ContextLoaderListener and defines Dispatcher Servlet, expects JavaConfig
    
####XML MVC namespace
- In xml config, `<mvc:>` namespace can be used to greatly simplify configuration compared to previous versions
- `<mvc:annotation-driven/>` to enable Default @MVC setup
    - @EnableWebMvc is equivalent is Java Configuration
    - Enables @Controller mappings using @RequestMapping
    - Enables features such as REST, validation, formatting, controller exception handling
    - From Spring 3.0+, not backwards compatible with Spring 2
    - Omit when maintaining Spring 2 backwards compatibility
- `<mvc:default-servlet-handler/>`
    - Needed If mapping Dispatcher Servlet in web.xml to / using servlet-mapping, so exceptions for static resources can be set using `<mvc:resources ... />`
    - `<mvc:resources mapping="/resources/**" location="classpath:/META-INF/web-content" cachePeriod="315569" />`
    - If mapped to /, static resources need special exception not to be handled by dispatcher servlet.
    - Configured in spring boot by default
-  Mapping urls to views without controllers
    - ```<mvc:view-controller path="/foo" view-name = "foo"/>```
    - ```<mvc:redirect-view-controller path="/old" redirect-url="/new" status-code="308" keep-query-params="true"/>```

####Java Config - @EnableWebMvc
- Equivalent of `<mvc:annotation-driven/>`
- On class level on @Configuration class
- Customisation - @Configuration class implements WebMvcConfigurer
    - Or for convenience extends WebMvcConfigurerAdapter, which implements given interface with empty methods
    - Methods - addInterceptors, addResourceHandlers (similiar to `<mvc:resource>`), configureDefaultServletHandling, addControllers, configureContentNegotiation
- Not specified by Spring Boot, but similar config is; Should not be specified when using Boot; If specified boot skips mvc autoconfiguration and needs to be done manually
