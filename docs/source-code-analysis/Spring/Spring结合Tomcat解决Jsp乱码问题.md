# Spring结合Tomcat解决Jsp乱码问题



前置知识：[使用SpringMVC来解析资源](SpringMVC/SpringMVC系列二：SpringMVC的HandlerMapping.md)

分析Tomcat的 url 匹配优先级

```java
private final void internalMapWrapper(ContextVersion contextVersion,
                                      CharChunk path,
                                      MappingData mappingData) throws IOException {

    int pathOffset = path.getOffset();
    int pathEnd = path.getEnd();
    boolean noServletPath = false;

    int length = contextVersion.path.length();
    if (length == (pathEnd - pathOffset)) {
        noServletPath = true;
    }
    int servletPath = pathOffset + length;
    path.setOffset(servletPath);

    // Rule 1 -- Exact Match
    MappedWrapper[] exactWrappers = contextVersion.exactWrappers;
    internalMapExactWrapper(exactWrappers, path, mappingData);

    // Rule 2 -- Prefix Match
    boolean checkJspWelcomeFiles = false;
    MappedWrapper[] wildcardWrappers = contextVersion.wildcardWrappers;
    if (mappingData.wrapper == null) {
        internalMapWildcardWrapper(wildcardWrappers, contextVersion.nesting,
                                   path, mappingData);
        if (mappingData.wrapper != null && mappingData.jspWildCard) {
            char[] buf = path.getBuffer();
            if (buf[pathEnd - 1] == '/') {
                /*
                 * Path ending in '/' was mapped to JSP servlet based on
                 * wildcard match (e.g., as specified in url-pattern of a
                 * jsp-property-group.
                 * Force the context's welcome files, which are interpreted
                 * as JSP files (since they match the url-pattern), to be
                 * considered. See Bugzilla 27664.
                 */
                mappingData.wrapper = null;
                checkJspWelcomeFiles = true;
            } else {
                // See Bugzilla 27704
                mappingData.wrapperPath.setChars(buf, path.getStart(),
                                                 path.getLength());
                mappingData.pathInfo.recycle();
            }
        }
    }

    if(mappingData.wrapper == null && noServletPath &&
            contextVersion.object.getMapperContextRootRedirectEnabled()) {
        // The path is empty, redirect to "/"
        path.append('/');
        pathEnd = path.getEnd();
        mappingData.redirectPath.setChars
            (path.getBuffer(), pathOffset, pathEnd - pathOffset);
        path.setEnd(pathEnd - 1);
        return;
    }

    // Rule 3 -- Extension Match
    MappedWrapper[] extensionWrappers = contextVersion.extensionWrappers;
    if (mappingData.wrapper == null && !checkJspWelcomeFiles) {
        internalMapExtensionWrapper(extensionWrappers, path, mappingData,
                true);
    }

    // Rule 4 -- Welcome resources processing for servlets
    if (mappingData.wrapper == null) {
        boolean checkWelcomeFiles = checkJspWelcomeFiles;
        if (!checkWelcomeFiles) {
            char[] buf = path.getBuffer();
            checkWelcomeFiles = (buf[pathEnd - 1] == '/');
        }
        if (checkWelcomeFiles) {
            for (int i = 0; (i < contextVersion.welcomeResources.length)
                     && (mappingData.wrapper == null); i++) {
                path.setOffset(pathOffset);
                path.setEnd(pathEnd);
                path.append(contextVersion.welcomeResources[i], 0,
                        contextVersion.welcomeResources[i].length());
                path.setOffset(servletPath);

                // Rule 4a -- Welcome resources processing for exact macth
                internalMapExactWrapper(exactWrappers, path, mappingData);

                // Rule 4b -- Welcome resources processing for prefix match
                if (mappingData.wrapper == null) {
                    internalMapWildcardWrapper
                        (wildcardWrappers, contextVersion.nesting,
                         path, mappingData);
                }

                // Rule 4c -- Welcome resources processing
                //            for physical folder
                if (mappingData.wrapper == null
                    && contextVersion.resources != null) {
                    String pathStr = path.toString();
                    WebResource file =
                            contextVersion.resources.getResource(pathStr);
                    if (file != null && file.isFile()) {
                        internalMapExtensionWrapper(extensionWrappers, path,
                                                    mappingData, true);
                        if (mappingData.wrapper == null
                            && contextVersion.defaultWrapper != null) {
                            mappingData.wrapper =
                                contextVersion.defaultWrapper.object;
                            mappingData.requestPath.setChars
                                (path.getBuffer(), path.getStart(),
                                 path.getLength());
                            mappingData.wrapperPath.setChars
                                (path.getBuffer(), path.getStart(),
                                 path.getLength());
                            mappingData.requestPath.setString(pathStr);
                            mappingData.wrapperPath.setString(pathStr);
                        }
                    }
                }
            }

            path.setOffset(servletPath);
            path.setEnd(pathEnd);
        }

    }

    /* welcome file processing - take 2
     * Now that we have looked for welcome files with a physical
     * backing, now look for an extension mapping listed
     * but may not have a physical backing to it. This is for
     * the case of index.jsf, index.do, etc.
     * A watered down version of rule 4
     */
    if (mappingData.wrapper == null) {
        boolean checkWelcomeFiles = checkJspWelcomeFiles;
        if (!checkWelcomeFiles) {
            char[] buf = path.getBuffer();
            checkWelcomeFiles = (buf[pathEnd - 1] == '/');
        }
        if (checkWelcomeFiles) {
            for (int i = 0; (i < contextVersion.welcomeResources.length)
                     && (mappingData.wrapper == null); i++) {
                path.setOffset(pathOffset);
                path.setEnd(pathEnd);
                path.append(contextVersion.welcomeResources[i], 0,
                            contextVersion.welcomeResources[i].length());
                path.setOffset(servletPath);
                internalMapExtensionWrapper(extensionWrappers, path,
                                            mappingData, false);
            }

            path.setOffset(servletPath);
            path.setEnd(pathEnd);
        }
    }


    // Rule 7 -- Default servlet
    if (mappingData.wrapper == null && !checkJspWelcomeFiles) {
        if (contextVersion.defaultWrapper != null) {
            mappingData.wrapper = contextVersion.defaultWrapper.object;
            mappingData.requestPath.setChars
                (path.getBuffer(), path.getStart(), path.getLength());
            mappingData.wrapperPath.setChars
                (path.getBuffer(), path.getStart(), path.getLength());
            mappingData.matchType = MappingMatch.DEFAULT;
        }
        // Redirection to a folder
        char[] buf = path.getBuffer();
        if (contextVersion.resources != null && buf[pathEnd -1 ] != '/') {
            String pathStr = path.toString();
            // Note: Check redirect first to save unnecessary getResource()
            //       call. See BZ 62968.
            if (contextVersion.object.getMapperDirectoryRedirectEnabled()) {
                WebResource file;
                // Handle context root
                if (pathStr.length() == 0) {
                    file = contextVersion.resources.getResource("/");
                } else {
                    file = contextVersion.resources.getResource(pathStr);
                }
                if (file != null && file.isDirectory()) {
                    // Note: this mutates the path: do not do any processing
                    // after this (since we set the redirectPath, there
                    // shouldn't be any)
                    path.setOffset(pathOffset);
                    path.append('/');
                    mappingData.redirectPath.setChars
                        (path.getBuffer(), path.getStart(), path.getLength());
                } else {
                    mappingData.requestPath.setString(pathStr);
                    mappingData.wrapperPath.setString(pathStr);
                }
            } else {
                mappingData.requestPath.setString(pathStr);
                mappingData.wrapperPath.setString(pathStr);
            }
        }
    }

    path.setOffset(pathOffset);
    path.setEnd(pathEnd);
}
```

结论：

1. Rule 1 -- Exact Match：精确匹配
2. Rule 2 -- Prefix Match：前缀匹配
3. Rule 3 -- Extension Match：后缀匹配
4. Rule 4 -- Welcome resources processing for servlets：欢迎页
5. Rule 7 -- Default servlet：默认/匹配

所以Spring结合Tomcat也可以玩转jsp，只需要配置DispatcherServlet 的路径映射为/

```java
// 添加 DispatcherServlet
DispatcherServlet dispatcherServlet = new DispatcherServlet(ctx);
dispatcherServlet.setThrowExceptionIfNoHandlerFound(true);
ServletRegistration.Dynamic springmvc = servletContext.addServlet("springmvc", dispatcherServlet);
// 给 DispatcherServlet 添加路径映射
springmvc.addMapping("/");
// 给 DispatcherServlet 添加启动时机
springmvc.setLoadOnStartup(1);
```

这样就先匹配到 jsp 处理的 servlet：JspServlet来进行视图渲染

如果匹配的DispatcherServlet 的路径映射为/*，那么会导致

```java
RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
```

获取的 StandardWrapper 中的 Servlet 还是 DispatcherServlet 导致死循环，最终导致请求没有办法交给JspServlet 来渲染
