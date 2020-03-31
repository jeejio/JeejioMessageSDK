#添加依赖
请将如下 jar 包放入 libs 文件夹中
- j-message.jar
- smack-core.jar
- smack-extensions.jar
- smack-im.jar
- smack-sasl-provided.jar
- smack-tcp.jar

除此以外，JM 还用到了 OkHttp 请求,Gson 解析及阿里云 Oss 文件存储，所以还需在 build.gradle 中添加如下依赖：
```
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

#登录注册
##初始化SDK
登录前需做 SDK 的初始化。
```java
// 初始化 Oss
OssHelper.SINGLETON.init(context);
// 初始化 Jeejio Message
try {
	// 如果需要处理命令，可以在 init 方法中传入命令处理者的 Class，命令规则详见：命令
	JMClient.SINGLETON.init(context);
} catch (InstantiationException e) {
} catch (IllegalAccessException e) {
}
```
##注册
接入 SDK 的应用是提供给物栖智能设备的应用，无需注册，当该应用下载到物栖智能设备上会自动完成注册操作。

##登录
SDK 中所有操作均需在登录之后进行。
```java
// 第三方应用用户登录
// machineCodeAndPackageName: machineCode_包名
JMClient.SINGLETON.applicationLogin(machineCodeAndPackageName, new ILoginListener() {
	@Override
	public void onSuccess() {
		// 登录成功
	}

	@Override
	public void onFailure(Exception e) {
		// 登录失败
	}
});

// 用户名+密码登录模式
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
});
```
##登出
虽然即使不调用登出方法，服务器也会在 300s 后断开闲置客户端，但为了避免丢失离线消息，需在应用退出时需调用登出方法。
```java
JMClient.SINGLETON.logout();
```
##添加连接状态监听
当监听器的 connectionClosedOnError() 方法被回调时，如果错误信息中包含 conflict 关键字，则表示是同一个账号在另一个客户端上登录，当前客户端会被踢下线。
```java
JMClient.SINGLETON.addConnectionListener(new ConnectionListener() {
	@Override
	public void connected(XMPPConnection connection) {
		// 连接成功
	}

	@Override
	public void authenticated(XMPPConnection connection, boolean resumed) {
		// 登录成功
	}

	@Override
	public void connectionClosed() {
		// 连接异常关闭
	}

	@Override
	public void connectionClosedOnError(Exception e) {
		if (e.getMessage().contains("conflict")) {
			// 登录冲突
		}
	}
});
```
##手动重连
服务端为了节省资源，通常会关闭闲置客户端，虽然客户端与服务端有心跳机制进行保活，但仍然存在会话失效的情况，这时候客户端在 ping 服务器发现没有响应（3 次）的时候，会关闭连接。
客户端可以监听到连接关闭，这时候可以选择调用 JMClient.SINGLETON.reconnect() 方法进行手动重连。
eg：
```java
JMClient.SINGLETON.applicationLogin(new ILoginListener() {
    @Override
    public void onSuccess() {
        ShowLogUtil.logi("登录成功");
        // 添加连接监听
        JMClient.SINGLETON.addConnectionListener(new ConnectionListener() {
            @Override
            public void connected(XMPPConnection xmppConnection) {
                // 连接成功
            }

            @Override
            public void authenticated(XMPPConnection xmppConnection, boolean b) {
                // 验证成功
            }

            @Override
            public void connectionClosed() {
                // 连接被关闭，手动重连
                JMClient.SINGLETON.reconnect();
            }

            @Override
            public void connectionClosedOnError(Exception e) {
                // 连接异常关闭
            }
        });
    }

    @Override
    public void onFailure(Exception e) {
        // 登录失败
    }
});
```
#消息
发送文本、语音、命令等消息（单聊、群聊通用）
##获取消息管理者
```java
JMClient.SINGLETON.getMessageManager();
```

##接收消息
在需要监听消息的界面添加监听器，单聊、群聊通用，根据接收到的 message 的不同类型做不同的业务逻辑即可。
```java
JMClient.SINGLETON.getMessageManager().addChatListener(new IChatListener() {
	@Override
	public void onReceive(MessageBean message) {
		// 收到消息
	}
});
```

##发送消息
###发送文本消息
```java
// toUsername:单聊为对方 id，群聊为群 id
MessageBean message = MessageBean.createTextMessage(toUsername,content);
// 群聊 Message
// MessageBean message = MessageBean.createTextMessage(toUsername,content);
// message.setType(MessageBean.Type.GROUP_CHAT.getValue())
// 或
// MessageBean message = MessageBean.createTextMessage(toUsername,content,MessageBean.Type.GROUP_CHAT);
JMClient.SINGLETON.getMessageManager().sendMessage(message);
```
如果需要监听发送结果，则调用
```java
JMClient.SINGLETON.getMessageManager().sendMessage(message, new JMCallback<MessageBean>() {
	@Override
	public void onSuccess(MessageBean messageBean) {
		// 发送成功
	}

	@Override
	public void onFailure(Exception e) {
		// 发送失败会回调该方法，目前只有双方不为好友时会发送失败
	}
});
```

###发送语音消息
```java
// toUsername:单聊为对方 id，群聊为群 id
// filePath:语音文件路径
// time:语音时长
MessageBean message = MessageBean.createVoiceMessage(toUsername,filePath,time);
// 群聊 Message 请参考文本消息
JMClient.SINGLETON.getMessageManager().sendMessage(message);
```
如果需要监听发送结果，请参考文本消息

###发送命令消息
命令本质上与普通文本并无区别，所以在 Message 的创建上与普通文本一样，只是调用的创建方法不同。
```java
// toUsername:单聊为对方 id，群聊为群 id
// content:命令内容
MessageBean message = MessageBean.createCommandMessage(toUsername,content);
// 群聊 Message 请参考文本消息
JMClient.SINGLETON.getMessageManager().sendMessage(message);
```
如果需要监听发送结果，请参考文本消息

###发送扩展消息
如果以上方法仍满足不了需求，可以设置 Message 的 extend 属性，作为扩展字段，自己定义解析规则，推荐 json 格式的字符串。
```java
MessageBean message = new MessageBean(content, toUsername);
// 群聊 Message 请参考文本消息
message.setExtend("{"key":"value"}");


