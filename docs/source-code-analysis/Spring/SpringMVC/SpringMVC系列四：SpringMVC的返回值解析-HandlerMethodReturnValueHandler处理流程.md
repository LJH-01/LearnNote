# SpringMVC的http请求的返回值解析器HandlerMethodReturnValueHandler来处理返回值

[TOC]



## 本文分析的HandlerMethodArgumentResolver

![image-20221123204706649](assets/image-20221123204706649.png)

RequestResponseBodyMethodProcessor

ModelAndViewMethodReturnValueHandler

## 处理方法的返回值

### @ResponseBody标注方法的返回值解析器是：RequestResponseBodyMethodProcessor



#### 简单使用

```java
@Override
	protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
		converters.clear();
		converters.add(stringHttpMessageConverter());
		converters.add(fastJsonHttpMessageConverter());
	}
@Bean
	public FastJsonHttpMessageConverter fastJsonHttpMessageConverter() {
		FastJsonHttpMessageConverter fastJsonHttpMessageConverter = new FastJsonHttpMessageConverter();

		FastJsonConfig fastJsonConfig = new FastJsonConfig();
		fastJsonConfig.setSerializerFeatures(
				SerializerFeature.QuoteFieldNames,
				SerializerFeature.WriteMapNullValue,//保留空的字段
				SerializerFeature.WriteNullListAsEmpty,//List null-> []
				SerializerFeature.WriteDateUseDateFormat,// 日期格式化
				SerializerFeature.WriteNullStringAsEmpty);//String null -> ""

		List<MediaType> mediaTypeList = new ArrayList<>();
		mediaTypeList.add(MediaType.APPLICATION_JSON_UTF8);
		mediaTypeList.add(MediaType.APPLICATION_JSON);
		fastJsonHttpMessageConverter.setSupportedMediaTypes(mediaTypeList);
		fastJsonHttpMessageConverter.setFastJsonConfig(fastJsonConfig);
		return fastJsonHttpMessageConverter;
	}

/**
 * 在ResponseBody注解下，Spring处理返回值为String时会用到StringHttpMessageConverter
 *
 */
@Bean
public StringHttpMessageConverter stringHttpMessageConverter() {
	StringHttpMessageConverter httpMessageConverter = new StringHttpMessageConverter();
	httpMessageConverter.setDefaultCharset(Charset.forName("UTF-8"));
	return httpMessageConverter;
}
```



#### 判断能否处理返回值

```java
public boolean supportsReturnType(MethodParameter returnType) {
   return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
         returnType.hasMethodAnnotation(ResponseBody.class));
}
```

类上或者请求方法上有@ResponseBody方法

