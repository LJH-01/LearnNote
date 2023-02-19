
# Spring中的${}与#{}源码解析
## ${} 源码解析

```java
public String resolveEmbeddedValue(@Nullable String value) {
   if (value == null) {
      return null;
   }
   String result = value;
   // 遍历 StringValueResolver
   for (StringValueResolver resolver : this.embeddedValueResolvers) {
      result = resolver.resolveStringValue(result);
      if (result == null) {
         return null;
      }
   }
   return result;
}
```

```java
public String resolveStringValue(String strVal) throws BeansException {
   String resolved = this.helper.replacePlaceholders(strVal, this.resolver);
   if (trimValues) {
      resolved = resolved.trim();
   }
   return (resolved.equals(nullValue) ? null : resolved);
}
```

```java
public String replacePlaceholders(String value, PlaceholderResolver placeholderResolver) {
   Assert.notNull(value, "'value' must not be null");
   return parseStringValue(value, placeholderResolver, new HashSet<>());
}
```

继续分析parseStringValue

```java
/**
 * 解析 ${}，得到${}的值a之后，从Spring工厂中根据a获取对应的配置属性
 *
 * @param value 需要解析的字符串
 * @param placeholderResolver placeholderResolver
 * @param visitedPlaceholders visitedPlaceholders 防止循环引用
 * @return 获取入参字符串的对应的配置属性
 */
protected String parseStringValue(
      String value, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {

   StringBuilder result = new StringBuilder(value);

   // 找到 ${ 位置
   int startIndex = value.indexOf(this.placeholderPrefix);
   while (startIndex != -1) {
      // 找到最后一个 } 位置
      int endIndex = findPlaceholderEndIndex(result, startIndex);
      if (endIndex != -1) {
         // 拿到 ${} 中的值
         String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
         String originalPlaceholder = placeholder;
         // 循环引用则抛异常
         if (!visitedPlaceholders.add(originalPlaceholder)) {
            throw new IllegalArgumentException(
                  "Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
         }
         // Recursive invocation, parsing placeholders contained in the placeholder key.递归处理
         placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
         // Now obtain the value for the fully resolved key...
         String propVal = placeholderResolver.resolvePlaceholder(placeholder);
         // 解析默认值 valueSeparator是:
         if (propVal == null && this.valueSeparator != null) {
            int separatorIndex = placeholder.indexOf(this.valueSeparator);
            if (separatorIndex != -1) {
               String actualPlaceholder = placeholder.substring(0, separatorIndex);
               // 获取:之前的字符串
               String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
               // 解析默认值，递归处理
               propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
               if (propVal == null) {
                  propVal = defaultValue;
               }
            }
         }
         if (propVal != null) {
            // Recursive invocation, parsing placeholders contained in the
            // previously resolved placeholder value.
            // 解析 获取到的字符串，该字符串中可能含有${}，递归处理
            propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
            result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
            if (logger.isTraceEnabled()) {
               logger.trace("Resolved placeholder '" + placeholder + "'");
            }
            startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
         }
         else if (this.ignoreUnresolvablePlaceholders) {
            // Proceed with unprocessed value.
            startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
         }
         else {
            throw new IllegalArgumentException("Could not resolve placeholder '" +
                  placeholder + "'" + " in value \"" + value + "\"");
         }
         visitedPlaceholders.remove(originalPlaceholder);
      }
      else {
         startIndex = -1;
      }
   }
   // 获取结果
   return result.toString();
}
```



## #{} 解析源码详见：

https://blog.csdn.net/quyixiao/article/details/108635298



