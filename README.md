# 概述

Jeejio JM SDK 致力于让第三方开发者开发基于 Jeejio 智能设备的应用时，快速集成与智能设备及其所属用户通信的能力。除了接收用户发送的文字、语音外，还可以处理规定格式的命令。

# 集成指南

## 添加依赖

请将如下 aar 包放入 libs 文件夹中

-   jmessagemodule-release
-   smack-core-release
-   smack-extensions-release
-   smack-im-release
-   smack-sasl-provided-release
-   smack-tcp-release

eg:

```xml
    implementation(name: 'jmessagemodule-release', ext: 'aar')
    implementation(name: 'smack-core-release', ext: 'aar')
    implementation(name: 'smack-extensions-release', ext: 'aar')
    implementation(name: 'smack-im-release', ext: 'aar')
    implementation(name: 'smack-sasl-provided-release', ext: 'aar')
    implementation(name: 'smack-tcp-release', ext: 'aar')
```

QA 环境

-   jmessagemodule-debug
-   smack-core-debug
-   smack-extensions-debug
-   smack-im-debug
-   smack-sasl-provided-debug
-   smack-tcp-debug

```xml
    implementation(name: 'jmessagemodule-debug', ext: 'aar')
    implementation(name: 'smack-core-debug', ext: 'aar')
    implementation(name: 'smack-extensions-debug', ext: 'aar')
    implementation(name: 'smack-im-debug', ext: 'aar')
    implementation(name: 'smack-sasl-provided-debug', ext: 'aar')
    implementation(name: 'smack-tcp-debug', ext: 'aar')
```

除此以外，JM 还用到了 OkHttp 请求,Gson 解析及阿里云 Oss 文件存储，所以还需在 build.gradle 中添加如下依赖：

```xml
    // 网络请求库
    implementation 'com.squareup.okhttp3:okhttp:3.11.0'
    // Json 数据解析库
    implementation 'com.google.code.gson:gson:2.8.5'
    // 阿里云 OSS 存储
    implementation('com.aliyun.dpa:oss-android-sdk:2.9.2', {
    	exclude module: "okhttp"
    	exclude module: "okio"
    })
```

* * *

## 登录注册

## 初始化SDK

登录前需做 SDK 的初始化。

eg：

```java
// 初始化 Oss
OssHelper.SINGLETON.init(context);
// 初始化 Jeejio Message
try {
	// 如果需要处理命令，可以在 init 方法中传入命令处理者的 Class，命令规则详见：命令
	JMClient.SINGLETON.init(context);
} catch (InstantiationException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
}
```

### 注册

接入 SDK 的应用是提供给物栖智能设备的应用，无需注册，当该应用下载到物栖智能设备上会自动完成注册操作。

### 登录

SDK 中所有操作均需在登录之后进行。

eg：

```java
// 第三方应用用户登录
JMClient.SINGLETON.applicationLogin(new ILoginListener() {
	@Override
	public void onSuccess() {
		// 登录成功
	}

	@Override
	public void onFailure(Exception e) {
		// 登录失败
	}
});
```

<!-- // 用户名+密码登录模式
// username:用户名
// password:密码
JMClient.SINGLETON.login(username, password, new ILoginListener() {
	@Override
	public void onSuccess() {
		// 登录成功
	}

	@Override
	public void onFailure(Exception e) {
		// 登录失败
	}
}); -->

### 登出

虽然即使不调用登出方法，服务器也会在一段时间后自动断开闲置客户端，但为了更好的状态同步，需在应用退出时及时调用登出方法。

eg：

```java
JMClient.SINGLETON.logout();
```

### 添加连接状态监听

当监听器的 connectionClosedOnError() 方法被回调时，如果错误信息中包含 conflict 关键字，则表示是同一个账号在另一个客户端上登录，当前客户端会被踢下线。

eg：

```java
JMClient.SINGLETON.addOnConnectListener(new IOnConnectListener() {
    @Override
    public void onConnected() {
        // 连接成功
    }

    @Override
    public void onAuthenticated() {
		// 登录成功
    }

    @Override
    public void onConnectFailure(Exception e) {
		// 连接异常断开
		if (e instanceof SmackException.LoginConflictException) {
			// 登录冲突
		}
    }

    @Override
    public void onDisconnected() {
		// 连接断开
    }
});
```

### 手动重连

