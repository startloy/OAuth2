# Integrating many third party login interfaces, including qq-login、wx-login、sina-login、github-login、alipay-login and so on

# 通用第三方登录说明文档

* 微信
* QQ
* 微博
* 支付宝
* GitHub
* Facebook
* Twitter
* Line
* Google

### 安装

```
composer require tinymeng/oauth
```

> 类库使用的命名空间为`\\tinymeng\\oauth`

### 目录结构

```
.
├── README.md                        说明文件
├── composer.json                    composer文件
├── src                              代码源文件目录
│   ├── Connector
│   │   ├── Gateway.php              必须继承的抽象类
│   │   └── GatewayInterface.php     必须实现的接口
│   ├── Gateways
│   │   ├── Alipay.php
│   │   ├── Facebook.php
│   │   ├── Google.php
│   │   ├── Line.php
│   │   ├── Qq.php
│   │   ├── Twitter.php
│   │   ├── Weibo.php
│   │   └── Weixin.php
│   ├── Helper
│   │   └── Str.php                  字符串辅助类
│   └── OAuth.php                    抽象实例类
└── wx_proxy.php                     微信多域名代理文件
```

### 公共方法

在接口文件中，定义了4个方法，是每个第三方基类都必须实现的，用于相关的第三方登录操作和获取数据。方法名如下：

```php

    /**
     * Description:  得到跳转地址
     * @author: JiaMeng <666@majiameng.com>
     * Updater:
     * @return mixed
     */
    public function getRedirectUrl();

    /**
     * Description:  获取当前授权用户的openid标识
     * @author: JiaMeng <666@majiameng.com>
     * Updater:
     * @return mixed
     */
    public function openid();

    /**
     * Description:  获取格式化后的用户信息
     * @author: JiaMeng <666@majiameng.com>
     * Updater:
     * @return mixed
     */
    public function userInfo();

    /**
     * Description:  获取原始接口返回的用户信息
     * @author: JiaMeng <666@majiameng.com>
     * Updater:
     * @return mixed
     */
    public function getUserInfo();
    
```

微信有一个额外的方法，用于获取代理请求的地址

```php
    /**
     * 获取中转代理地址
     */
    public function getProxyURL();
```

### 典型用法

以ThinkPHP5为例

```php
<?php

namespace app\mobile\controller;

use tinymeng\OAuth2\OAuth;
use think\Config;

class Sns
{
    private $config;

    /**
     * 第三方登录，执行跳转操作
     *
     * @param string $name 第三方渠道名称，目前可用的为：weixin,qq,weibo,alipay,facebook,twitter,line,google
     */
    public function login($name)
    {
        //获取配置
        $this->config = Config::get('sns.' . $name);

        //设置回跳地址
        $this->config['callback'] = $this->makeCallback($name);

        //可以设置代理服务器，一般用于调试国外平台
        $this->config['proxy'] = 'http://127.0.0.1:1080';

        /**
         * 对于微博，如果登录界面要适用于手机，则需要设定->setDisplay('mobile')
         *
         * 对于微信，如果是公众号登录，则需要设定->setDisplay('mobile')，否则是WEB网站扫码登录
         *
         * 其他登录渠道的这个设置没有任何影响，为了统一，可以都写上
         */
        return redirect(OAuth::$name($this->config)->setDisplay('mobile')->getRedirectUrl());

        /**
         * 如果需要微信代理登录，则需要：
         *
         * 1.将wx_proxy.php放置在微信公众号设定的回调域名某个地址，如 http://www.abc.com/proxy/wx_proxy.php
         * 2.config中加入配置参数proxy_url，地址为 http://www.abc.com/proxy/wx_proxy.php
         *
         * 然后获取跳转地址方法是getProxyURL，如下所示
         */
        $this->config['proxy_url'] = 'http://www.abc.com/proxy/wx_proxy.php';
        return redirect(OAuth::$name($this->config)->setDisplay('mobile')->getProxyURL());
    }

    public function callback($name)
    {
        //获取配置
        $this->config = Config::get('sns.' . $name);

        //设置回跳地址
        $this->config['callback'] = $this->makeCallback($name);

        //获取格式化后的第三方用户信息
        $snsInfo = OAuth::$name($this->config)->userinfo();

        //获取第三方返回的原始用户信息
        $snsInfoRaw = OAuth::$name($this->config)->getUserInfo();

        //获取第三方openid
        $openid = OAuth::$name($this->config)->openid();
    }

    /**
     * 生成回跳地址
     *
     * @return string
     */
    private function makeCallback($name)
    {
        //注意需要生成完整的带http的地址
        return url('/sns/callback/' . $name, '', 'html', true);
    }
}
```

