---
layout: post
title: Zuul Auth
subtitle: zuul 认证
date: 2019-08-10
categories: java
cover: 
tags: all springcloud 框架 微服务
---

> 库存 java-springcloud 

## 简述
权限认证需求很多人都遇到过，身份认证可以采用jwt，然而接口权限却没什么受广泛接收的框架。

究其原因，个人认为接口权限设计本身与组织架构关联性很强，并非如同登录认证一般是一个比较独立的需求。

工作中我们自己设计了一套权限认证系统，配合我们自己的组织架构设计，包含了登录认证与接口权限

## 认证中心
此处简述一下认证中心设计

<img src="/img/oauth1.png">

所有接口存在一个map中
```go
var URLMAP URLMap

func init()  {
	URLMAP.mm = make(map[string]uint32)
}

type URLMap struct {
	mm    map[string]uint32
	index uint32
	sync.RWMutex
}

func (u *URLMap) Store(url string, index uint32) {
	u.Lock()
	defer u.Unlock()
	u.mm[url] = index
	if u.index < index {
		u.index = index
	}
}

func (u *URLMap) StoreAuto(url string) uint32 {
	u.Lock()
	defer u.Unlock()
	u.index++
	u.mm[url] = u.index
	return u.index
}

func (u *URLMap) Get(url string) (index uint32, ok bool) {
	u.RLock()
	defer u.RUnlock()
	index, ok = u.mm[url]
	return
}
```

用户权限数据及登录数据
```go
var (
	Lim LoginInfoMap
	Uim UserInfoMap
)

type UserInfo struct {
	OauthUser
	AuthSet        []uint32
	LastLoginToken string
	UserName 		string
}

func (u *UserInfo) GetUserAuth(){
	var err error
	if u.AuthStatus != constant.GETED { // 从角色模块获取权限
		err = GetAuthFromRemote(u)
	} else {
		u.AuthSet, err = (&UserApi{UserId:u.Id}).GetApiList()
	}
	if err != nil{
		logs.Error(err)
	}
	return
}

func (u *UserInfo) CheckAuth(operate string) bool {
	logs.Info("操作", operate)
	index, ok := URLMAP.Get(operate) // 先在接口集合中找到本接口对应的index
	if !ok {
		logs.Warn("权限不足")
		return false
	}
	for i, j := 0, len(u.AuthSet) -1; i <= j; { // 查找登录人是否包含该index
		m := (i + j)/ 2
		if u.AuthSet[m] == index {
			return true
		}
		if u.AuthSet[m] > index {
			j = m - 1
		}else {
			i = m + 1
		}
	}
	return false
}

type UserInfoMap struct {
	sync.Map
}

func (u *UserInfoMap) Get(abstract string) (userInfo *UserInfo, ok bool) {
	val, ok := u.Load(abstract)
	if ok {
		userInfo, ok = val.(*UserInfo)
	}
	return
}

func (u *UserInfoMap) Put(abstract string, val *UserInfo) {
	u.Store(abstract, val)
}

type LoginInfo struct {
	ExpireTime time.Time `json:"-"`
	User       *UserInfo
}

type LoginInfoMap struct {
	sync.Map
}

func (u *LoginInfoMap) Get(token string) (loginInfo *LoginInfo, ok bool) {
	val, ok := u.Load(token)
	if ok {
		loginInfo, ok = val.(*LoginInfo)
	}
	return
}

func (u *LoginInfoMap) Put(token string, val *LoginInfo) {
	u.Store(token, val)
}
```

- 登录用户信息存储在两个全局变量中（uim userInfoMap/  lim loginInfoMap）
- 通过**登录token**可以从lim中找到**过期时间**以及**用户信息**
- 通过**用户信息(用户名、客户id)摘要**可以从uim中找到**用户信息**以及**最近登录token**
#### 登录操作
- 当用户名不存在已登录情况时，校验密码，生成token，存储uim，lim
- 当用户名已登录时，判断该账号是否允许多人登录
  - 允许时，生成token，将uim中该用户信息的lastlogintoken设为当前token，存储lim
  - 不允许时，从lim删除之前登录信息，其他操作同允许情况

#### 权限信息存储结构
```
URLmap:{
    "url1":1
    "url2":2
    "url3":3
    .
    .
    .
}
userInfo:{
    "authStatus":"GETED" //UNGET、GETED、UPDATED
    "authSet":[1,2,3,5,6...]
}
```
#### 鉴权
- 鉴权时先通过URLmap获取url对应的id
- 再通过二分查找法在authSet中查找是否存在该id