#好友管理
##获取好友管理者
```java
JMClient.SINGLETON.getFriendManager();
```

##监听好友状态
```java
JMClient.SINGLETON.getFriendManager().addFriendListener(new IFriendListener() {
	@Override
	public void onOnline(String username) {
		// 好友上线
	}

	@Override
	public void onOffline(String username) {
		// 好友离线
	}

	@Override
	public void onSubscribe(String username, String status) {
		// 收到好友请求
	}

	@Override
	public void onSubscribed(String username) {
		// 有人同意了你的好友申请
	}

	@Override
	public void onUnsubscribed(String username) {
		// 有人拒绝了你的好友申请
	}

	@Override
	public void onUnsubscribe(String username) {
		// 被删除
	}

	@Override
	public void onError() {
		// 发生错误
	}
});
```

##获取好友列表
由于 JM 的用户系统是区分类型的，分为普通用户类型（userType 为 1）、设备用户类型（userType 为 2）、应用用户类型（userType 为 3），所以在查询好友列表时需分别查询。
```java
// 默认从缓存中获取
JMClient.SINGLETON.getFriendManager().getList(userType, new JMCallback<List<UserBean>>() {
	@Override
	public void onSuccess(List<UserBean> userBeanList) {
		
	}

	@Override
	public void onFailure(Exception e) {
		
	}
});

// 从服务器获取
JMClient.SINGLETON.getFriendManager().getListFromServer(userType, new JMCallback<List<UserBean>>() {
	@Override
	public void onSuccess(List<UserBean> userBeanList) {
		
	}

	@Override
	public void onFailure(Exception e) {
		
	}
});
```

##获取单个好友
```java
JMClient.SINGLETON.getFriendManager().getContactInfo(userType, username, new JMCallback<UserBean>() {
	@Override
	public void onSuccess(UserBean userBean) {

	}

	@Override
	public void onFailure(Exception e) {

	}
});
```

##添加好友
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

##同意添加好友
```java
// loginName 为待添加好友的用户名
JMClient.SINGLETON.getFriendManager().agreeAdd(loginName);
```

##删除好友
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

##判断与对方是否互为好友
```java
// 如果是好友,返回对应好友信息,否则返回 null
FriendBean friend = JMClient.SINGLETON.getFriendManager().isFriend(loginName);
```

##修改备注
```java
// loginName 为对方的用户名
// remarkName 为新备注
JMClient.SINGLETON.getFriendManager().updateRemarkName(loginName, remarkName);
```

##获取好友消息置顶状态
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

## 设置好友消息置顶
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

##获取好友消息免打扰状态
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

## 设置好友消息免打扰
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

##获取好友申请历史
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

#群组管理
##获取群组管理者
```java
JMClient.SINGLETON.getGroupChatManager();
```

