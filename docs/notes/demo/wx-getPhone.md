---
title: 微信小程序--获取手机号功能
tags:
  - 随笔
createTime: 2025/03/19 22:38:48
permalink: /article/jwdvn3lh/
---

微信从基础库2.21.2开始，采用了新的获取用户手机号方式，本笔记适用于新方式。

## 一、前端

依据微信小程序开发文档描述，[小程序官方文档](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/getPhoneNumber.html)，先写一个button按钮，`open-type` 设为 `getPhoneNumber`，在uniapp页面中创建按钮：

```vue
<button open-type="getPhoneNumber" @getphonenumber="onGetPhoneNumber">唤起授权</button>
```

::: tip 提示
以下小程序官方的写法，但我们在uniapp想要触发按钮点击，并进行后续操作的话，建议使用上面的写法。

```vue
<button open-type="getPhoneNumber" bindgetphonenumber="getPhoneNumber"></button>
```

:::

接下来，我们需要写按钮的点击事件处理。以UniAPP为例。微信官方的写法，可以参考[官方文档](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/getPhoneNumber.html)。

我们的需要：将 `bindgetphonenumber` 事件回调中的动态令牌`code`传到开发者后台，并在开发者后台调用微信后台提供的 [phonenumber.getPhoneNumber](https://developers.weixin.qq.com/miniprogram/dev/api-backend/open-api/phonenumber/phonenumber.getPhoneNumber.html) 接口，消费`code`来换取用户手机号。每个`code`有效期为5分钟，且只能消费一次。

```javascript
methods:{  
    onGetPhoneNumber(e){  
        if(e.detail.errMsg=="getPhoneNumber:fail user deny"){       //用户决绝授权  

            //拒绝授权后弹出一些提示  

        }else{      //允许授权  
            uni.request({  
                url: 'https://www.example.com/',        //请以你的后端接口为准  
                method:'POST',  
                data: {  
                    iv:e.detail.iv,  
                    encryptedData: e.detail.encryptedData ,             
                    session:this.session_key,         
                },  
                success: (res) => {  
                    console.log(res.data)       //res.data 即为后端返回的解密数据  
                }  
            });  

        }  
    }  
}
```

::: info 注意

`getPhoneNumber` 返回的 `code` 与 `wx.login` 返回的 `code` 作用是不一样的，不能混用。

:::

## 二、后端

### 2.1 获取Access_Token

Access_Token是小程序全局唯一后台接口调用凭据，也是后端通过前端传进来的code获取到用户手机号的必须参数。根据文档：token有效期为7200s，开发者需要进行妥善保存。

**请求地址**

```
GET https://api.weixin.qq.com/cgi-bin/token
```

**请求参数**

| 属性       | 类型   | 必填 | 说明                                                         |
| :--------- | :----- | :--- | :----------------------------------------------------------- |
| grant_type | string | 是   | 填写常量： client_credential                                 |
| appid      | string | 是   | 小程序唯一凭证，即 AppID，可在「微信公众平台 - 设置 - 开发设置」页中获得。（需要已经成为开发者，且帐号没有异常状态） |
| secret     | string | 是   | 小程序唯一凭证密钥，即 AppSecret，获取方式同 appid           |

**返回参数**

| 属性         | 类型   | 说明                                           |
| :----------- | :----- | :--------------------------------------------- |
| access_token | string | 获取到的凭证                                   |
| expires_in   | number | 凭证有效时间，单位：秒。目前是7200秒之内的值。 |

示例Spring Boot函数：

```java
    @Scheduled(fixedRate = 7200000) // 每隔7200秒运行一次
    public void getAccessToken() {
        String appid = "your_appid"; // 你的小程序AppID
        String secret = "your_secret"; // 你的小程序AppSecret
        String grantType = "client_credential";
        // 构造请求URL
        String url = String.format("https://api.weixin.qq.com/cgi-bin/token?grant_type=%s&appid=%s&secret=%s",
                grantType, appid, secret);

        // 使用hutool发送GET请求并获取响应
        try {
            HttpResponse response = HttpRequest.get(url).execute();
            if (response.isOk()) {
                ...
 				//使用你的项目的Json解析库来解析结果
            } else {
                System.out.println("Failed to get access token. Status code: " + response.getStatus());
            }
        } catch (Exception e) {
            System.out.println("Error occurred while getting access token: " + e.getMessage());
        }
    }
```

### 2.2 换取真实数据

这部分[官方文档](https://developers.weixin.qq.com/miniprogram/dev/OpenApiDoc/user-info/phone-number/getPhoneNumber.html)写得非常完备，基础HTTP请求过程也可以参考2.1，因此可以不再赘述。