服务端为了节省资源，通常会关闭闲置客户端，虽然客户端与服务端有心跳机制进行保活，但仍然存在会话失效的情况，这时候客户端在 ping 服务器发现没有响应（3 次）的时候，会关闭连接。
客户端可以监听到连接关闭，这时候可以调用登录方法进行手动重连。

### 添加好友状态监听

eg:

```java
JMClient.SINGLETON.addOnFriendStatusListener(new IOnFriendStatusListener() {
    /**
     * 联系人上线时回调该方法
     * @param username 上线的用户的 username,即 localPart 部分
     */
    @Override
    public void onOnline(String username) {
    }

    /**
	 * 联系人下线时回调该方法
     * @param username 下线的用户的 username,即 localPart 部分
     */
    @Override
    public void onOffline(String username) {
    }
    /**
     * 收到好友请求时回调
     * @param username  发送请求的用户的 username,即 localPart 部分
     * @param additionalMsg    附加的描述,该参数可能没有
     * @param createTime 申请时间
     */
    @Override
    public void onSubscribe(String username, String additionalMsg, String createTime) {
    }

    /**
     * 对方同意添加联系人时回调该方法
     * @param username 同意请求的用户的 username,即 localPart 部分
     */
    @Override
    public void onSubscribed(String username) {
    }

    /**
     * 对方拒绝添加联系人时回调该方法
     * @param username 对方的 username,即 localPart 部分
     */
    @Override
    public void onUnsubscribed(String username) {
    }
    
    /**
     * 对方删除联系人时回调该方法
     * @param username 发送请求的用户的 username,即 localPart 部分
     */
    @Override
    public void onUnsubscribe(String username) {
    }
    
    /**
     * 发生错误时回调该方法
     */
    @Override
    public void onError() {
    }
});
```

### 移除好友状态监听

eg：

```java
JMClient.SINGLETON.removeFriendListener(mOnFriendStatusListener);
```

### 添加群聊监听

eg：

```java
JMClient.SINGLETON.addOnGroupChatStatusListener(new IOnGroupChatStatusListener() {
    /**
     * 群聊被创建时回调
     * @param localpart 群聊标识
     * @param owner     群主的 username
     */
    @Override
    public void groupChatCreated(String localpart, String owner) {
    }

    /**
     * 群主变更时回调
     * @param localpart 群聊唯一标识
     * @param ownerCurrent 当前群主的 username
     * @param ownerBefore  之前群主的 username
     */
    @Override
    public void ownerChanged(String localpart, String ownerCurrent, String ownerBefore) {
    }

    /**
     * 有人修改在本群中的昵称时回调
     * @param localpart 群聊唯一标识
     * @param occupant  群成员的 username
     * @param name      群成员的新的在本群中的昵称
     */
    @Override
    public void occupantNameChanged(String localpart, String occupant, String name) {
    }

    /**
     * 群头像被修改时回调
     * @param localpart 群聊唯一标识
     * @param img       群头像
     * @param executor  执行人的 username
     */
    @Override
    public void imageChanged(String localpart, String img, String executor) {
    }

    /**
     * 群名称被修改时回调
     * @param localpart 群聊唯一标识
     * @param name      群聊名称
     * @param executor  执行人的 username
     */
    @Override
    public void nameChanged(String localpart, String name, String executor) {
    }

    /**
     * 有人退出群聊时回调
     * @param localpart 群聊唯一标识
     * @param occupant  退出的群成员的 username
     */
    @Override
    public void leave(String localpart, String occupant) {
    }

    /**
     * 群销毁时回调
     * @param localpart 群聊唯一标识
     */
    @Override
    public void destroy(String localpart) {
    }

    /**
     * 有群成员被踢出群聊时回调
     * @param localpart 群聊唯一标识
     * @param kicked    被踢出的群成员的 username
     * @param executor  执行人的 username
     */
    @Override
    public void kick(String localpart, String kicked, String executor) {
    }

    /**
     * 有新的用户加入群聊时回调
     * @param localpart 群聊唯一标识
     * @param occupant  新成员的 username
     * @param executor  执行人的 username
     * @param pOccupant 新成员信息
     */
    @Override
    public void join(String localpart, String occupant, String executor, Presence.Occupant pOccupant) {
    }
});
```

### 移除群聊监听

eg：

```java
JMClient.SINGLETON.removeOnGroupChatStatusListener(mOnGroupChatStatusListener);
```

* * *

## 消息

