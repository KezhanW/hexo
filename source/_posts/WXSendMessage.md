---
title: 微信小程序给用户发送消息
date: 2020-12-25 18:26:45
tags:
cover: https://s3.ax1x.com/2020/12/25/rW4nJg.jpg
copyright_author_href: https://github.com/KezhanW
copyright_url: https://kezhanw.github.io/2020/12/25/WXSendMessage
---
## 基本参数
- 用户的`openid`
- `access_token`
- 公众号和小程序的`appid`
- 模板ID
***
`openid`获取方法之前的文章有提到过
公众号和小程序的`appid`和模板ID都可以在[微信公众号平台](https://mp.weixin.qq.com)获取到
[![rfVgSO.png](https://s3.ax1x.com/2020/12/25/rfVgSO.png)](https://imgchr.com/i/rfVgSO)

下面我们就讲讲`access_token`的获取方式
将AppID和AppSecret(小程序密钥)调用接口即可
注意：`access_token`是有时间限制的最好存到缓存里
```
        /// <summary>
        /// 获取accessToken
        /// </summary>
        /// <returns></returns>
        public static string JsCode2Session()
        {
            string appid = "wx************";
            string secret = "************************";
            string JsCode2SessionUrl = "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid={0}&secret={1}";
            var url = string.Format(JsCode2SessionUrl, appid, secret);
            var str = GetFunction(url);
            try
            {
                JObject jo = (JObject)JsonConvert.DeserializeObject(str);
                string access_token = jo["access_token"].ToString();
                return access_token;
            }
            catch (Exception ex)
            {
                return "";
            }
        }
```

```
        public static string GetFunction(string url)
        {
            string serviceAddress = url;
            HttpWebRequest request = (HttpWebRequest)WebRequest.Create(serviceAddress);
            request.Method = "GET";
            request.ContentType = "textml;charset=UTF-8";
            HttpWebResponse response = (HttpWebResponse)request.GetResponse();
            Stream myResponseStream = response.GetResponseStream();
            StreamReader myStreamReader = new StreamReader(myResponseStream, Encoding.UTF8);
            string retString = myStreamReader.ReadToEnd();
            myStreamReader.Close();
            myResponseStream.Close();
            //Response.Write(retString);
            return retString;
        }
```

## 发送消息

首先找到该模板找到详细信息里面分别对应的参数`first` `keyword1` `keyword2` `keyword3`
备注有的话就显示没有的话就不显示
注意：详细信息前面的标语是不变的所以找模板的时候要找对应的
[![rfei5t.png](https://s3.ax1x.com/2020/12/25/rfei5t.png)](https://imgchr.com/i/rfei5t)

第一个appid是公众号的
第二个是微信小程序的
因为微信之前取消了小程序发送消息现在只基于公众号所以用户也必须关注该公众号

```
                /// <summary>
                /// 开票内容的退回并发送信息
                /// </summary>
                /// <param name="ID"></param>
                /// <param name="yuanyi"></param>
                /// <param name="zhuID"></param>
                /// <returns></returns>
                [HttpPost]
                public string approvalcontent(string openid, string createdate, string createUsername, string Reason)
                {
                    var accesstoken = fly_Fp_ApplyForIBLL.JsCode2Session();
                    string json = "{\"touser\": \"" + openid + "\", " +
                                  "\"mp_template_msg\": { " +
                                  "\"appid\": \"wx************\"," +
                                  "\"template_id\": \"X82CtoGmegLQpGZAV6lDpT3uBcVA7vZNJhx52fwVBpg\"," +
                                  "\"url\": \"http://weixin.qq.com/download\"," +
                                  "\"miniprogram\": { " +
                                  "\"appid\": \"wx**************\"," +
                                  "\"pagepath\": \"pages/login\" }," +
                                  "\"data\": {" +
                                  "\"first\": { \"value\": \"审批退回！\"}," +
                                  "\"keyword1\": { \"value\": \"开票内容\", \"color\": \"#173177\"}," +
                                  "\"keyword2\": { \"value\": \"" + createdate + "\",\"color\": \"#173177\"}," +
                                  "\"keyword3\": { \"value\": \"" + createUsername + "\", \"color\": \"#173177\"}," +
                                  "\"remark\": { \"value\": \"" + Reason + "\",\"color\": \"#173177\"}}}}";
                    return wxapi.SendTemplateMsg(json, accesstoken);
                }
```

```
        /// <summary>
        /// 发送模板消息
        /// </summary>
        /// <param name="accessToken">AccessToken</param>
        /// <param name="data">发送的模板数据</param>
        /// <returns></returns>
        public static string SendTemplateMsg(string data,string accesstoken)
        {
            string url = string.Format("https://api.weixin.qq.com/cgi-bin/message/wxopen/template/uniform_send?access_token={0}", accesstoken);
            HttpWebRequest hwr = System.Net.WebRequest.Create(url) as HttpWebRequest;
            hwr.Method = "POST";
            hwr.ContentType = "application/x-www-form-urlencoded";
            byte[] payload;
            payload = System.Text.Encoding.UTF8.GetBytes(data); //通过UTF-8编码
            hwr.ContentLength = payload.Length;
            Stream writer = hwr.GetRequestStream();
            writer.Write(payload, 0, payload.Length);
            writer.Close();
            var result = hwr.GetResponse() as HttpWebResponse; //此句是获得上面URl返回的数据
            string strMsg = WebResponseGet(result);
            //如果token失效则重新获取
            if (strMsg.IndexOf("42001") != -1) {
                string token = wxapi.JsCode2Session();
                return SendTemplateMsg(data, token);
            }
            return strMsg;
        }
```

```
        /// <summary>
        /// 获取Response
        /// </summary>
        /// <param name="webResponse"></param>
        /// <returns></returns>
        public static string WebResponseGet(HttpWebResponse webResponse)
        {
            StreamReader responseReader = null;
            string responseData = "";
            try
            {
                responseReader = new StreamReader(webResponse.GetResponseStream());
                responseData = responseReader.ReadToEnd();
            }
            catch
            {
                throw;
            }
            finally
            {
                webResponse.GetResponseStream().Close();
                responseReader.Close();
                responseReader = null;
            }
            return responseData;
        }
```