2.0版本不再通过系统自动设置state，如有需要请自行处理验证，state也放入config里即可

Line和Facebook强制要求传递state，如果你没有设置，则会传递随机值

如果要验证state，则在获取用户信息的时候要加上`->mustCheckState()`方法。

```php
$snsInfo = OAuth::$name($this->config)->mustCheckState()->userinfo();
```

> 注意，不是所有的平台都支持传递state，请自行阅读官方文档

### 配置文件样例

#### 1.微信

> 微信会返回特有的unionid字段

```
'weixin'=>[
    'pc'=>[
        'app_id' => 'wx52e2b2464*****',
        'app_secret' => 'd5dad705a1159d*********',
        'callback' => 'http://i.959.cn/qq-login.php',
        'scope'      => 'snsapi_login',//扫码登录
    ],
    'mobile'=>[
        'app_id' => 'wx6ca7410f8******',
        'app_secret' => '30a206b87b7689b19f11******',
        'callback' => 'http://i.959.cn/qq-login.php',
        'scope'      => 'snsapi_userinfo',//静默授权=>snsapi_base;获取用户信息=>snsapi_userinfo
    ],
]
```

#### 2.QQ

```
'qq'=>[
    'app_id'        => '1014*****',
    'app_secret'    => '8a2b322610d7a0d****',
    'scope'         => 'get_user_info',
    'callback' => 'http://majiameng.com/login/qq',
    'withUnionid' => true //已申请unioid打通
]
```
QQ现在可以获取`unionid`了，详见: http://wiki.connect.qq.com/unionid%E4%BB%8B%E7%BB%8D
只需要配置参数`$config['withUnionid'] = true`，默认不会请求获取Unionid

#### 3.微博

```
'app_id'     => '78734****',
'app_secret' => 'd8a00617469018d61c**********',
'scope'      => 'all',
```

#### 4.支付宝

```
'app_id'      => '2016052*******',
'scope'       => 'auth_user',
'pem_private' => Env::get('ROOT_PATH') . 'pem/private.pem', // 你的私钥
'pem_public'  => Env::get('ROOT_PATH') . 'pem/public.pem', // 支付宝公钥
```

#### 5.Facebook

```
'app_id'     => '2774925********',
'app_secret' => '99bfc8ad35544d7***********',
'scope'      => 'public_profile,user_gender',//user_gender需要审核，所以不一定能获取到
```

facebook有个特殊的配置`$config['field']`，默认是`'id,name,gender,picture.width(400)'`，你可以根据需求参考官方文档自行选择要获取的用户信息

#### 6.Twitter

```
'app_id'     => '3nHCxZgcK1WpYV**********',
'app_secret' => '2byVAPayMrG8LISjopwIMcJGy***************',
```

#### 7.Line

```
'app_id'     => '159******',
'app_secret' => '1f19c98a61d148f2************',
'scope'      => 'profile',
```

#### 8.Google

```
'app_id'     => '7682717*******************.apps.googleusercontent.com',
'app_secret' => 'w0Kq-aYA***************',
'scope'      => 'https://www.googleapis.com/auth/userinfo.profile',
```

### 返回样例

```
Array
(
    [openid] => 1047776979*******
    [channel] => google
    [nick] => Coeus Rowe
    [gender] => m //twitter和line不会返回性别，所以这里是n，Facebook根据你的权限，可能也不会返回，所以也可能是n
    [avatar] => https://lh6.googleusercontent.com/-iLps1iAjL8Q/AAAAAAAAAAI/AAAAAAAAAAA/Bu5l0EIquF0/photo.jpg
)
```
