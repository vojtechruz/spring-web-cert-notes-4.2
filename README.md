#Basic MVC Configuration
###Dispatcher Servlet
- Implementation of Front Controller Pattern (see Patterns of Enterprise Application Architecture)
- Single point of entry - handle all incomming requests
- Orchestrates request handling by delegating requests to additional components (@Controllers, Views, View Resolvers, handler mappers, ...)
- Can map directly to /, but then exception for handling static resources needs to be configured

###Basic Request handling process
- Client sends a request to a specific URL
- Dispatcher Servlet receives the request
- DS passes request to a specific controller depending on the URL requested
- Controller returns LOGICAL view name and model to DS
- DS consults view resolvers until actual View is determined to render the output
- DS contacts the chosen view (e.g. Thymeleaf, Freemarker, JSP) with model data and it renders the output depending on the model data
- Rendered output is returned to the client as response

###Application Contexts
- Two application contexts are created: Root And Web
- Root Application Context: Web independent; Services, Repositories, ...
- Web Application Context: Contains only web layer specific beans - Controllers, Views, Wier Resolvers,...
- Web Application Context is a child context of Root Application Context
- If a bean is not found in a context, it is searched in parent context (but not vice versa)

**Root webapp context**
- Need to declare org.springframework.web.context.ContextLoaderListener or only DispatcherServlet's context will be created
- If contextConfigLocation is not declared, it defaults to WEB-INF\applicationContext.xml
- ContextLoaderListener - Bootstrap listener to start up and shut down Spring's root WebApplicationContext. Optional.
```xml
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
```xml
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

###Minimal MVC Configuration
web.xml + WEB-INF/dispatcher-servlet.xml (for xml, annotation based alternative instead)

**web.xml**  
Declare DispatcherServlet as servlet and provide servlet-mapping
```xml
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
```xml
<bean id="myController" class="com.example.MyController" />
```

###Configuration without web.xml 
- From Servlet 3.x web.xml is optional
- Can declare servlets and filters using annotations or implementing interface ServletContainerInitializer
- Needs Servlet 3.x compatible application server
- Servlet container looks for classes implementing ServletContainerInitializer (Spring provides SpringServletContainerInitializer)
- SpringServletContainerInitializer looks for classes implementing WebApplicationInitializer, which specify configuration instead of web.xml
- Spring provides two convenience implementations
    - AbstractContextLoaderInitializer - only Registers ContextLoaderListener
    - AbstractAnnotationConfigDispatcherServletInitializer - Registers ContextLoaderListener and defines Dispatcher Servlet, expects JavaConfig
    
###XML MVC namespace
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

###Java Config - @EnableWebMvc
- Equivalent of `<mvc:annotation-driven/>`
- On class level on @Configuration class
- Customisation - @Configuration class implements WebMvcConfigurer
    - Or for convenience extends WebMvcConfigurerAdapter, which implements given interface with empty methods
    - Methods - addInterceptors, addResourceHandlers (similiar to `<mvc:resource>`), configureDefaultServletHandling, addControllers, configureContentNegotiation
- Not specified by Spring Boot, but similar config is; Should not be specified when using Boot; If specified boot skips mvc autoconfiguration and needs to be done manually
  
----------------
  
#MVC Components
###Model
- Is always created and passed to the view
- If a mapped controller method has Model as a method parameter, then a model instance is automatically injected by Spring to that method
- Any attributes set on injected model are preserved and passed to the View
```java
public String personDetail(Model model) {
  ...
  model.addAttribute("name", "Joe");
  ...
}
```
- Can be also added without attribute name `model.addAttribute(person);` Attribute name is then set depending on the added type name. E.g. Person type results in "person" name.

###View
- Template for rendering output to client based on Model data
- Display: JSP, Thymeleaf, Freemarker, Velocity,...
- Content: JSON, XML, PDF,...
- Implements View interface - defines which content type and how to render
```java
public interface View {
  String getContentType();
  void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception;
}
```
- JstlView - forwards to JSP page, model attributes are passed to the JSP as `${attributeName}`
- Content generating view - unlike views which generate html and are declared and managed by Spring, these need to be declared manually
    - Create view class which extends specific parent - e.g. AstractPdfView and implement details of generating content manually
    - protected void buildPdfDocument(Map<String, Object> model, Document document, PdfWriter pdfWriter, HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse) throws Exception

###Controller
- POJO annotated @Controller
- Has methods which corresponds to specific urls, incl. variants for different HTTP methods (GET, POST), parameters provided etc.
- Methods are mapped to URLs usually using annotations @RequestMapping
- Controller fills Model with data, which is later provided to the View
- Controller can directly return data using @ResponseBody annotation
- Or can return a Logical view name as String. That is later resolved to a specific view using ViewResolvers
- If returns null or return type is void, the view name will be determined from URL requested
    - remove leading slash and remove extension
    - using default RequestToViewNameTranslator
- When specified as method parameters, certain objects can be automatically injected by spring to be used inside the controller methods
    - Model, HttpServletRequest, HttpServletResponse, Locale, Principal, HttpSession, HttpEntity<?>, TimeZone,...
- Can be well unit tested (is POJO) without container

----------------

#Mapping Controllers
###Handler Resolving process
Controller is a specific type of Handler
- Client sends a request to a specific url
- Dispatcher servlet consults handler mappings, specific handler is returned based on the request - `getHandler(request)`
- Handler is invoked through HandlerAdapter - `handle(request,response,handler)`


###Handler Mapping
- RequestMappingHandlerMapping - enabled by default - Takes into account @RequestMapping on class and method level of @Controllers
- ControllerClassNameHandlerMapping - @RequestMapping on method level, but not on class level, class level is mapped from controller name
    - (PersonController -> /person/*) + @RequestMapping("edit") -> /person/edit
    - If request mapping not specified on method level, method name is taken instead -  edit() -> /edit
- SimpleUrlHandlerMapping
    - Mapping is defined declaratively
```xml
<bean class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping" >
  <property name= "mappings" >
    <value>
      /home=homeController
      /persons/**=personsController
    </value>
  </property>
</bean>
```
- Can have multiple chained HandlerMappings with specified order (same as view resolvers)
    - The first match wins
    - return of null means chain continues
- `<mvc:interceptors>` or `WebMvcConfigurer.addInterceptors()` adds interceptors for all mappings. To add interceptor to a specific mapping:
```xml
<bean class="org.springframework.web.servlet.mvc.support.ControllerClassNameHandlerMapping">
  <property name="interceptors" >
    <list>
      <bean class="com.example.FooInterceptor"/>
      <bean class="com.example.BarInterceptor"/>
    </list>
  </property>
</bean>
```
- Spring boot enables by default RequestMappingHandlerMapping, BeanNameUrlHandlerMapping, SimpleUrlHandlerMapping

#HandlerAdapter
- Adapter for invoking specific Handlers (one type of handler is @Controller)
- Has method handle(request,response,controller)
- Shields Dispatcher Servlet from specific types of Handlers
- WebflowHandlerAdapter, HttpRequestHandlerAdapter (remoting, resource handling,...)
- Spring Boot enables by default RequestMappingHandlerAdapter, HttpRequestHandlerAdapter, SimpleControllerHandlerAdapter
- RequestMappingHandlerAdapter
    - Adapts calls to @RequestMapping methods
    - Injects method params such as Model, HttpServletRequest, @PathVariables etc.
    - Interprets method return value - logical view name/ModelAndView/@ResponseBody/...
    - Supports Optional<> - for @RequestParam, @RequestHeader ("Pragma"),...
    - Enabled by default