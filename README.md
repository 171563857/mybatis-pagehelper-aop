# mybatis-pagehelper-aop
## 通过aop拦截mybatis的dao类,然后使用pagehelper.startPage设置物理分页,如果帮助到你,请点个赞吧:)

### 目前我使用的是springboot2,所以只提供java注解方式,后续有空会增加其它的

## 首先在Application.java所在包下创建PageHelperAOP类用于aop切面拦截

## 其次创建"PageInterceptor"类用于重写pagehelper的拦截类"PageInterceptor"

# aop拦截类
```
package com.xxx.aop;

import com.github.pagehelper.PageHelper;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Repository;

import java.util.Arrays;

/**
 * author: GoL
 * time:   2018-05-12
 */
@Aspect
@Repository
public class PageHelperAOP {
    //我的dao类是放在com.xxx.dao包下,此处拦截dao类下的get*,find*,select*方法,这里大家自行调整
    //自行了解execution规则
    @Pointcut("execution(* com.xxx.*.dao.*Dao.get*(..)) || " +
            "execution(* com.xxx.*.dao.*Dao.find*(..)) || " +
            "execution(* com.xxx.*.dao.*Dao.select*(..))")
    public void pageHelperAop() {
    }

    @Before("pageHelperAop()")
    public void beforePageHelper(JoinPoint point) {
        //判断dao方法是否传入Pageable对象,如果有则设置分页,如果没有则不设置
        Pageable pageable = (Pageable) Arrays.stream(point.getArgs()).filter(p -> p instanceof Pageable).findFirst().orElse(null);
        if (null != pageable) {
            PageHelper.startPage(pageable.getPageNumber(), pageable.getPageSize());
        }
    }
}
```

# PageInterceptor 拦截

### 直接复制pagehelper工具的PageInterceptor类,然后修改里面的传参,由于在映射mybatis时有设置parameterType,会当传入多个参数时会导致报错异常

### 所以要过滤到pageable对象,Object parameter = args[1];
### 这就是参入的参数,会有 arg*和param*,arg*是原传入参数,param*是封装后要传入mybatis的对象

### 添加私有方法filterPageable,用于过滤pageable对象

```
public Object intercept(Invocation invocation) throws Throwable {
    //...省略上面
    Object parameter = args[1];
    Method method = invocation.getMethod();
    //过滤query方法
    if (method.getName().equals("query") && parameter instanceof MapperMethod.ParamMap && null != parameter) {
        parameter = filterPageable((MapperMethod.ParamMap) parameter);
    }
    //...省略下面
}
private Object filterPageable(MapperMethod.ParamMap paramMap) {
    Map<String, Object> map = new HashMap<>();
    boolean havePageable = false;
    for (Object key : paramMap.keySet()) {
        if (String.valueOf(key).contains("arg")) continue;
        Object value = paramMap.get(key);
        if (value instanceof Pageable) {
            havePageable = true;
            continue;
        }
        if (value instanceof Map) {
            map.putAll((HashMap) value);
        } else {
            map.put(String.valueOf(key), value);
        }
    }
    return havePageable ? map : paramMap;
}
```

# mybatis的dao类

## dao类要用springboot可以扫描到的注解,不然无法做aop和拦截

## aop要拦截的就是findxxx方法,然后获取Pageable并通过pageHelper.startPage设置分页
```
import org.springframework.data.domain.Pageable;
@Componet 
public interface TestDao {
  List<Test> findxxx(Map<String, Object> param, Pageable pageable);
}
```

# 今天只做了这个,后续会不断扩展方法,让方法更加通用:)

# 如果解决你的问题,请点个赞吧:)
