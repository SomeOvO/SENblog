---
title: 三角洲官网扫码登陆获取openid与access_token
published: 2026-01-31
description: 最新的教程，使用Go重写
image: ""
tags:
  - 开发日记
category: 我得
draft: false
lang: zh-CN
---
在之前，我写了一篇[教程](https://blog.sakurasen.cn/post/20250828025102/)，使用的语言为JS。

现在使用Go根据最新方法重写一次：

# 获取登录二维码

通过访问

https://xui.ptlogin2.qq.com/ssl/ptqrshow?appid=716027609&e=2&l=M&s=3&d=72&v=4&t=0.8668949625611146&daid=383&pt_3rd_aid=101491592&u1=https%3A%2F%2Fgraph.qq.com%2Foauth2.0%2Flogin_jump

获取Cookie中的Qrsig

:::warning
loger.Loger 为 zap 中的Loger，你可以改为fmt或panic
:::

```go
  func GetQr(retry int) (qrs string) {

    if retry > 2 {

        loger.Loger.Fatal("[获取登录二维码]获取登录二维码失败，已达到最大错误数" + strconv.Itoa(retry))

    }

    url := "https://xui.ptlogin2.qq.com/ssl/ptqrshow?appid=716027609&e=2&l=M&s=3&d=72&v=4&t=0.8668949625611146&daid=383&pt_3rd_aid=101491592&u1=https%3A%2F%2Fgraph.qq.com%2Foauth2.0%2Flogin_jump"

    resp, err := http.Get(url)

    if err != nil {

        loger.Loger.Error("[获取登录二维码]获取登录二维码失败，重正在重试")

        retry++

        time.Sleep(3 * time.Second)

        GetQr(retry)

    }

    RespCookies := resp.Cookies()

    var qrsig string

    for _, c := range RespCookies {

        if c.Name == "qrsig" {

            qrsig = c.Value

        }

    }

    if qrsig == "" {

        loger.Loger.Error("[获取登录二维码]获取获取QrSig失败，正在重试")

        retry++

        time.Sleep(3 * time.Second)

        GetQr(retry)

    }

    data, err := io.ReadAll(resp.Body)

    if err != nil {

        loger.Loger.Fatal("获取二维码图片失败")

    }

    workdir, err := os.Getwd()

    if err != nil {

        loger.Loger.Fatal("[获取登录二维码]获取当前目录失败", zap.Error(err))

    }

    err = os.WriteFile(workdir+"/qr.png", data, 0775)

    if err != nil {

        loger.Loger.Fatal("[获取登录二维码]创建二维码图片失败", zap.Error(err))

    }

    return qrsig

}
```

这个函数通过解析cookie，返回qrsig，在

获取到qrsig我们要轮询

在轮询之前，我们需要一个工具函数来提供 Hash33加密：

```go
  func Hash33(str string, seed int32) int32 {

    hash := seed

    for i := range str {

        hash += (hash << 5) + int32(str[i])

    }

    hash &= 0x7fffffff

    return hash

}
```

然后创建轮询：

```go
func Check(qrsig string) {

    client := http.Client{}

    rawurl, err := url.Parse("https://xui.ptlogin2.qq.com/ssl/ptqrlogin")

  

    if err != nil {

        loger.Loger.Fatal("[轮询登录状态]创建Http请求失败")

    }

  

    url := rawurl.Query()

    url.Set("ptqrtoken", strconv.Itoa(int(Hash33(qrsig, 0))))

    url.Set("login_sig", qrsig)

    url.Set("aid", "716027609")

    url.Set("daid", "383")

    url.Set("pt_3rd_aid", "101491592")

    url.Set("has_onekey", "1")

    url.Set("js_ver", "25112611")

    url.Set("ptredirect", "0")

    url.Set("pt_js_version", "42f2bcc1")

    url.Set("ptlang", "2052")

    url.Set("h", "0")

    url.Set("t", "1")

    url.Set("g", "1")

    url.Set("from_ui", "1")

    url.Set("u1", "https://graph.qq.com/oauth2.0/login_jump")

    rawurl.RawQuery = url.Encode()

  

    req, err := http.NewRequest("GET", rawurl.String(), nil)

    req.Header.Set("cookie", "qrsig="+qrsig)

    req.Header.Set("referer", "https://xui.ptlogin2.qq.com/cgi-bin/xlogin?appid=716027609&daid=383&style=33&login_text=%E7%99%BB%E5%BD%95&hide_title_bar=1&hide_border=1&target=self&s_url=https%3A%2F%2Fgraph.qq.com%2Foauth2.0%2Flogin_jump&pt_3rd_aid=101491592&pt_feedback_link=https%3A%2F%2Fsupport.qq.com%2Fproducts%2F77942%3FcustomInfo%3Dmilo.qq.com.appid101491592&theme=2&verify_theme=")

    rep, err := client.Do(req)

    if err != nil {

        loger.Loger.Fatal("[轮询登录状态]发送请求失败")

    }

  

    data, err := io.ReadAll(rep.Body)

    if err != nil {

        loger.Loger.Fatal("[轮询登录状态]获取Body失败")

    }

    arr := strings.Split(string(data), "'")

    fmt.Println(arr[9])

    if arr[2] == "0" {

        cks := rep.Cookies()

        GetPsKey(arr[5], cks)

        return

    }

    time.Sleep(1 * time.Second)

    Check(qrsig)

}
```

此接口只会返回类似格式的TEXT：

`ptuiCB('66','0','','0','二维码未失效。', '')`

如果扫描后第三个 ' ' 会被替换为可请求的链接，我们需要访问这个链接获取**p_skey**，而且还不能跟随重定向。

```go
func GetPsKey(url string, cookies []*http.Cookie) (p_skey string, cookie []*http.Cookie) {

    client := http.Client{

        CheckRedirect: (func(req *http.Request, via []*http.Request) error {

            return http.ErrUseLastResponse

        }),

    }

    req, err := http.NewRequest("GET", url, nil)

    if err != nil {

        loger.Loger.Fatal("[获取Ptkey]创建请求失败", zap.Error(err))

    }

    for _, c := range cookies {

        req.AddCookie(c)

    }

  

    resp, err := client.Do(req)

    if err != nil {

        loger.Loger.Fatal("[获取Ptkey]请求失败", zap.Error(err))

    }

    cks := resp.Cookies()

    var pskey string

    for _, c := range cks {

        if c.Name == "p_skey" && c.Value != "" {

            pskey = c.Value

        }

    }

    if pskey == "" {

        loger.Loger.Fatal("[获取Ptkey]无法解析PSKEY")

    }

    return pskey, resp.Cookies()

}
```

我们需要获得pskey和cookie进行下一步，获取CODE

其中 gtk通过调用函数`Hash33`获得，其**str**为**pskey** ， **seed** 为 **5381**

```go
func GetCode(gtk int32, cks []*http.Cookie) (code string) {

    requrl := "https://graph.qq.com/oauth2.0/authorize"

  

    formData := url.Values{}

    formData.Set("response_type", "code")

    formData.Set("client_id", "101491592")

    formData.Set("redirect_uri", "https://milo.qq.com/comm-htdocs/login/qc_redirect.html?parent_domain=https://df.qq.com&isMiloSDK=1&isPc=1")

    formData.Set("scope", "")

    formData.Set("state", "STATE")

    formData.Set("switch", "")

    formData.Set("form_ptlogin", "1")

    formData.Set("src", "1")

    formData.Set("update_auth", "1")

    formData.Set("openapi", "1010")

    formData.Set("auth_time", strconv.Itoa(int(time.Now().UnixMilli())))

    formData.Set("ui", "2C0B3209-A5BA-46E3-88EC-3C7D0FA2C6AE")

    formData.Set("g_tk", strconv.Itoa(int(gtk)))

    req, err := http.NewRequest("POST", requrl, bytes.NewReader([]byte(formData.Encode())))

    if err != nil {

        loger.Loger.Fatal("[获取CODE]创建请求失败", zap.Error(err))

    }

    for _, c := range cks {

        req.AddCookie(c)

    }

    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")

    req.Header.Set("referer", "https://xui.ptlogin2.qq.com/")

    client := http.Client{

        CheckRedirect: func(req *http.Request, via []*http.Request) error {

            return http.ErrUseLastResponse

        },

    }

    resp, err := client.Do(req)

    if err != nil {

        loger.Loger.Fatal("[获取CODE]请求失败", zap.Error(err))

    }

    localton := resp.Header.Get("Location")

    qrcode := strings.Split(localton, "code=")

    finalqrcode := strings.Split(qrcode[1], "&")

    return finalqrcode[0]

}
```

此code用来交换最终的openid与accesstoken：

```go
func GetOA(code string) (openid string, token string) {

    client := http.Client{}

    url := "https://ams.game.qq.com/ams/userLoginSvr"

    par := "?a=qcCodeToOpenId&appid=101491592&redirect_uri=https://milo.qq.com/comm-htdocs/login/qc_redirect.html&callback=miloJsonpCb_80466&qc_code=" + code + "&_=" + strconv.Itoa(int(time.Now().UnixMilli()))

    resq, err := http.NewRequest("GET", url+par, nil)

    if err != nil {

        loger.Loger.Fatal("[获取OA]创建请求失败")

    }

    resq.Header.Set("referer", "https://df.qq.com/")

    resp, err := client.Do(resq)

    if err != nil {

        loger.Loger.Fatal("[获取OA]请求失败")

    }

    data, err := io.ReadAll(resp.Body)

    if err != nil {

        if err != nil {

            loger.Loger.Fatal("[获取OA]获取返回信息失败")

        }

    }

    a := string(data)

    b := strings.Split(a, `"openid":"`)

    c := strings.Split(b[1], `","access_token":"`)

    d := strings.Split(c[1], `"`)

    return c[0], d[0]

}
```

至此，已完结