---
title: 飞书AT人
date: 2024-05-06 11:22:12
tags:
---

飞书实现 @ 人操作

**第一步**

自建应用获取 tenant_access_token （可以缓存2个小时）

https://open.feishu.cn/document/ukTMukTMukTM/ukDNz4SO0MjL5QzM/auth-v3/auth/tenant_access_token_internal


说明： `app_access_token` 的最大有效期是 2 小时。如果在有效期小于 30 分钟的情况下，调用本接口，会返回一个新的 `app_access_token`，这会同时存在两个有效的 `app_access_token`。

**第二步**

通过手机号或邮箱获取用户 ID （仅支持组织范围内）

### 通过手机号或邮箱获取用户 ID

contact:user.id:readonly

https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/reference/contact-v3/user/batch_get_id

**第三步**

webhook 调用机器人即可

https://open.feishu.cn/document/ukTMukTMukTM/ucTM5YjL3ETO24yNxkjN

如果使用阿里云SLS对接，请使用 Markdown语法

https://open.feishu.cn/document/ukTMukTMukTM/uADOwUjLwgDM14CM4ATN?spm=5176.2020520112.help.10.53c534c0OKWv7y

### 如何实现@指定人、@所有人

可以在机器人发送的普通文本消息（text）、富文本消息（post）、消息卡片（interactive）中，使用at标签实现@人效果。具体请求示意如下：
**(1) 在普通文本消息 (text) 中@人、@所有人:**
`at`标签说明:

```
// at 指定用户
<at user_id="ou_xxx">Name</at> //取值必须使用ou_xxxxx格式的 open_id 来at指定人
// at 所有人
<at user_id="all">所有人</at> 
```

请求体示意：

```
{
	"msg_type": "text",
	"content": {
		"text": "<at user_id = \"ou_f43d7bf0bxxxxxxxxxxxxxxx\">Tom</at> text content"
	}
}
```

**(2) 在富文本消息 (post) 中@人、@所有人:**
`at`标签说明:

```
// at 指定用户
{
	"tag": "at",
	"user_id": "ou_xxxxxxx",//取值必须使用ou_xxxxx格式的 open_id 来at指定人
	"user_name": "tom"
}
// at 所有人
{
	"tag": "at",
	"user_id": "all",//取值使用"all"来at所有人
	"user_name": "所有人"
} 
```

请求体示意：

```
{
	"msg_type": "post",
	"content": {
		"zh_cn": {
			"title": "我是一个标题",
			"content": [
				[{
						"tag": "text",
						"text": "第一行 :"
					},
					{
						"tag": "at",
						"user_id": "ou_xxxxxx",
						"user_name": "tom"
					}
				],
				[{
						"tag": "text",
						"text": "第二行:"
					},
					{
						"tag": "at",
						"user_id": "all",
						"user_name": "所有人"
					}
				]
			]
		}
	}

}
```

**(3) 在消息卡片 (interactive) 中@人、@所有人:**
可以使用消息卡片Markdown内容中的at人标签，标签示意：

```
// at 指定用户
	<at id=ou_xxx></at> //取值必须使用ou_xxxxx格式的 open_id 来at指定人
// at 所有人
	<at id=all></at> 
```

请求体中的 `card` 内容示意：

```
{
	"msg_type": "interactive",
	"card": {
		"elements": [{
			"tag": "div",
			"text": {
				"content": "at所有人<at id=all></at> \n at指定人<at id=ou_xxxxxx></at>",
				"tag": "lark_md"
			}
		}]
	}
}
```