#### 处理返回值

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
      throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

   mavContainer.setRequestHandled(true);
   ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
   ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

   // Try even with null return value. ResponseBodyAdvice could get involved.
   // 使用HttpMessageConventer来输出
   if (logger.isDebugEnabled()) {
      logger.debug("使用HttpMessageConventer来输出");
   }
   writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```



```java
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
      ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
      throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

   Object outputValue;
   Class<?> valueType;
   Type declaredType;

   if (value instanceof CharSequence) {
      outputValue = value.toString();
      valueType = String.class;
      declaredType = String.class;
   }
   else {
      outputValue = value;
      valueType = getReturnValueType(outputValue, returnType);
      declaredType = getGenericType(returnType);
   }

   if (isResourceType(value, returnType)) {
      outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
      if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null) {
         Resource resource = (Resource) value;
         try {
            List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
            outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
            outputValue = HttpRange.toResourceRegions(httpRanges, resource);
            valueType = outputValue.getClass();
            declaredType = RESOURCE_REGION_LIST_TYPE;
         }
         catch (IllegalArgumentException ex) {
            outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
            outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
         }
      }
   }


   List<MediaType> mediaTypesToUse;

   MediaType contentType = outputMessage.getHeaders().getContentType();
   if (contentType != null && contentType.isConcrete()) {
      mediaTypesToUse = Collections.singletonList(contentType);
   }
   else {
      HttpServletRequest request = inputMessage.getServletRequest();
      // 获取客户端可接受的类型
      List<MediaType> requestedMediaTypes = getAcceptableMediaTypes(request);
      // 获取服务端可以生成的所有 MediaType 类型
      List<MediaType> producibleMediaTypes = getProducibleMediaTypes(request, valueType, declaredType);

      if (outputValue != null && producibleMediaTypes.isEmpty()) {
         throw new HttpMessageNotWritableException(
               "No converter found for return value of type: " + valueType);
      }
      // acceptableTypes 和 producibleTypes 比较，找出可用的 MediaType
      mediaTypesToUse = new ArrayList<>();
      for (MediaType requestedType : requestedMediaTypes) {
         for (MediaType producibleType : producibleMediaTypes) {
            if (requestedType.isCompatibleWith(producibleType)) {
               mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
            }
         }
      }
      if (mediaTypesToUse.isEmpty()) {
         if (outputValue != null) {
            throw new HttpMediaTypeNotAcceptableException(producibleMediaTypes);
         }
         return;
      }
      MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
   }

   MediaType selectedMediaType = null;
   for (MediaType mediaType : mediaTypesToUse) {
      if (mediaType.isConcrete()) {
         selectedMediaType = mediaType;
         break;
      }
      else if (mediaType.equals(MediaType.ALL) || mediaType.equals(MEDIA_TYPE_APPLICATION)) {
         selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
         break;
      }
   }

   if (selectedMediaType != null) {
      selectedMediaType = selectedMediaType.removeQualityValue();
      for (HttpMessageConverter<?> converter : this.messageConverters) {
         GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
               (GenericHttpMessageConverter<?>) converter : null);
         // 判断 HttpMessageConverter 是否支持转换目标类型
         if (genericConverter != null ?
               ((GenericHttpMessageConverter) converter).canWrite(declaredType, valueType, selectedMediaType) :
               converter.canWrite(valueType, selectedMediaType)) {
            outputValue = getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
                  (Class<? extends HttpMessageConverter<?>>) converter.getClass(),
                  inputMessage, outputMessage);
            // body 非空，则进行写入
            if (outputValue != null) {
               addContentDispositionHeader(inputMessage, outputMessage);
               // 真正则进行写入
               if (genericConverter != null) {
                  genericConverter.write(outputValue, declaredType, selectedMediaType, outputMessage);
               }
               else {
                  // 真正则进行写入
                  ((HttpMessageConverter) converter).write(outputValue, selectedMediaType, outputMessage);
               }
               if (logger.isDebugEnabled()) {
                  logger.debug("Written [" + outputValue + "] as \"" + selectedMediaType +
                        "\" using [" + converter + "]");
               }
            }
            return;
         }
      }
   }

   if (outputValue != null) {
      throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
   }
}
```

##### 如果返回值是String类型使用StringHttpMessageConverter

```java
@Override
public boolean supports(Class<?> clazz) {
   return String.class == clazz;
}
```

##### 如果返回值是其他的使用FastJsonHttpMessageConverter

```java
@Override
protected boolean supports(Class<?> clazz) {
    return true;
}
```

##### 共同的真正的写入逻辑

```java
public final void write(final T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
      throws IOException, HttpMessageNotWritableException {

   final HttpHeaders headers = outputMessage.getHeaders();
   // 如果 Content-Type 为空则设置默认的
   addDefaultHeaders(headers, t, contentType);

   if (outputMessage instanceof StreamingHttpOutputMessage) {
      StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) outputMessage;
      streamingOutputMessage.setBody(outputStream -> writeInternal(t, new HttpOutputMessage() {
         @Override
         public OutputStream getBody() {
            return outputStream;
         }
         @Override
         public HttpHeaders getHeaders() {
            return headers;
         }
      }));
   }
   else {
      // 写数据到 Response.OutputBuffer 中
      writeInternal(t, outputMessage);
      // 直接将数据写到 socket 缓存区发送到网络上
      outputMessage.getBody().flush();
   }
}
```

**其中writeInternal(t, outputMessage);走的是两个子类各个的实现类**

如果 Content-Type 为空则设置默认的

```java
protected void addDefaultHeaders(HttpHeaders headers, T t, @Nullable MediaType contentType) throws IOException {
   if (headers.getContentType() == null) {
      MediaType contentTypeToUse = contentType;
      if (contentType == null || contentType.isWildcardType() || contentType.isWildcardSubtype()) {
         contentTypeToUse = getDefaultContentType(t);
      }
      else if (MediaType.APPLICATION_OCTET_STREAM.equals(contentType)) {
         MediaType mediaType = getDefaultContentType(t);
         contentTypeToUse = (mediaType != null ? mediaType : contentTypeToUse);
      }
      if (contentTypeToUse != null) {
         if (contentTypeToUse.getCharset() == null) {
            // 设置默认的字符集
            Charset defaultCharset = getDefaultCharset();
            if (defaultCharset != null) {
               contentTypeToUse = new MediaType(contentTypeToUse, defaultCharset);
            }
         }
         // 设置 Content-Type
         headers.setContentType(contentTypeToUse);
      }
   }
   if (headers.getContentLength() < 0 && !headers.containsKey(HttpHeaders.TRANSFER_ENCODING)) {
      Long contentLength = getContentLength(t, headers.getContentType());
      if (contentLength != null) {
         headers.setContentLength(contentLength);
      }
   }
}
```



**最终利用outputMessage.getBody().flush()将数据 写到网络中**

#### 小结：

1. 获取支持的HttpMessageConverter
2. 添加 Content-Type 设置响应头之后，直接将数据写到 socket 缓存区发送到网络上
3. @ResponseBody标注方法直接将数据刷到网络中，普通的是tomcat的servlet处理完之后由tomcat把数据刷到网络上



### 其他的返回值解析器是：ModelAndViewMethodReturnValueHandler

#### 判断能否处理返回值

```java
@Override
public boolean supportsReturnType(MethodParameter returnType) {
   return ModelAndView.class.isAssignableFrom(returnType.getParameterType());
}
```

处理方法的返回值的ModelAndView的请求

#### 处理返回值

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

   if (returnValue == null) {
      mavContainer.setRequestHandled(true);
      return;
   }

   ModelAndView mav = (ModelAndView) returnValue;
   if (mav.isReference()) {
      String viewName = mav.getViewName();
      mavContainer.setViewName(viewName);
      if (viewName != null && isRedirectViewName(viewName)) {
         mavContainer.setRedirectModelScenario(true);
      }
   }
   else {
      View view = mav.getView();
      mavContainer.setView(view);
      if (view instanceof SmartView && ((SmartView) view).isRedirectView()) {
         mavContainer.setRedirectModelScenario(true);
      }
   }
   mavContainer.setStatus(mav.getStatus());
   // 将 ModelAndView 中的 Model 加入到 ModelAndViewContainer 中
   mavContainer.addAllAttributes(mav.getModel());
}
```