##添加群聊监听
```java
// localpart: 群聊唯一标识
JMClient.SINGLETON.getGroupChatManager().addGroupChatListener(mGroupChatLocalpart, new GroupChatManager.GroupChatListeners(new ParticipantStatusListener() {
	@Override
	public void joined(EntityFullJid participant) {
		// 有新的群成员加入房间时回调该方法
	}

	@Override
	public void left(EntityFullJid participant) {
		// 有群成员退出房间时回调该方法
	}

	@Override
	public void kicked(EntityFullJid participant, Jid actor, String reason) {
		// 有群成员被踢出房间时回调该方法
	}

	@Override
	public void voiceGranted(EntityFullJid participant) {

	}

	@Override
	public void voiceRevoked(EntityFullJid participant) {

	}

	@Override
	public void banned(EntityFullJid participant, Jid actor, String reason) {

	}

	@Override
	public void membershipGranted(EntityFullJid participant) {

	}

	@Override
	public void membershipRevoked(EntityFullJid participant) {

	}

	@Override
	public void moderatorGranted(EntityFullJid participant) {

	}

	@Override
	public void moderatorRevoked(EntityFullJid participant) {

	}

	@Override
	public void ownershipGranted(EntityFullJid participant) {

	}

	@Override
	public void ownershipRevoked(EntityFullJid participant) {

	}

	@Override
	public void adminGranted(EntityFullJid participant) {

	}

	@Override
	public void adminRevoked(EntityFullJid participant) {

	}

	@Override
	public void nicknameChanged(EntityFullJid participant, String newNickname) {

	}
}, new UserStatusListener() {
	@Override
	public void kicked(Jid room, Jid actor, String reason) {
		// 自己被踢出房间时回调
	}

	@Override
	public void voiceGranted() {

	}

	@Override
	public void voiceRevoked() {

	}

	@Override
	public void banned(Jid actor, String reason) {

	}

	@Override
	public void membershipGranted() {

	}

	@Override
	public void membershipRevoked() {

	}

	@Override
	public void moderatorGranted() {

	}

	@Override
	public void moderatorRevoked() {

	}

	@Override
	public void ownershipGranted() {

	}

	@Override
	public void ownershipRevoked() {

	}

	@Override
	public void adminGranted() {

	}

	@Override
	public void adminRevoked() {

	}

	@Override
	public void roomDestroyed(MultiUserChat alternateMUC, String reason) {

	}

	@Override
	public void naturalNameUpdated(String groupchat, String name) {
		// 群聊名称被修改
	}

	@Override
	public void roomImageUpdated(String groupchat, String imgUrl) {
		// 群聊头像被修改
	}

	@Override
	public void nicknameChanged(String nickname) {
		// 我在群聊中的昵称被修改
	}
}));
```

##移除群聊监听
```java
// localpart: 群聊唯一标识
JMClient.SINGLETON.getGroupChatManager().removeGroupChatListener(localpart,groupChatListeners);
```

##加入群聊
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
```

##退出群聊
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
```

##获取群成员列表
```java
// localpart: 群聊唯一标识
JMClient.SINGLETON.getGroupChatManager().getGroupChatOccupants(localpart, new JMCallback<List<GroupChatOccupantBean>>() {
	@Override
	public void onSuccess(List<GroupChatOccupantBean> groupChatOccupantBeans) {
		// groupChatOccupantBeans 即群成员列表
	}

	@Override
	public void onFailure(Exception e) {
		
	}
});
```

##获取群成员数量
```java
// localpart: 群聊唯一标识
JMClient.SINGLETON.getGroupChatManager().getGroupChatOccupantsCount(localpart, new JMCallback<Integer>() {
	@Override
	public void onSuccess(Integer integer) {
		// integer 即群成员数量
	}

	@Override
	public void onFailure(Exception e) {

	}
});
```

##获取群聊默认名称
群聊名称允许为空，当名称为空时，可以通过该方法获取一个默认名称用于展示，默认名称类似微信，使用前几个群成员的昵称拼接。
```java
// localpart: 群聊唯一标识
JMClient.SINGLETON.getGroupChatManager().getGroupChatNameDefault(localpart, new JMCallback<String>() {
	@Override
	public void onSuccess(String s) {
                            groupChatBean.setNameDefault(s);
                            // 将最新群聊信息保存到本地
                            jmCallback.onSuccess(new GroupChatDaoImpl().createOrUpdate(groupChatBean));
                        }

                        @Override
                        public void onFailure(Exception e) {
                            // 将最新群聊信息保存到本地
                            jmCallback.onSuccess(new GroupChatDaoImpl().createOrUpdate(groupChatBean));
                        }
                    });
```

##获取群主的 username
```java
// localpart: 群聊唯一标识
JMClient.SINGLETON.getGroupChatManager().getOwner(localpart, new JMCallback<String>() {
	@Override
	public void onSuccess(String owner) {
		// owner 即群主的 username
	}

	@Override
	public void onFailure(Exception e) {
		
	}
});
```

##设置群名称
只有群主可以设置群名称
```java
// localpart: 群聊唯一标识
// groupChatName：群名称
JMClient.SINGLETON.getGroupChatManager().setGroupChatName(localpart, groupChatName);
```