发送文本、语音、命令等消息（单聊、群聊通用）。发送消息需要明确指示消息接收方，其唯一标识可以在获取我的好友列表后通过 userDetailBean.getLoginName() 或获取我的群列表后通过 groupChatBean.getId() 拿到。

### 消息管理者类 MessageManager

通过 JMClient 单例来获取 MessageManager

eg：

```java
JMClient.SINGLETON.getMessageManager();
```

### 接收消息

在需要监听消息的界面添加监听器，单聊、群聊通用，根据接收到的 message 的不同类型做不同的业务逻辑即可。

#### 消息类型 MessageType

目前 MessageType 中仅列举了两种类型

-   CHAT：单聊，对应值 1001
-   GROUP_CHAT：群聊，对应值 1002

#### 消息内容类型 MessageContentType

目前 MessageContentType 中列举了五种类型
TEXT：普通文本，对应值 2001
IMAGE：图片，对应值 2002
VOICE：语音，对应值 2003
FILE：（暂不支持）文件，对应值 2004
COMMAND：命令，对应值 2005

eg：

```java
JMClient.SINGLETON.addOnMessageListener(new IOnMessageListener() {
    @Override
    public void onMessage(MessageBean messageBean) {
        // 收到消息
    }
});
```

### 发送消息

#### 发送文本消息

eg：

```java
// toUsername:单聊为对方 id，群聊为群 id
MessageBean message = MessageBean.createTextMessage(toUsername,content);
// 群聊 Message
// MessageBean message = MessageBean.createTextMessage(toUsername,content);
// message.setType(MessageBean.Type.GROUP_CHAT.getValue())
// 或
// MessageBean message = MessageBean.createTextMessage(toUsername,MessageBean.Type.GROUP_CHAT,content);
JMClient.SINGLETON.getMessageManager().sendMessage(message);
```

如果需要监听发送结果，则调用

eg:

```java
JMClient.SINGLETON.getMessageManager().sendMessage(message, new JMCallback<MessageBean>() {
	@Override
	public void onSuccess(MessageBean messageBean) {
		// 发送成功
	}

	@Override
	public void onFailure(Exception e) {
		// 发送失败会回调该方法，目前只有双方不为好友或超时会发送失败
	}
});
```

#### 发送语音消息

eg:

```java
// toUsername:单聊为对方 id，群聊为群 id
// filePath:语音文件路径
// time:语音时长
MessageBean message = MessageBean.createVoiceMessage(toUsername,filePath,time);
// 群聊 Message 请参考文本消息
JMClient.SINGLETON.getMessageManager().sendMessage(message);
```

如果需要监听发送结果，请参考文本消息

#### 发送命令消息

命令本质上与普通文本并无区别，所以在 Message 的创建时传参与普通文本一样，只是调用的创建方法不同。

eg:

```java
// toUsername:单聊为对方 id，群聊为群 id
// content:命令内容
MessageBean message = MessageBean.createCommandMessage(toUsername,content);
// 群聊 Message 请参考文本消息
JMClient.SINGLETON.getMessageManager().sendMessage(message);
```

如果需要监听发送结果，请参考文本消息

#### 发送扩展消息

如果以上方法仍满足不了需求，可以设置 Message 的 extend 属性，作为扩展字段，自己定义解析规则，推荐 json 格式的字符串。

eg:

```java
MessageBean message = new MessageBean(content, toUsername);
// 群聊 Message 请参考文本消息
message.setExtend("{"key":"value"}");
```

* * *

## 好友管理

### 好友管理者类 FriendManager

通过 JMClient 单例来获取 FriendManager

eg:

```java
JMClient.SINGLETON.getFriendManager();
```

## 获取好友列表

由于 JM 的用户系统是区分类型的，分为普通用户类型（userType 为 1）、设备用户类型（userType 为 2）、应用用户类型（userType 为 3），所以在查询好友列表时需分别查询。
好友列表在登录成功后会同步一遍服务器数据，之后都是从缓存中获取

eg：

```java
JMClient.SINGLETON.getFriendManager().getList(userType, new JMCallback<List<UserDetailBean>>() {
	@Override
	public void onSuccess(List<UserDetailBean> userDetailBeanList) {
		
	}

	@Override
	public void onFailure(Exception e) {
		
	}
});
```

<!-- ### 获取单个好友
```java
JMClient.SINGLETON.getFriendManager().getContactInfo(userType, username, new JMCallback<UserBean>() {
	@Override
	public void onSuccess(UserBean userBean) {

	}

	@Override
	public void onFailure(Exception e) {

	}
});
``` -->

