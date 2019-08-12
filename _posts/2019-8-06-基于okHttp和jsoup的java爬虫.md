---
layout:     post
title:      使用okHttp和Jsoup的Java爬虫实现
subtitle:   —— okHttp 3.11 + jsoup 1.10.3 + fastjson
date:       2019-08-12
author:     Zwx
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - java
    - 爬虫
---

话不多说，直接上代码：

## 依赖
```xml
...
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>3.11.0</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.jsoup</groupId>
            <artifactId>jsoup</artifactId>
            <version>1.10.3</version>
        </dependency>
...
```

## 工具类
```java
package com.yc.message.utils;

import lombok.extern.slf4j.Slf4j;
import okhttp3.FormBody;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import org.apache.commons.lang.StringUtils;

import java.io.IOException;

/**
 * @program: message
 * @description: okhttp 发请求
 * @author: Zwx
 * @create: 2019-07-30 18:44
 **/
@Slf4j
public class RequestSupport {

    /**
     * 网易红彩 start
     * @return
     * @throws IOException
     */
    public static String findWangyiFreeRecommandIds() throws IOException {
        String url = "https://hongcai.163.com/api/front/free/v3";
        OkHttpClient okHttpClient = new OkHttpClient();
        Request request = new Request.Builder().url(url).build();
        return okHttpClient.newCall(request).execute().body().string();
    }

    public static String findWangyiFreeRecommand(String tId) throws IOException {
        String url = "https://hongcai.163.com/api/front/thread/v4/query/" + tId+ "/0";
        OkHttpClient okHttpClient = new OkHttpClient();
        Request request = new Request.Builder().url(url).build();
        return okHttpClient.newCall(request).execute().body().string();
    }
    /**
     * 网易红彩 end
     */

    /**
     * 新浪足球
     * @return
     * @throws IOException
     */
    public static String findSinaFootball() throws IOException {
        String url = "http://sports.sina.com.cn/l/football/";
        OkHttpClient okHttpClient = new OkHttpClient();
        Request request = new Request.Builder().url(url).build();
        return okHttpClient.newCall(request).execute().body().string();
    }
    /**
    * 新浪足球 end
    */



    /**
     * @param url
     * @param requestBody
     * @param cookie
     * @return 发post请求方法
     * @throws IOException
     */
    private static String sendBodyRequest(String url, RequestBody requestBody, String cookie) throws IOException {

        OkHttpClient okHttpClient = new OkHttpClient();
        Request request;
        if (StringUtils.isBlank(cookie)) {
            request = new Request.Builder().url(url).post(requestBody).build();
        } else {
            request = new Request.Builder().url(url).addHeader("Cookie", cookie).post(requestBody).build();
        }

        return okHttpClient.newCall(request).execute().body().string();
    }

    /**
     *  发GET请求方法
     * @param url
     * @return 
     * @throws IOException
     */
    public static String sendGetRequest(String url) throws IOException {
        OkHttpClient okHttpClient = new OkHttpClient();
        Request request = new Request.Builder().url(url).build();
        return okHttpClient.newCall(request).execute().body().string();
    }
}

```

## 业务类
```java
    @Override
    public List<String> findFreeIds() throws IOException {
        List<String> threadIds = new ArrayList<>();
        JSONObject jsonObject = JSON.parseObject(RequestSupport.findWangyiFreeRecommandIds());
        JSONArray array = jsonObject.getJSONObject("data").getJSONArray("freeThreads");

        for(int i=0;i<array.size();i++) {
            threadIds.add(array.getJSONObject(i).getString("threadId"));
        }

        return threadIds;
    }

    @Override
    public List<WangyiRecommand> findWYFreeMessage(List<String> threadIds) throws IOException {
        List<WangyiRecommand> wangyiRecommands = new ArrayList<>();
        for (String tId : threadIds){
            JSONObject jsonObject = JSON.parseObject(RequestSupport.findWangyiFreeRecommand(tId));
            String title = jsonObject.getJSONObject("data").getString("title");
            String content = jsonObject.getJSONObject("data").getString("content")
                    .replace("<p>","")
                    .replace("</p>","")
                    .replace("</b>","")
                    .replace("<b>","")
                    .replace("<br>","")
                    .replace("</br>","");
            wangyiRecommands.add(new WangyiRecommand(title,content));
        }
        return wangyiRecommands;
    }
    
        @Override
        public List<WangyiRecommand> findSinaRecommand(List<String> urls) throws IOException {
    
            List<WangyiRecommand> sinaRecommands = new ArrayList<>();
            for (String url : urls){
                Document document = Jsoup.parse(RequestSupport.sendGetRequest(url));
                Elements elements = document.select("p");
                StringBuilder s = new StringBuilder();
                for(Element e:elements){
                    if(e.select("p").text().contains("　　")){
                        s.append(e.select("p").text());
                    }
                }
                WangyiRecommand recommand =new WangyiRecommand(document.select("h1").text(),s.toString());
                sinaRecommands.add(recommand);
            }
    
            return sinaRecommands;
        }
```