之后将ModelAndViewContainer中的 model重新设置到 ModelAndView 中

```java
// 完成过程调用
// 对请求参数进行处理，调用目标HandlerMethod，并且将返回值封装为一个ModelAndView对象
invocableMethod.invokeAndHandle(webRequest, mavContainer);
if (asyncManager.isConcurrentHandlingStarted()) {
   return null;
}

// 包装ModelAndView，将 mavContainer 中的 model 设置到 ModelAndView 中
return getModelAndView(mavContainer, modelFactory, webRequest);
```

```java
private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
      ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {

   modelFactory.updateModel(webRequest, mavContainer);
   if (mavContainer.isRequestHandled()) {
      return null;
   }
   ModelMap model = mavContainer.getModel();
   // 将 mavContainer 中的 model 设置到 ModelAndView 中
   ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
   if (!mavContainer.isViewReference()) {
      mav.setView((View) mavContainer.getView());
   }
   if (model instanceof RedirectAttributes) {
      Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
      HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
      if (request != null) {
         RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
      }
   }
   return mav;
}
```

并在解析页面的时候将 model 中的值设置到 request的attribute中

```java
protected void exposeModelAsRequestAttributes(Map<String, Object> model,
      HttpServletRequest request) throws Exception {

   // 将 model 中的值设置到 request 的 attribute中
   model.forEach((modelName, modelValue) -> {
      if (modelValue != null) {
         request.setAttribute(modelName, modelValue);
         if (logger.isDebugEnabled()) {
            logger.debug("Added model object '" + modelName + "' of type [" + modelValue.getClass().getName() +
                  "] to request in view with name '" + getBeanName() + "'");
         }
      }
      else {
         request.removeAttribute(modelName);
         if (logger.isDebugEnabled()) {
            logger.debug("Removed model object '" + modelName +
                  "' from request in view with name '" + getBeanName() + "'");
         }
      }
   });
}
```

最终利用request.getAttribute() 来获取 model中的数据

#### 小结：

1. 将ModelAndView中 model 中的值 设置到 request 的 attribute中