<!--
### 添加好友
eg：
```java
// loginName 为待添加好友的用户名
// remark 为备注,可以为空
// source 为用户添加来源
// invitationMessage  为附加验证消息
JMClient.SINGLETON.getFriendManager().add(loginName, remark, source, invitationMessage, new JMCallback<Object>() {

	@Override
	public void onSuccess(Object object) {
		// 发送添加请求成功
	}

	@Override
	public void onFailure(Exception e) {
		// 发送添加请求失败
	}
}); 
```
-->

<!--
### 同意添加好友
eg：
```java
// loginName 为待添加好友的用户名
JMClient.SINGLETON.getFriendManager().agreeAdd(loginName);
```
-->

<!--
### 删除好友
eg：
```java
// loginName 为待删除好友的用户名
JMClient.SINGLETON.getFriendManager().delete(loginName, new JMCallback<Object>() {

	@Override
	public void onSuccess(Object object) {
		// 删除成功
	}

	@Override
	public void onFailure(Exception e) {
		// 删除失败
	}
});
```
-->

### 修改备注

eg：

```java
// loginName 为对方的用户名
// remarkName 为新备注
JMClient.SINGLETON.getFriendManager().updateRemark("", "", new JMCallback<Object>() {
    @Override
    public void onSuccess(Object object) {

    }

    @Override
    public void onFailure(Exception e) {

    }
});
```

<!--
### 获取好友消息置顶状态
eg：
```java
// loginName 为对方的用户名
JMClient.SINGLETON.getFriendManager().getTop(loginName, new JMCallback<Boolean>() {
	@Override
	public void onSuccess(Boolean aBoolean) {
		// true 表示置顶,false 表示非置顶
	}

	@Override
	public void onFailure(Exception e) {

	}
});
```
-->

<!--
### 设置好友消息置顶
eg：
```java
// loginName 为对方的用户名
// top 表示置顶状态,0 表示非置顶,1 表示置顶
JMClient.SINGLETON.getFriendManager().setTop(loginName,top, new JMCallback<Integer>() {
	@Override
	public void onSuccess(Integer integer) {
		// 设置成功
	}

	@Override
	public void onFailure(Exception e) {
		// 设置失败
	}
});
```
-->

<!--
### 获取好友消息免打扰状态
eg：
```java
// loginName 为对方的用户名
JMClient.SINGLETON.getFriendManager().getDoNotDisturb(loginName, new JMCallback<Boolean>() {
	@Override
	public void onSuccess(Boolean aBoolean) {
		// true 表示免打扰,false 表示非免打扰
	}

	@Override
	public void onFailure(Exception e) {

	}
});
```
-->

<!--
### 设置好友消息免打扰
eg：
```java
// loginName 为对方的用户名
// doNotDisturb 表示置顶状态,0 表示非免打扰,1 表示免打扰
JMClient.SINGLETON.getFriendManager().setDoNotDisturb(loginName,doNotDisturb, new JMCallback<Integer>() {
	@Override
	public void onSuccess(Integer integer) {
		// 设置成功
	}

	@Override
	public void onFailure(Exception e) {
		// 设置失败
	}
});
```
-->

<!--
### 获取好友申请历史
eg：
```java
JMClient.SINGLETON.getFriendManager().getHistoryOfFriendRequest(new JMCallback<List<RequestHistoryBean>>() {
	@Override
	public void onSuccess(List<RequestHistoryBean> requestHistoryBeans) {

	}

	@Override
	public void onFailure(Exception e) {

	}
});
```
-->

* * *

## 群组管理

## 获取群组管理者 GroupChatManager

通过 JMClient 单例来获取 GroupChatManager

eg：

```java
JMClient.SINGLETON.getGroupChatManager();
```

### 获取我的群聊

eg：

```java
JMClient.SINGLETON.getGroupChatManager().getMyGroupChat(new JMCallback<List<GroupChatBean>>() {
	@Override
	public void onSuccess(List<GroupChatBean> groupChatBeanList) {
		// groupChatBeanList 即我的群聊列表
	}

	@Override
	public void onFailure(Exception e) {
		
	}
});
```

<!-- ### 加入群聊
```java
// localpart: 群聊唯一标识
JMClient.SINGLETON.getGroupChatManager().joinGroupChat(localpart, new JMCallback<Object>() {
	@Override
	public void onSuccess(Object object) {
		// 加入成功
	}

	@Override
	public void onFailure(Exception e) {
		// 加入失败
	}
});
``` -->

