支付系统 API 文档 (版本 1.0-snapshot)
================

***注意这不是稳定版本，部分内容可能会更改，部分功能还未实现，具体详情咨询 [风信子团队](mailto:team-hyacinth@fun.tv?subject=[BESTPAY]支付系统咨询)***

* [1.名词解释](#noun)
* [2.协议规则](#protocol)
* [3.安全规范](#security)
    * [3.1.签名算法](#security_sign)
* [4.对外接口](#interface)
    * [4.1.创建订单接口](#interface_create_order)
    * [4.2.支付接口](#interface_create_pay)
    * [4.3.创建订单并支付接口](#interface_create_order_and_pay)
    * [4.4.支付结果通知](#interface_notify_biz_sys)
    * [4.5.查询账号余额接口](#interface_balance_query)
    * [4.6.订单查询接口接口](#interface_order_query)
* [5.枚举类型参数取值](#enum)
    * [5.1.订单类型取值](#enum_order_type)
    * [5.2.账号类型取值](#enum_account_type)
    * [5.3.订单状态取值](#enum_order_status)
    * [5.4.支付状态取值](#enum_trade_status)
    * [5.5.订单消费状态取值](#enum_consume_status)
    * [5.6.网关具体取值](#enum_gateway_code)
    * [5.7.支付场景具体取值](#enum_pay_scene)
      * [5.7.1.支付网关支持的支付场景](#enum_pay_scene_gateway)
* [6.错误码](#error_code)

<h2 id="noun">1.名词解释</h2>

***支付系统***

* 代表本系统，负责对接微信或支付宝相关支付网关，统一完成支付过程

***网关***

* 在本文档中代表支付系统接入的三方支付平台，如：微信，支付宝

***业务系统***

* 使用支付系统完成支付的其他系统
  
<h2 id="protocol">2.协议规则</h2>

***传输方式***

* 采用 HTTP 传输

***签名算法***

* MD5

***URL Encoding***

* 根据 HTTP 协议要求，传递参数的值中如果存在特殊字符（如：&，@ 等），那么该值需要做 URL Encoding(UTF-8)，这样请求接收方才能接收到正确的参数值

***交易金额***
  
* 交易金额默认为【人民币】交易，接口中参数支付金额单位为 【分】，参数值不能带小数。
* 对账单中的交易金额单位为【元】。

***时间戳***
 
* 标准北京时间，时区为东八区，自1970年1月1日 0点0分0秒以来的【秒数】。
* 注意：部分系统取到的值为毫秒级，需要转换成秒（10位数字）。

***nonce_str***

* 现时（nonce）是用于防御回放攻击（Replay attack）和选择明文攻击（Chosen-plaintext attacks）的参数，不允许重复。
* 我们推荐生成随机数算法如下：时间戳 + 随机数。

<h2 id="protocol">3.安全规范</h2>
<h3 id="security_sign">3.1.签名算法</h3>

* **第一步**，设所有发送或者接收到的数据为集合M，将集合M内非空参数值的参数按照参数名ASCII码从小到大排序（字典序），使用URL键值对的格式（即key1=value1&key2=value2…）拼接成字符串stringA。特别注意以下重要规则：
      
    * 参数名 ASCII 码从小到大排序（字典序）
    * 如果参数的值为空不参与签名
    * 参数名区分大小写
    * 如果参数值在传输过程中做了 URL Encoding (charset=UTF-8)，待签名的数据应该是原生值而不是 encoding 之后的值。
    * 验证调用返回或系统主动通知签名时，传送的 sign 参数不参与签名，将生成的签名与该 sign 值作校验。
    * 接口可能增加字段，验证签名时必须支持增加的扩展字段
      
* **第二步**，在 stringA 最后拼接上 key=(API密钥的值) 得到 stringSignTemp 字符串，并对 stringSignTemp 进行 MD5 运算，再将得到的字符串所有字符转换为大写，得到 sign 值 signValue。

* 举例（使用伪代码说明）:

假设传送的参数如下:

```
    nonce_str        = 19LQ68TII127
    account_type     = funshion
    account_id       = jiangxu
    api_service_code = balance_query
    v                = 1.0
    client_code      = game
```

第一步：对参数按照key=value的格式，并按照参数名ASCII字典序排序如下:

```
    stringA = "account_id=jiangxu&account_type=funshion&api_service_code=balance_query&client_code=game&nonce_str=19LQ68TII127&v=1.0";
```

第二步：拼接API密钥:

```
    stringSignTemp = stringA + "&key=987654321";
    signValue = DigestUtils.md5(stringSignTemp).toUpperCase();
    
    Assert.assertEquals("3F8A4B77ACC01379714AEC1A37BAB774", signValue);
```

<h2 id="interface">4.对外接口</h2>
* 假设提供服务的URL为 **http://pay.fun.tv/trade/...**
* 由于支付系统支持接入多套账号系统，所以必须通过 [账号类型](#enum_account_type) （ *account_type* ）和此类型下的 账号ID （ *account_id* ）唯一确定一个账号

<h3 id="interface_create_order">4.1.创建订单接口</h3>
* URL: **http://pay.fun.tv/trade/api**
* 请求方式: GET
* 说明:

  * 在支付系统创建一条订单记录，返回支付系统的订单ID（ *order_id* ）
  * 当订单类型为 充值（`recharge`）和 充消（`recharge_consume`）时，相当于预先下好一个订单，并不会发起支付行为，发起支付行为要靠 [支付接口](#interface_create_pay)
  * 当订单类型为 消费（`consume`）时，单纯从用户余额中扣除费用 *consume_fee* ，立即生效

  
* 请求参数:

<table border="1" width="100%" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >参数</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>api_service_code</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>接口代码，固定值: <code>create_order</code> </td>
  </tr>
  <tr>
    <td>v</td>
    <td align="center" >是</td>
    <td>String(6)</td>
    <td>接口版本号，固定值: <code>1.0</code> </td>
  </tr>
  <tr>
    <td>client_code</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>授权可以调用接口的业务方代码。需要提前在 <a href="mailto:team-hyacinth@fun.tv?subject=[BESTPAY]申请接入支付系统">风信子团队</a> 申请</td>
  </tr>
  <tr>
    <td>nonce_str</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>用于防御回放攻击的，参考 <a href="#protocol">协议规则</a></td>
  </tr>
  <tr>
    <td>sign</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>签名，参考 <a href="#security_sign">签名算法</a></td>
  </tr>
  <tr>
    <td>type</td>
    <td align="center" >是</td>
    <td>String(16)</td>
    <td>订单类型，参考 <a href="#enum_order_type">订单类型取值</a></td>
  </tr>
  <tr>
    <td>biz_trade_no</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>
      对应业务系统（参考 <a href="#noun">名词解释</a>）的唯一编号<br>
      规则为:大写字母、小写字母与数字的组合
    </td>
  </tr>
  <tr>
    <td>account_type</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>账号类型，参考 <a href="#enum_account_type">账号类型取值</a></td>
  </tr>
  <tr>
    <td>account_id</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td> <b>account_type</b> 标识的类型下的账号ID</td>
  </tr>
  <tr>
    <td>item_name</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>购买商品名称</td>
  </tr>
  <tr>
    <td>description</td>
    <td align="center" >否</td>
    <td>String(512)</td>
    <td>对一笔交易的具体描述信息</td>
  </tr>
  <tr>
    <td>callback_url</td>
    <td align="center" >是</td>
    <td>String(1024)</td>
    <td>订单完成之后通知业务方的地址</td>
  </tr>
  <tr>
    <td>return_url</td>
    <td align="center" >否</td>
    <td>String(1024)</td>
    <td>用户支付完订单后跳转的地址</td>
  </tr>
  <tr>
    <td>consume_fee</td>
    <td align="center" >是</td>
    <td>Long</td>
    <td>用户消费掉的余额，单位为分。纯充值操作时传 0</td>
  </tr>
</table>

* 返回结果：

  * 返回结果为 json 格式，字段说明：

<table border="1" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >字段</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>return_code</td>
    <td align="center" >是</td>
    <td>String(16)</td>
    <td>
      返回状态码:
      <ul>
        <li><code>SUCCESS</code> - 成功</li>
        <li>其他情况为具体错误代码，参考 <a href="#error_code">错误码</a> </li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>return_msg</td>
    <td align="center" >否</td>
    <td>String(128)</td>
    <td>返回信息，成功时内容为空，失败时返回错误原因</td>
  </tr>
  <tr>
    <td>order_id</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>成功时返回支付系统的订单ID</td>
  </tr>
</table>  
      
成功的例子:

```
  {
    "return_code" : "SUCCESS",
    "return_msg" : "",
    "order_id" : "19LJ1V1IR2B2zuan"
  }
```

失败的例子:

```
  {
    "return_code" : "SIGN_ERROR",
    "return_msg" : "签名失败" 
  }
```
  
<h3 id="interface_create_pay">4.2.支付接口</h3>
* URL: **http://pay.fun.tv/trade/api**
* 请求方式: GET
* 说明:

  * 对 [4.1.创建订单接口](#interface_create_order) 创建的订单进行具体的支付行为
  
* 请求参数:

<table border="1" width="100%" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >参数</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>api_service_code</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>接口代码，固定值: <code>create_pay</code> </td>
  </tr>
  <tr>
    <td>v</td>
    <td align="center" >是</td>
    <td>String(6)</td>
    <td>接口版本号，固定值: <code>1.0</code> </td>
  </tr>
  <tr>
    <td>client_code</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>授权可以调用接口的业务方代码。需要提前在 <a href="mailto:team-hyacinth@fun.tv?subject=[BESTPAY]申请接入支付系统">风信子团队</a> 申请</td>
  </tr>
  <tr>
    <td>nonce_str</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>用于防御回放攻击的，参考 <a href="#protocol">协议规则</a></td>
  </tr>
  <tr>
    <td>sign</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>签名，参考 <a href="#security_sign">签名算法</a></td>
  </tr>
  <tr>
    <td>order_id</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>对应支付系统的唯一编号，从 <a href="#interface_create_order">创建订单接口</a> 的结果中获得</td>
  </tr>
  <tr>
    <td>gateway_code</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>支付网关代码,参考 <a href="#enum_gateway_code">网关具体取值</a></td>
  </tr>
  <tr>
    <td>pay_scene</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>支付场景,参考 <a href="#enum_pay_scene">支付场景具体取值</a></td>
  </tr>
  <tr>
    <td>open_id</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>微信用户标识，当 <b>pay_scene</b> 为 <code>js_sdk</code> 时必填</td>
  </tr>
  <tr>
    <td>payment_fee</td>
    <td align="center" >是</td>
    <td>Long</td>
    <td>用户支付的实际金额（可能含手续费），单位为分</td>
  </tr>
</table>

* 返回结果：

    * 返回结果为 json 格式，字段说明：

<table border="1" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >字段</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>return_code</td>
    <td align="center" >是</td>
    <td>String(16)</td>
    <td>
      返回状态码:
      <ul>
        <li><code>SUCCESS</code> - 成功</li>
        <li>其他情况为具体错误代码，参考 <a href="#error_code">错误码</a> </li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>return_msg</td>
    <td align="center" >否</td>
    <td>String(128)</td>
    <td>返回信息，成功时内容为空，失败时返回错误原因</td>
  </tr>
  <tr>
    <td>data</td>
    <td align="center" >否</td>
    <td>Map</td>
    <td>
      成功时返回一个 Map 类型的数据，Key 值是传入的 pay_scene 参数值（参考 <a href="#enum_pay_scene">支付场景具体取值</a>）, 
      Value 值根据支付场景的不同各不相同（参考 <a href="#enum_pay_scene_gateway">支付网关支持的支付场景</a>）:
      <ul>
        <li>
          当 Key 为 <code>app_sdk</code> 时，如果网关支持此场景:<br>
          Value 为一个 Map 包含若干调用 App SDK 的参数</li>
        <li>
          当 Key 为 <code>js_sdk</code> 时，如果网关支持此场景:<br>
          Value 为一个 Map 包含若干调用 Js SDK 的参数</li>
        <li>
          当 Key 为 <code>qr_code</code> 时，如果网关支持此场景:<br>
          Value 为一个二维码使用的 URL 地址字符串</li>
        <li>
          当 Key 为 <code>url</code> 时，如果网关支持此场景:<br>
          Value 为一个跳转到支付网关的 URL 地址字符串</li>
      </ul>
    </td>
  </tr>
</table>

成功的例子:

```
  {
    "return_code" : "SUCCESS",
    "return_msg" : "",
    "data" : [
      "qr_code" : "weixin://wxpay/bizpayurl?pr=o8k7ySC"
    ]
  }
```

失败的例子:

```
  {
    "return_code" : "SIGN_ERROR",
    "return_msg" : "签名失败" 
  }
```

<h3 id="interface_create_order_and_pay">4.3.创建订单并支付接口</h3>
* URL: **http://pay.fun.tv/trade/api**
* 请求方式: GET
* 说明:

  * 结合了 [创建订单接口](#interface_create_order) 和 [支付接口](#interface_create_pay) 的行为
  
* 请求参数:

<table border="1" width="100%" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >参数</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>api_service_code</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>接口代码，固定值: <code>create_order_and_pay</code> </td>
  </tr>
  <tr>
    <td>v</td>
    <td align="center" >是</td>
    <td>String(6)</td>
    <td>接口版本号，固定值: <code>1.0</code> </td>
  </tr>
  <tr>
    <td>client_code</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>授权可以调用接口的业务方代码。需要提前在 <a href="mailto:team-hyacinth@fun.tv?subject=[BESTPAY]申请接入支付系统">风信子团队</a> 申请</td>
  </tr>
  <tr>
    <td>nonce_str</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>用于防御回放攻击的，参考 <a href="#protocol">协议规则</a></td>
  </tr>
  <tr>
    <td>sign</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>签名，参考 <a href="#security_sign">签名算法</a></td>
  </tr>
  <tr>
    <td>type</td>
    <td align="center" >是</td>
    <td>String(16)</td>
    <td>订单类型，参考 <a href="#enum_order_type">订单类型取值</a></td>
  </tr>
  <tr>
    <td>biz_trade_no</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>
      对应业务系统（参考 <a href="#noun">名词解释</a>）的唯一编号<br>
      规则为:大写字母、小写字母与数字的组合
    </td>
  </tr>
  <tr>
    <td>account_type</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>账号类型，参考 <a href="#enum_account_type">账号类型取值</a></td>
  </tr>
  <tr>
    <td>account_id</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td> <b>account_type</b> 标识的类型下的账号ID</td>
  </tr>
  <tr>
    <td>item_name</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>购买商品名称</td>
  </tr>
  <tr>
    <td>description</td>
    <td align="center" >否</td>
    <td>String(512)</td>
    <td>对一笔交易的具体描述信息</td>
  </tr>
  <tr>
    <td>callback_url</td>
    <td align="center" >是</td>
    <td>String(1024)</td>
    <td>订单完成之后通知业务方的地址</td>
  </tr>
  <tr>
    <td>return_url</td>
    <td align="center" >否</td>
    <td>String(1024)</td>
    <td>用户支付完订单后跳转的地址</td>
  </tr>
  <tr>
    <td>consume_fee</td>
    <td align="center" >是</td>
    <td>Long</td>
    <td>用户消费掉的余额，单位为分。纯充值操作时传 0</td>
  </tr>
  <tr>
    <td>gateway_code</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>支付网关代码,参考 <a href="#enum_gateway_code">网关具体取值</a></td>
  </tr>
  <tr>
    <td>pay_scene</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>支付场景,参考 <a href="#enum_pay_scene">支付场景具体取值</a></td>
  </tr>
  <tr>
    <td>open_id</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>微信用户标识，当 <b>pay_scene</b> 为 <code>js_sdk</code> 时必填</td>
  </tr>
  <tr>
    <td>payment_fee</td>
    <td align="center" >是</td>
    <td>Long</td>
    <td>用户支付的实际金额（可能含手续费），单位为分</td>
  </tr>
</table>

* 返回结果：

  * 返回结果为 json 格式，字段说明：

<table border="1" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >字段</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>return_code</td>
    <td align="center" >是</td>
    <td>String(16)</td>
    <td>
      返回状态码:
      <ul>
        <li><code>SUCCESS</code> - 成功</li>
        <li>其他情况为具体错误代码，参考 <a href="#error_code">错误码</a> </li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>return_msg</td>
    <td align="center" >否</td>
    <td>String(128)</td>
    <td>返回信息，成功时内容为空，失败时返回错误原因</td>
  </tr>
  <tr>
    <td>order_id</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>成功时返回支付系统的订单ID</td>
  </tr>
  <tr>
    <td>data</td>
    <td align="center" >否</td>
    <td>Map</td>
    <td>
      成功时返回一个 Map 类型的数据，Key 值是传入的 pay_scene 参数值, 参考 <a href="#enum_pay_scene">支付场景具体取值</a>, 
      Value 值根据支付场景的不同各不相同（参考 <a href="#enum_pay_scene_gateway">支付网关支持的支付场景</a>）:
      <ul>
        <li>
          当 Key 为 <code>app_sdk</code> 时，如果网关支持此场景:<br>
          Value 为一个 Map 包含若干调用 App SDK 的参数</li>
        <li>
          当 Key 为 <code>js_sdk</code> 时，如果网关支持此场景:<br>
          Value 为一个 Map 包含若干调用 Js SDK 的参数</li>
        <li>
          当 Key 为 <code>qr_code</code> 时，如果网关支持此场景:<br>
          Value 为一个二维码使用的 URL 地址字符串</li>
        <li>
          当 Key 为 <code>url</code> 时，如果网关支持此场景:<br>
          Value 为一个跳转到支付网关的 URL 地址字符串</li>
      </ul>
    </td>
  </tr>
</table>

成功的例子:

```
  {
    "return_code" : "SUCCESS",
    "return_msg" : "",
    "order_id" : "19LJ1V1IR2B2zuan",
    "data" : [
      "qr_code" : "weixin://wxpay/bizpayurl?pr=o8k7ySC"
    ]
  }
```

失败的例子:

```
  {
    "return_code" : "SIGN_ERROR",
    "return_msg" : "签名失败" 
  }
```

<h3 id="interface_notify_biz_sys">4.4.支付结果通知</h3>
* 说明:

  * 支付完成后，支付系统会将相关支付结果和用户信息发送给业务系统（通过 [创建订单接口](#interface_create_order) 和 [创建订单并支付接口](#interface_create_order_and_pay) 的 *callback_url* 参数）
  * 对后台通知交互时，如果业务系统应答不是成功或超时，支付系统认为通知失败，会通过一定策略定期发送通知——但不保证最终能成功
  * 由于存在重新发送后台通知的情况，因此业务系统必须能正确处理重复的通知
  * return_code 参数值只可能为 `SUCCESS` 或 `CONSUME_FAIL`，因为支付失败是不会发送通知给业务系统的

* 请求方式: GET
* 请求参数:

<table border="1" width="100%" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >参数</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>v</td>
    <td align="center" >是</td>
    <td>String(6)</td>
    <td>接口版本号，固定值: <code>1.0</code> </td>
  </tr>
  <tr>
    <td>client_code</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>授权可以调用接口的业务方代码。需要提前在 <a href="mailto:team-hyacinth@fun.tv?subject=[BESTPAY]申请接入支付系统">风信子团队</a> 申请</td>
  </tr>
  <tr>
    <td>nonce_str</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>用于防御回放攻击的，参考 <a href="#protocol">协议规则</a></td>
  </tr>
  <tr>
    <td>sign</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>签名，参考 <a href="#security_sign">签名算法</a></td>
  </tr>
  <tr>
    <td>return_code</td>
    <td align="center" >是</td>
    <td>String(16)</td>
    <td>
      返回状态码:
      <ul>
        <li><code>SUCCESS</code> - 表示支付及消费都成功</li>
        <li><code>CONSUME_FAIL</code> - 支付成功但消费失败</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>return_msg</td>
    <td align="center" >否</td>
    <td>String(128)</td>
    <td>返回信息，如非空，为错误原因</td>
  </tr>
  <tr>
    <td>biz_trade_no</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>对应业务系统（参考 <a href="#noun">名词解释</a>）的唯一编号</td>
  </tr>
  <tr>
    <td>gateway_code</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>支付网关代码,参考 <a href="#enum_gateway_code">网关具体取值</a> </td>
  </tr>
  <tr>
    <td>recharge_fee</td>
    <td align="center" >是</td>
    <td>Long</td>
    <td>真正可以进入用户账号余额中的金额。单位为分</td>
  </tr>
  <tr>
    <td>payment_fee</td>
    <td align="center" >是</td>
    <td>Long</td>
    <td>用户真正支付的金额，单位为分</td>
  </tr>
  <tr>
    <td>consume_fee</td>
    <td align="center" >否</td>
    <td>Long</td>
    <td>用户消费掉的余额。单位为分</td>
  </tr>
</table>

* 业务系统需要返回 json 格式文本：

<table border="1" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >字段</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>return_code</td>
    <td align="center" >是</td>
    <td>String(16)</td>
    <td>
      返回状态码:
      <ul>
        <li><code>SUCCESS</code> - 成功</li>
        <li><code>FAIL</code> - 失败</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>return_msg</td>
    <td align="center" >否</td>
    <td>String(128)</td>
    <td>返回信息，如非空，为错误原因</td>
  </tr>
</table>  
    
成功的例子:

```
  {
    "return_code" : "SUCCESS",
    "return_msg" : ""
  }
```

失败的例子:
```
  {
    "return_code" : "FAIL",
    "return_msg" : "签名失败" 
  }
```

<h3 id="interface_balance_query">4.5.查询账号余额接口</h3>
* URL: **http://pay.fun.tv/trade/api**
* 请求方式: GET
* 说明:

  * 查询业务系统中账号在支付系统中的余额情况
  
* 请求参数:

<table border="1" width="100%" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >参数</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>api_service_code</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>接口代码，固定值: <code>balance_query</code> </td>
  </tr>
  <tr>
    <td>v</td>
    <td align="center" >是</td>
    <td>String(6)</td>
    <td>接口版本号，固定值: <code>1.0</code> </td>
  </tr>
  <tr>
    <td>client_code</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>授权可以调用接口的业务方代码。需要提前在 <a href="mailto:team-hyacinth@fun.tv?subject=[BESTPAY]申请接入支付系统">风信子团队</a> 申请</td>
  </tr>
  <tr>
    <td>nonce_str</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>用于防御回放攻击的，参考 <a href="#protocol">协议规则</a></td>
  </tr>
  <tr>
    <td>sign</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>签名，参考 <a href="#security_sign">签名算法</a></td>
  </tr>
  <tr>
    <td>account_type</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>账号类型，参考 <a href="#enum_account_type">账号类型取值</a></td>
  </tr>
  <tr>
    <td>account_id</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td> <b>account_type</b> 标识的类型下的账号ID</td>
  </tr>
  <tr>
</table>

* 返回结果：

    * 返回结果为 json 格式，字段说明：

<table border="1" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >字段</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>return_code</td>
    <td align="center" >是</td>
    <td>String(16)</td>
    <td>
      返回状态码:
      <ul>
        <li><code>SUCCESS</code> - 成功</li>
        <li>其他情况为具体错误代码，参考 <a href="#error_code">错误码</a> </li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>return_msg</td>
    <td align="center" >否</td>
    <td>String(128)</td>
    <td>返回信息，成功时内容为空，失败时返回错误原因</td>
  </tr>
  <tr>
    <td>balance</td>
    <td align="center" >否</td>
    <td>Long</td>
    <td>余额，单位为分</td>
  </tr>
</table>

成功的例子:

```
  {
    "return_code" : "SUCCESS",
    "return_msg" : "",
    "balance" : 4000
  }
```

失败的例子:

```
  {
    "return_code" : "SIGN_ERROR",
    "return_msg" : "签名失败" 
  }
```

<h3 id="interface_order_query">4.6.订单查询接口接口</h3>
* URL: **http://pay.fun.tv/trade/api**
* 请求方式: GET
* 说明:

  * 指定一个支付系统（参考 [名词解释](#noun) ）的订单ID（ *order_id* ） 查询订单详情
  
* 请求参数:

<table border="1" width="100%" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >参数</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>api_service_code</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>接口代码，固定值: <code>order_query</code> </td>
  </tr>
  <tr>
    <td>v</td>
    <td align="center" >是</td>
    <td>String(6)</td>
    <td>接口版本号，固定值: <code>1.0</code> </td>
  </tr>
  <tr>
    <td>client_code</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>授权可以调用接口的业务方代码。需要提前在 <a href="mailto:team-hyacinth@fun.tv?subject=[BESTPAY]申请接入支付系统">风信子团队</a> 申请</td>
  </tr>
  <tr>
    <td>nonce_str</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>用于防御回放攻击的，参考 <a href="#protocol">协议规则</a></td>
  </tr>
  <tr>
    <td>sign</td>
    <td align="center" >是</td>
    <td>String(32)</td>
    <td>签名，参考 <a href="#security_sign">签名算法</a></td>
  </tr>
  <tr>
    <td>order_id</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>对应支付系统的唯一编号，从 <a href="#interface_create_order">创建订单接口</a> 的结果中获得</td>
  </tr>
</table>

* 返回结果：

    * 返回结果为 json 格式，字段说明：

<table border="1" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >字段</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>return_code</td>
    <td align="center" >是</td>
    <td>String(16)</td>
    <td>
      返回状态码:
      <ul>
        <li><code>SUCCESS</code> - 成功</li>
        <li>其他情况为具体错误代码，参考 <a href="#error_code">错误码</a> </li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>return_msg</td>
    <td align="center" >否</td>
    <td>String(128)</td>
    <td>返回信息，成功时内容为空，失败时返回错误原因</td>
  </tr>
  <tr>
    <td>order_type</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>订单类型，参考 <a href="#enum_order_type">订单类型取值</a> </td>
  </tr>
  <tr>
    <td>biz_trade_no</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>
      对应业务系统，（参考 <a href="#noun">名词解释</a>）的唯一编号<br>
      规则为:大写字母、小写字母与数字的组合 
    </td>
  </tr>
  <tr>
    <td>account_type</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>这个订单相关的账号类型，参考 <a href="#enum_account_type">账号类型取值</a></td>
  </tr>
  <tr>
    <td>account_id</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>这个订单相关的账号ID（在 <b>account_type</b> 标识的类型下的账号ID）</td>
  </tr>
  <tr>
    <td>item_name</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>购买商品名称</td>
  </tr>
  <tr>
    <td>order_status</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>订单当前状态，参考 <a href="#enum_order_status">订单状态取值</a> </td>
  </tr>
  <tr>
    <td>trade_status</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>
      订单当前总的支付状态:，参考 <a href="#enum_trade_status">支付状态取值</a> 
      <ul>
        <li>如果这个订单不涉及支付，这个值为空字符串</li>
        <li>如果任何一次支付成功，就认为订单总的支付状态是 <code>success</code> </li>
        <li>在没有任何一个网关支付成功的情况下，如果有网关返回支付失败，就认为总的支付状态是 <code>failed</code> </li>
        <li>不是上面两种情况，就认为总的支付状态是 <code>paying</code> </li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>consume_status</td>
    <td align="center" >否</td>
    <td>String(64)</td>
    <td>订单的消费状态。如果这个订单不涉及消费，这个值为空字符串。参考 <a href="#enum_consume_status">订单消费状态取值</a> </td>
  </tr>
  <tr>
    <td>consume_fee</td>
    <td align="center" >否</td>
    <td>Long</td>
    <td>消费的金额，单位为分</td>
  </tr>
  <tr>
    <td>create_time</td>
    <td align="center" >否</td>
    <td>Long</td>
    <td>一个时间戳，记录订单创建时间，参考 <a href="#protocol">协议规则</a> </td>
  </tr>
  <tr>
    <td>trades</td>
    <td align="center" >否</td>
    <td>Map</td>
    <td>
      这是一个 Map ，包含了各个网关当前的支付情况（只有发起过支付流程的网关才会列出）:<br>
      Key 值为网关代码;<br>
      Value 为具体信息对象（参考 <a href="#interface_order_query_return_trades">具体网关支付信息内容</a>） 
    </td>
  </tr>
</table>

<p id="interface_order_query_return_trades">具体网关支付信息内容: </p>

<table border="1" cellpadding="8" cellspacing="0" >
  <tr bgcolor="#d5d5d5" >
    <th width="200" >字段</th>
    <th width="50" >必填</th>
    <th width="100" >类型</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>gateway_name</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>返网关的名称</td>
  </tr>
  <tr>
    <td>trade_status</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>此网关当前的支付状态，参考 <a href="#enum_trade_status">支付状态取值</a></td>
  </tr>
  <tr>
    <td>pay_count</td>
    <td align="center" >是</td>
    <td>String(64)</td>
    <td>指定网关尝试支付次数</td>
  </tr>
  <tr>
    <td>recharge_fee</td>
    <td align="center" >是</td>
    <td>Long</td>
    <td>用户实际充值金额。单位为分</td>
  </tr>
  <tr>
    <td>payment_fee</td>
    <td align="center" >是</td>
    <td>Long</td>
    <td>用户实际支付金额，在有手续费的情况下大于用户实际充值金额( <b>recharge_fee</b> )，否则两个值相等。单位为分</td>
  </tr>
</table>
    
成功的例子:

```
  {
    "return_code" : "SUCCESS",
    "return_msg" : "",
    "order_type" : "recharge_consume",
    "biz_trade_no" : "d2dfdaf9d32f332321",
    "account_type" : "funshion",
    "account_id" : "jiangxu",
    "item_name" : "爱疯6",
    "order_status" : "pending",
    "trade_status" : "paying",
    "consume_status" : "waiting",
    "consume_fee" : 600000,
    "create_time" : 1431314507,
    "trades" : {
      "WEIXIN" : {
        "gateway_name" : "微信支付",
        "trade_status" : "paying",
        "pay_count" : 0,
        "payment_fee" : 600000,
        "recharge_fee" : 600000
      },
      "ALIPAY_WAP_TRADE" : {
        "gateway_name" : "支付宝手机网页即时到账",
        "trade_status" : "paying",
        "pay_count" : 0,
        "payment_fee" : 600000,
        "recharge_fee" : 600000
      }
    }
  }
```

失败的例子:

```
  {
    "return_code":"SIGN_ERROR",
    "return_msg":"签名失败" 
  }
```

<h2 id="enum">5.枚举类型参数取值</h2>
<h3 id="enum_order_type">5.1.订单类型取值</h3>

`recharge`

 * 充值
 * 表示通过支付网关向用户的余额中充值

`consume`

 * 消费
 * 表示从用户的余额中消费

`recharge_consume`

 * 充消
 * 表示先通过支付网关向用户的余额中充值，充值成功后从用户的余额中消费

<h3 id="enum_account_type">5.2.账号类型取值</h3>

`funshion`

* 风行账号

<h3 id="enum_order_status">5.3.订单状态取值</h3>

`created`

* 创建
* 当调用 [创建订单接口](#interface_create_order) 时，如果 type 参数为 充值( *recharge* ) 或 充消( *recharge_consume* )，此时订单为 *created* 状态，表示订单处于一种预下单且没有决定到底如何支付的状态。

`pending`

* 处理中
* 当调用 [支付接口](#interface_create_pay) 或 [创建订单并支付接口](#interface_create_order_and_pay) 时，在任何一个网关通知支付系统支付成功之前，订单都处于 *pending* 状态，表示订单已经开始和网关进行支付流程，但还没有取得成功。

`closed`

* 关闭
* 如果一个订单创建后，收到明确的关闭指令（可能是通过管理后台），订单为 *closed* 状态。
* 对于一个已经处于关闭状态的订单，无法再发起支付动作。

`finished`

* 结束
* 当任何一个网关通知支付系统支付成功后，订单处于 *finished* 状态，表示整个支付过程已经成功。
* 处于 *closed* 状态的订单一样可能由于网关的通知而变成 *finished* 状态。
* 从余额中消费的过程是不会影响订单状态的。

<h3 id="enum_trade_status">5.4.支付状态取值</h3>

`paying`

* 正在支付
* 对网关发起一次支付流程后，在收到网关通知之前，此网关的支付状态处于 *paying* 。

`failed`

* 支付失败
* 当网关通知支付失败后，此网关的支付状态为 *failed* 。
* 对有些网关可以对 *failed* 状态的支付过程再次发起支付流程。

`success`

* 支付成功
* 当网关通知支付成功后，此网关的支付状态为 *success* 。 
* 由于网络通讯可能不可靠，如果已经收到网关通知成功后，再收到网关通知失败，此时任务这次支付还是成功的。

<h3 id="enum_consume_status">5.5.订单消费状态取值</h3>

`waiting`

* 等待消费
* 当订单的 type 参数为 充消( *recharge_consume* ) 时，在充值过程成功之前，消费状态都为 *waiting* 。

`success`

* 消费成功
* 当成功从用户余额中消费后，消费状态为 *success* 。

`failed`

* 消费失败
* 当从用户余额消费失败时（一般的情况有用户被锁定或用户余额不足），消费状态为 *failed* 。

<h3 id="enum_gateway_code">5.6.网关具体取值</h3>

`WEIXIN`

* 微信

`ALIPAY`

* 支付宝

<h3 id="enum_pay_scene">5.7.支付场景具体取值</h3>

`qr_code`

* 二维码支付

`js_sdk`

* 通过 Js SDK 支付

`app_sdk`

* 通过 App SDK

`url`

* 通过 URL 跳转方式

<h3 id="enum_pay_scene_gateway">5.7.1.支付网关支持的支付场景</h3>

<table border="1" cellpadding="8" cellspacing="0" >
  <tr>
    <th align="center" width="200" bgcolor="#d5d5d5" >网关 \ 支付场景</th>
    <td align="center" width="50" bgcolor="#d5d5d5">qr_code</td>
    <td align="center" width="50" bgcolor="#d5d5d5">js_sdk</td>
    <td align="center" width="50" bgcolor="#d5d5d5">app_sdk</td>
    <td align="center" width="50" bgcolor="#d5d5d5">url</td>
  </tr>
  <tr>
    <td align="center" bgcolor="#d5d5d5">微信(WEIXIN)</td>
    <td align="center" > √ </td>
    <td align="center" > √ </td>
    <td align="center" > √ </td>
    <td align="center" > &nbsp; </td>
  </tr>
  <tr>
    <td align="center" bgcolor="#d5d5d5">支付宝(ALIPAY)</td>
    <td align="center" > √ </td>
    <td align="center" > &nbsp; </td>
    <td align="center" > &nbsp; </td>
    <td align="center" > √ </td>
  </tr>
</table>

<h2 id="error_code">6.错误码</h2>

`SYSTEM_ERROR`

* 系统错误
* 支付系统出现未知错误，需要与 <a href="mailto:team-hyacinth@fun.tv?subject=[BESTPAY]协助排查SYSTEM_ERROR原因">风信子团队</a> 联系排查原因

`SYSTEM_BUSY`

* 系统繁忙
* 此时请稍后重试，或与 <a href="mailto:team-hyacinth@fun.tv?subject=[BESTPAY]协助排查SYSTEM_BUSY原因">风信子团队</a> 联系排查原因

`NO_SYS_PROP`

* 缺少\[ *api_service_code* , *client_code* , *sign* \]等必传参数

`NO_AUTH`

* 没有访问该接口的权限 

`PARAM_NULL`

* 参数为空
* 未传递必填参数，或传递了一个空值

`PARAM_ERROR`

* 参数错误
* 可能是参数不符合要求的格式，或者参数不符合约束的条件，或者此参数对应的数据不存在

`CONSUME_NOT_SUPPORT`

* 消费类型的订单请使用下单接口

`SIGN_ERROR`

* 签名错误 

`DUPLICATE_BIZ_ORDER`

* 重复下单
* 如果传递的业务系统ID *biz_trade_no* 重复，会被认为是重复下单

`URL_ERROR`

* 请求的URL错误 

`WEIXIN_ORDER_ERROR`

* 微信订单创建失败

`ALIPAY_ORDER_ERROR`

* 支付宝订单创建失败

`INVALID_ORDER_ID`

* 无效的订单号

`ORDER_OVER`

* 订单已经关闭或者完成

`ORDER_TYPE_NOT_SUPPORT`

* 此订单类型不支持该操作

`NOT_ENOUGH_PAYMENT_FEE`

* 支付金额不足

`CONSUME_ERROR`

* 消费失败
* 可能用户账号被锁定，或者用户余额不足

`PAY_SCENE_NOT_MATCH`

* 支付场景错误

`PAY_SCENE_NOT_SUPPORT`

* 网关不支持该支付场景

`CONSUME_MUST_ZERO`

* 充值订单的消费金额必须为 0

`CONSUME_MUST_GT_ZERO`

* 消费金额必须大于0

`NEED_OPEN_ID`

* 微信 *js_sdk* 的支付方式必须有 *open_id*

`GATEWAY_PAY_ERROR`

* 网关支付失败