##设置是否展示群昵称
```java
// localpart: 群聊唯一标识
// display: 0 为不展示群成员昵称，1 为展示
JMClient.SINGLETON.getGroupChatManager().setDisplayOccupantName(localpart, display, new JMCallback() {
	@Override
	public void onSuccess(Object object) {
		
	}

	@Override
	public void onFailure(Exception e) {
		
	}
});
```

##设置群头像
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
```

##获取我的群聊
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

##获取群信息
群信息在从服务器拉取后会在本地缓存，所以可以选择从本地获取还是从网络获取
```java
// localpart: 群聊唯一标识
JMClient.SINGLETON.getGroupChatManager().getGroupChatInfo(groupChatLocalPart, new JMCallback<GroupChatBean>() {
	@Override
	public void onSuccess(GroupChatBean groupChatBean) {
		// groupChatBean 即群信息
	}

	@Override
	public void onFailure(Exception e) {

	}
});

// localpart: 群聊唯一标识
// checkDb: 是否优先查询数据库，true 表示是，false 则直接查询网络
JMClient.SINGLETON.getGroupChatManager().getGroupChatInfo(groupChatLocalPart, checkDb, new JMCallback<GroupChatBean>() {
	@Override
	public void onSuccess(GroupChatBean groupChatBean) {
		// groupChatBean 即群信息
	}

	@Override
	public void onFailure(Exception e) {

	}
});
```

##设置是否免打扰
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
```

##设置是否置顶
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
```

##群主转让
```java
// localpart: 群聊唯一标识
// newOwnerUsername：新群主的 username
JMClient.SINGLETON.getGroupChatManager().transferOwner(localpart, newOwnerUsername);
```

##设置我在本群中的昵称
```java
// localpart: 群聊唯一标识
// occupantName: 我在本群中的昵称
JMClient.SINGLETON.getGroupChatManager().setOccupantName(mGroupChatLocalpart,content);
```

##获取用户在群聊中的昵称
```java
// localpart: 群聊唯一标识
// username: 查询目标用户
JMClient.SINGLETON.getGroupChatManager().getOccupantName(localpart, username, new JMCallback<String>() {
	@Override
	public void onSuccess(String occupantName) {

	}

	@Override
	public void onFailure(Exception e) {

	}
});
```

##创建群聊
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
```

##邀请人进群
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
```

##群踢人
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
```

#命令
命令是 JM SDK 的一大特色，使用者输入正确规则的命令内容，如果接收方对该命令有处理的话，会执行相应的操作。
##命令的输入规则
关键字[ -参数名简称 参数值]...
或
关键字[ --参数名全称 参数值]...
其中关键字必须要有，参数是不定参数，多个参数之间以“ _”隔开，参数可以传值也可以不传值，参数名与参数值之间以“ ”隔开。
eg：
- volume
- volume -up
- appInstall -i 1000
- appInstall --name 亲子教育
- appList -p 1 -pageSize 10

##命令的处理规则
上面是命令的正确输入规则，按照上述规则输入的命令会通过验证，被接收方收到，但在收到时 SDK 中还会验证你是否可以处理该命令，想要处理一个命令，需定义一个命令处理者类，在该类中声明处理命令的方法，并使用 @Command 注解修饰该方法，用 @Param 注解修饰参数，最后在 JMClient.SINGLETON.init(this) 初始化的时候传入该命令处理者。
@Command 注解用于声明命令对应的处理方法。
属性：
	keyword：命令关键字，同一个应用内的命令关键字不能重复，否则会覆盖。
	englishAlias：英文别名，默认为空，如果设置了，则输入时输入的关键字为英文别名也会成功匹配，如上面的 volume 命令如果声明了英文别名为 v，则输入 v -up 也会成功匹配。
	chineseAlias：中文别名，默认为空，同上。
	letterAliasL：字母别名，默认为空，同上。
	desc：命令功能描述，给开发者看的，相当于注释，无实际意义。

------------


@Param 注解用于标识命令中的参数，考虑到有的参数是必传参数值的，有的参数可以没有值，所以使用 @Param 修饰的参数必须是 Java 包装类。
属性：
	name：参数名简称，“ -”后面的被认为是参数名简称，如上面的 -i，i 会被认为是参数名简称。
	fullName：参数名全称，“ --”后面的被认为是参数名全称，如上面的 --name，name 会被认为是参数名全称。
	requireValue：该参数是否必须传值，默认为 true。当该值为 true 时，如果命令中传递了该参数而没有传参数值，而收到命令的验证就通不过，命令不会传递给命令处理者，当该值为 false 时，如果命令中传递了该参数，则收到的参数值为 defaultValue，如果没有设置 defaultValue，则值为参数对应类型的默认值，如 Integer 会为 0，Boolean 会为 false。
	defaultValue：当参数不是必须传值时的默认值。
	desc：参数描述，给开发者看的，相当于注释，无实际意义。
	

------------

发你   
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
} catch (IllegalAccessException e) {
}
```