<!-- ### 退出群聊
```java
// localpart: 群聊唯一标识
JMClient.SINGLETON.getGroupChatManager().leaveGroupChat(localpart, , new JMCallback<Object>() {
	@Override
	public void onSuccess(Object object) {
		// 退出成功
	}

	@Override
	public void onFailure(Exception e) {
		// 退出失败
	}
});
``` -->

<!-- ### 设置是否免打扰
```java
// localpart: 群聊唯一标识
// doNotDisturb: 0 为非免打扰，1 为免打扰
JMClient.SINGLETON.getGroupChatManager().setDoNotDisturb(mGroupChatLocalpart, doNotDisturb, new JMCallback<Integer>() {
	@Override
	public void onSuccess(Integer integer) {
		
	}

	@Override
	public void onFailure(Exception e) {

	}
});
``` -->

<!-- ### 设置是否置顶
```java
// localpart: 群聊唯一标识
// top: 0 为不置顶，1 为置顶
JMClient.SINGLETON.getGroupChatManager().setTop(mGroupChatLocalpart, top, new JMCallback<Integer>() {
	@Override
	public void onSuccess(Integer integer) {
		
	}

	@Override
	public void onFailure(Exception e) {

	}
});
``` -->

<!-- ### 设置是否展示群昵称
```java
// localpart: 群聊唯一标识
// display: 0 为不展示群成员昵称，1 为展示
JMClient.SINGLETON.getGroupChatManager().setDisplayOccupantName(localpart, display, new JMCallback<Integer>() {
    @Override
    public void onSuccess(Integer integer) {

    }

    @Override
    public void onFailure(Exception e) {

    }
});
``` -->

### 获取群信息

eg：

```java
// localpart: 群聊唯一标识
JMClient.SINGLETON.getGroupChatManager().getGroupChat(localpart, new JMCallback<GroupChatBean>() {
    @Override
    public void onSuccess(GroupChatBean groupChatBean) {

    }

    @Override
    public void onFailure(Exception e) {

    }
});
```

<!-- ### 群主转让
```java
// localpart: 群聊唯一标识
// newOwnerUsername：新群主的 username
JMClient.SINGLETON.getGroupChatManager().transferOwner(localpart, newOwnerUsername);
``` -->

### 设置我在本群中的昵称

eg：

```java
// localpart: 群聊唯一标识
// occupantName: 我在本群中的昵称
JMClient.SINGLETON.getGroupChatManager().updateOccupantName(localpart, occupantName, new JMCallback<Object>() {
    @Override
    public void onSuccess(Object o) {

    }

    @Override
    public void onFailure(Exception e) {

    }
});
```

<!-- ### 创建群聊
```java
// name: 群名称，可以为空字符串
// occupants: 群成员 username 的集合
// imgUrl: 群头像地址，可以为空字符串
JMClient.SINGLETON.getGroupChatManager().createGroupChat(name, occupants, imgUrl, new JMCallback<String>() {
	@Override
	public void onSuccess(String s) {
		
	}

	@Override
	public void onFailure(Exception e) {

	}
});
``` -->

<!-- ### 邀请人进群
```java
// localpart: 群聊唯一标识
// invitees: 被邀请人的 username 的集合
JMClient.SINGLETON.getGroupChatManager().sendGroupChatInvitation(localpart, invitees, new JMCallback<String>() {
	@Override
	public void onSuccess(String s) {
		
	}

	@Override
	public void onFailure(Exception e) {

	}
});
``` -->

<!-- ### 设置群名称
只有群主可以设置群名称
```java
// localpart: 群聊唯一标识
// groupChatName：群名称
JMClient.SINGLETON.getGroupChatManager().updateGroupChatName(localpart, groupChatName, new JMCallback<Object>() {
    @Override
    public void onSuccess(Object o) {

    }

    @Override
    public void onFailure(Exception e) {

    }
});
``` -->

<!-- ### 设置群头像
只有群主可以设置群头像
```java
// localpart: 群聊唯一标识
// imgUrl: 群头像地址
JMClient.SINGLETON.getGroupChatManager().setGroupChatImage(localpart, imgUrl, new JMCallback<Object>() {
	@Override
	public void onSuccess(Object object) {
		
	}

	@Override
	public void onFailure(Exception e) {
		
	}
});
``` -->

