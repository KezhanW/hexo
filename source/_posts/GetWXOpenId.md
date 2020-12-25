---
title: 获取微信小程序OpenId
date: 2020-12-25 13:22:01
tags:
cover: https://s3.ax1x.com/2020/12/25/rWSFij.jpg
copyright_author_href: https://github.com/KezhanW
copyright_url: https://kezhanw.github.io/2020/12/25/GetWXOpenId
---
## 基本参数
- 获取`appid`及`secret`
- 获取登录凭证code
***
登陆你的微信小程序在开发设置里面可以直接看到appid和secret
[![rRzDMV.jpg](https://s3.ax1x.com/2020/12/25/rRzDMV.jpg)](https://imgchr.com/i/rRzDMV)
获取登录凭证
``` js
			//获取openid
			uni.login({
				success: res => {
					var code=res.code;//登录凭证
					 if(code) {
						 uni.request({
						 	url: that.apiRoot + '/Fly_Fp/Fly_Fp_ApplyFor/getuseropenid',
						 	method: 'POST',
						 	data: { ...that.auth,
						 		data: res.code
						 	},
						 	success: (res) => {
						 		//openid
						 		var openId = res.data;
						 	}
						 })
					}
				}
			});
```
## 请求微信API
```
                /// <summary>
                /// 获取openid
                /// </summary>
                /// <param name="code">登录凭证</param>
                /// <param name="iv"></param>
                /// <param name="encryptedData"></param>
                /// <returns></returns>
                public string ProcessRequest(string code, string iv, string encryptedData)
                {
                    string Appid = "wx****************";
                    string Secret = "****************************";
                    string grant_type = "authorization_code";
        
                    //向微信服务端 使用登录凭证 code 获取 session_key 和 openid  
                    string url = "https://api.weixin.qq.com/sns/jscode2session?appid=" + Appid + "&secret=" + Secret + "&js_code=" + code + "&grant_type=" + grant_type;
                    string type = "utf-8";
                    var resData = this.GetUrltoHtml(url, type);
                    return resData;
                }
```

```
                /// <summary> 
                /// 获取链接返回数据 
                /// </summary> 
                /// <param name="Url">链接</param> 
                /// <param name="type">请求类型</param> 
                /// <returns></returns> 
                public string GetUrltoHtml(string Url, string type)
                {
                    try
                    {
                        System.Net.WebRequest wReq = System.Net.WebRequest.Create(Url);
                        // Get the response instance. 
                        System.Net.WebResponse wResp = wReq.GetResponse();
                        System.IO.Stream respStream = wResp.GetResponseStream();
                        // Dim reader As StreamReader = New StreamReader(respStream) 
                        using (System.IO.StreamReader reader = new System.IO.StreamReader(respStream, Encoding.GetEncoding(type)))
                        {
                            return reader.ReadToEnd();
                        }
                    }
                    catch (System.Exception ex)
                    {
                        return ex.Message;
                    }
                }
```