认证中心不再赘述，相关代码在[github](https://github.com/chenwj93/oauth)

## zuul
微服务中认证流程放在gateway层毫无疑问是合适的，不多说，直接上代码。
（代码质量一般-_-!）

#### 代码结构
- java
  - com.adatafun.platform
    - filter
      - AccessFilter.java
    - remote
      - Auther.java
    - GatewayApplication.java

**Auther.java**
```java
package com.adatafun.platform.remote;

import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

import java.util.Map;

@FeignClient(name = "oauth") //service id
public interface Auther {
    @PostMapping(value = "/authenticate") // path
    Map authenticate(@RequestBody Map param); 
}
```

**GatewayApplication.java**
```java
package com.adatafun.platform;

import com.adatafun.platform.filter.AccessFilter;
import org.jboss.logging.Logger;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.cloud.netflix.feign.EnableFeignClients;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
import org.springframework.context.annotation.Bean;

/**
 * @Description:
 * @Date: Create in 14:30 2018-05-23
 * @Modified By
 */
@EnableFeignClients
@EnableZuulProxy
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args){
        new SpringApplicationBuilder(GatewayApplication.class).web(true).run(args);
    }

    /**
     * 实例化过滤器
     * @return
     */
    @Bean
    public AccessFilter accessFilter(){
        return new AccessFilter();
    }

    @Bean
    Logger.Level feginLoggerLevel(){
        return Logger.Level.DEBUG;
    }
}
```
**AccessFilter**
```java
package com.adatafun.platform.filter;

import com.adatafun.platform.remote.Auther;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.netflix.zuul.http.ServletInputStreamWrapper;
import org.apache.commons.lang.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.util.StreamUtils;

import javax.servlet.ServletInputStream;
import javax.servlet.ServletOutputStream;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.Charset;
import java.util.HashMap;
import java.util.Map;

public class AccessFilter extends ZuulFilter {

    private static Logger log = LoggerFactory.getLogger(AccessFilter.class);
    @Autowired
    Auther auth;

    /**
     * 过滤类型
     * pre 可以在请求被路由之前调用
     * routing 在路由请求时被调用
     * post 在routing和error过滤器之后被调用
     * error 处理请求时发生错误时被调用
     *
     * @return
     */
    @Override
    public String filterType() {
        return "pre";
    }

    /**
     * 执行顺序
     * 通过int值来定义过滤器的执行顺序，数值越小优先级越高
     *
     * @return
     */
    @Override
    public int filterOrder() {
        return 0;
    }

    /**
     * 执行条件
     * 返回一个boolean值来判断该过滤器是否需要执行，我们可以通过此方法来指定过滤器的有效范围
     *
     * @return
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    /**
     * 具体操作
     * 过滤器的具体逻辑。
     * modified by chenwj on 2018-08-14
     * @return
     */
    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        String operate = request.getRequestURI(); 
        log.info("send {} request to {}", request.getMethod(), url);
        if (operate.endsWith("/login")) {
            return null;
        }
        Object accessToken = getAccessToken(request);
        if (accessToken == null) { // 既非登录请求 亦无access_token
            log.warn("access token is empty");
            setCtxDeny(ctx, HttpServletResponse.SC_UNAUTHORIZED);
            return null;
        }

        Map param = request.getParameterMap();
        param.put("access_token", accessToken);
        param.put("operate", operate);
        try {
            Map authRet = auth.authenticate(param);
            log.info(authRet.toString());
            setCtxWithAuthRet(ctx, authRet);
        } catch (Exception e) {
            setCtxDeny(ctx, HttpServletResponse.SC_FORBIDDEN);
        }

        return null;
    }

    private void setCtxDeny(RequestContext ctx, int status) {
        ctx.setSendZuulResponse(false);
        setResponse(ctx, status);
    }
    /**
    *  修改request的body或head
    */
    private void setCtxWithAuthRet(RequestContext ctx, Map authRet) {
        if ("200".equals(authRet.get("status").toString())) {
            Object data = authRet.get("data");
            try {
                InputStream in = ctx.getRequest().getInputStream();
                String body = StreamUtils.copyToString(in, Charset.forName("UTF-8"));
                if (StringUtils.isBlank(body)) {
                    body = "{}";
                }
                JSONObject jsonObject = JSON.parseObject(body);
                if (data instanceof Map) {
                    Map dataMap = (Map) data;
                    for (Object obj : dataMap.entrySet()) {
                        Map.Entry<String, Object> entry = (Map.Entry<String, Object>) obj;
                        jsonObject.put(entry.getKey(), entry.getValue()); // 此处注意中文encode
                    }
                }
                String newBody = jsonObject.toString();
                byte[] reqBodyBytes = newBody.getBytes();
                ctx.setRequest(newRequestWrapper(ctx.getRequest(), reqBodyBytes));
                ctx.getZuulRequestHeaders().put("access_token", "");
            } catch (Exception ee) {
                ee.printStackTrace();
                setResponse(ctx, HttpServletResponse.SC_BAD_REQUEST);
            }
        } else {
            ctx.setSendZuulResponse(false);
            setResponse(ctx, HttpServletResponse.SC_UNAUTHORIZED);
        }
    }
    
    /**
    * 关键部分
    */
    private HttpServletRequestWrapper newRequestWrapper(HttpServletRequest request, final byte[] reqBodyBytes){
        return new HttpServletRequestWrapper(request) {
            @Override
            public ServletInputStream getInputStream() throws IOException {
                return new ServletInputStreamWrapper(reqBodyBytes);
            }

            @Override
            public int getContentLength() {
                return reqBodyBytes.length;
            }

            @Override
            public long getContentLengthLong() {
                return reqBodyBytes.length;
            }
        };
    }

    private void setResponse(RequestContext ctx, int status) {
        ctx.setResponseStatusCode(status);
        HttpServletResponse response = ctx.getResponse();
        try {
            ServletOutputStream sos = response.getOutputStream();
            HashMap<String, Object> data = new HashMap<String, Object>() {
                {
                    put("status", status);
                }
            };
            String js = JSON.toJSONString(data);
            sos.write(js.getBytes());
            ctx.setResponse(response);
        } catch (Exception e) {
            log.error(e.toString());
            e.printStackTrace();
        }
    }

    private Object getAccessToken(HttpServletRequest request) {
        Object accessToken = request.getHeader("access_token");
        if (accessToken == null) {
            accessToken = request.getParameter("access_token");
        }
        return accessToken;
    }
}

```
zuul层还可以做执行时间统计、认证接口排除等操作，以后再补充