<!-- ### 群踢人
只有群主可以进行踢人操作
```java
// localpart: 群聊唯一标识
// occupants: 被踢用户的 username 的集合
// reason: 被踢出群的原因
JMClient.SINGLETON.getGroupChatManager().kick(localpart, occupants, reason, new JMCallback<String>() {
	@Override
	public void onSuccess(String s) {
		
	}

	@Override
	public void onFailure(Exception e) {

	}
});
``` -->

* * *

## 命令

命令是 JM SDK 的一大特色，使用者输入正确规则的命令内容，如果接收方对该命令有处理的话，会执行相应的操作。

### 命令的输入规则

-   关键字 -参数名简称 [参数值]
-   关键字 --参数名全称 [参数值]

其中关键字必须要有，参数是不定参数，多个参数之间以横线(“-”)隔开，参数可以传值也可以不传值，参数名与参数值之间以空格(“ ”)隔开。

eg：

-   volume
-   volume -up
-   appInstall -i 1000
-   appInstall --name 亲子教育
-   appList -p 1 -pageSize 10

### 命令的处理规则

上面是命令的正确输入规则，按照上述规则输入的命令会通过验证，被接收方收到，但在收到时 SDK 中还会验证你是否可以处理该命令，想要处理一个命令，需定义一个命令处理者类，在该类中声明处理命令的方法，并使用 @Command 注解修饰该方法，用 @Param 注解修饰参数，最后在 JMClient.SINGLETON.init(this) 初始化的时候传入该命令处理者。

@Command 注解用于声明命令对应的处理方法。

属性：

-   keyword：命令关键字，同一个应用内的命令关键字不能重复，否则会覆盖。
-   englishAlias：英文别名，默认为空，如果设置了，则输入时输入的关键字为英文别名也会成功匹配，如上面的 volume 命令如果声明了英文别名为 v，则输入 v -up 也会成功匹配。
-   chineseAlias：中文别名，默认为空，同上。
-   letterAliasL：字母别名，默认为空，同上。
-   desc：命令功能描述，给开发者看的，相当于注释，无实际意义。

@Param 注解用于标识命令中的参数，考虑到有的参数是必传参数值的，有的参数可以没有值，所以使用 @Param 修饰的参数必须是 Java 包装类。

属性：

-   name：参数名简称，“ -”后面的被认为是参数名简称，如上面的 -i，i 会被认为是参数名简称。
-   fullName：参数名全称，“ --”后面的被认为是参数名全称，如上面的 --name，name 会被认为是参数名全称。
-   requireValue：该参数是否必须传值，默认为 true。当该值为 true 时，如果命令中传递了该参数而没有传参数值，而收到命令的验证就通不过，命令不会传递给命令处理者，当该值为 false 时，如果命令中传递了该参数，则收到的参数值为 defaultValue，如果没有设置 defaultValue，则值为参数对应类型的默认值，如 Integer 会为 0，Boolean 会为 false。
-   defaultValue：当参数不是必须传值时的默认值。
-   desc：参数描述，给开发者看的，相当于注释，无实际意义。

eg：

```java
public class CommandHandler {
    @Command(keyword = "volume", chineseAlias = "音量", letterAlias = "v", desc = "查询和调整设备端音量")
        public void volume(@Param(name = "up", requireValue = false, defaultValue = "10", desc = "音量+") Integer up
                , @Param(name = "down", requireValue = false, defaultValue = "10", desc = "音量-") Integer down
                , @Param(name = "adjust", desc = "音量调整到") Integer adjust
                , MessageBean messageBean) {
			// 如果命令中有 up 参数,且有值,则 up 值为输入值,如 volume -up 20
			// 如果命令中有 up 参数,但没有输入值,则 up 值为 defaultValue,如 volume -up
			// 如果命令中没有 up 参数,则 up 为 null,如 volume -down 50
			
            // do something
        }
}
```

然后在 JMClient 初始化的时候传入该 Handler

```java
// 初始化 Jeejio Message
try {
	// 如果需要处理命令，可以在 init 方法中传入命令处理者的 Class，命令规则详见：命令
	JMClient.SINGLETON.init(context,CommandHandler.class);
} catch (InstantiationException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
}
```

# 错误对照码

# 常见问题

# 更新日志

## v1.0.1

1.  优化长连接断开时的资源回收处理；
2.  增加登录冲突异常和 Token 异常；
3.  优化部分变量名。
