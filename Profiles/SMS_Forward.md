
## 适用范围

拥有两个电话号码、两部手机的同学。只随身携带其中的一部，另一部装在包的深处，或者放在其他不方便取的地方。

两部手机都安装了Tasker，并且两个Tasker上面都开启了本profile。

## 功能说明

1. 将备机收到的短信转发给主机。转发前会通过关键字的判断排除掉一些无用的营销信息。
	
		大diao 萌妹、激情 对射 更多，福利 加V信：uvw789246xyz 退订T
	
	触发关键字的不转发

		我感冒了，好难受 ><
	
	未包含关键字的正常转发

2. 备机进入等待状态，此时可以用主机回复备机可进行如下操作：
		
		Re: 多喝热水
	
	用备机的号码向原发件人**回复**“多喝热水”。
	
		Fw: 13000007890
	
	用备机的号码将刚才的短信**转发**给13000007890。
	
	注意这里我放了一步判断，限制只能转发给13?~19?开头的11位手机号。
	
		多喝热水，下班我就过去
	
	直接回复**无效**。任意非 ~~```Re:```~~ 、非 ~~```Fw:```~~ 开头的内容都会使备机退出等待状态。
	
	因此，如果你还有利用其他短信指令激活的profile，本profile与它们**不互斥**。

3. “等待状态”只是一种方便理解的说法。实际上并不是一种持续的状态。无论退出与否，都不会影响性能与耗电。

4. 对称的profile设计。主机和备机只存在于使用者的观念上，切换主备机不需要重新设定profile。~~换SIM卡切换的除外~~

## 配置思路

首先要设计好指令短信格式。

回复指令：
```^(?i)re:\s*[\s\S]*$```

re: + 任意个空格 + 任意个字符

转发指令：
```^(?i)fw:\s*(1[3-9][0-9])\d{8}$```

fw: + 任意个空格 + 13?~19?开头的11位手机号码


Tasker内置了两个可以直接使用的短信变量：短信内容 ```%SMSRB``` 、发件人 ```%SMSRF```。

但是这两个内置变量都是被最新的一条短信赋值的。当备机收到主机发送的指令之后，原内容就被指令短信覆盖了。

那么怎么实现回复/转发呢？

我的思路是，备机在做第一步的转发时，顺便用这两个变量给两个自定义变量赋值：原短信内容 ```%SMSSB``` 、原发件人 ```%SMSST```。

当备机第二步收到主机发来的回复指令时，只需要处理新的指令短信 ```%SMSRB```：首先删掉开头的指令 ```re:``` ，然后用指令短信剩下的部分替换的正文部分```%SMSSB``` 。```%SMSST``` 仍然保持原发件人不变。

同理判断接收到转发指令时，则是替换修改转发收件人 ```%SMSST``` ，而原短信正文 ```%SMSSB``` 不变。

这样无论回复还是转发，只要将正文变量 ```%SMSSB``` 发送给号码变量 ```%SMSST``` ，即达成目的。

完整描述如下。配置时需要把其中的13000001234换成自己**另一台手机**的号码 ~~而**不是**本机号码~~

```
Profile: SMS转发
	Priority: 6 Enforce: no
	Event: Received Text [ Type:SMS Sender:* Content:* ]
Enter: SMS转发
	A1: If [ %SMSRF !~R 13000001234 & %SMSRB !~R (退订|回复?)(TD?|N|QX|BK) ]
	A2: Send SMS [ Number:13000001234 Message:%SMSRB From: %SMSRF Store In Messaging App:Off ] 
	A3: Variable Set [ Name:%SMSST To:%SMSRF Recurse Variables:Off Do Maths:Off Append:Off ] 
	A4: Variable Set [ Name:%SMSSB To:%SMSRB Recurse Variables:Off Do Maths:Off Append:Off ] 
	A5: Else If [ %SMSRF ~R 13000001234 ]
	A6: If [ %SMSRB ~R ^(?i)re:\s*[\s\S]*$ ]
	A7: Variable Set [ Name:%SMSSB To:%SMSRB Recurse Variables:Off Do Maths:Off Append:Off ] 
	A8: Variable Search Replace [ Variable:%SMSSB Search:^re:\s* Ignore Case:On Multi-Line:Off One Match Only:Off Store Matches In: Replace Matches:On Replace With: ] 
	A9: Else If [ %SMSRB ~R ^(?i)fw:\s*(1[3-9][0-9])\d{8}$ ]
	A10: Variable Set [ Name:%SMSST To:%SMSRB Recurse Variables:Off Do Maths:Off Append:Off ] 
	A11: Variable Search Replace [ Variable:%SMSST Search:^fw:\s* Ignore Case:On Multi-Line:Off One Match Only:Off Store Matches In: Replace Matches:On Replace With: ] 
	A12: Else 
	A13: Goto [ Type:Action Label Number:1 Label:JUMP ] 
	A14: End If 
	A15: Send SMS [ Number:%SMSST Message:%SMSSB Store In Messaging App:Off ] 
	<JUMP>
	A16: Variable Clear [ Name:%SMSST Pattern Matching:Off Local Variables Only:Off ] 
	A17: Variable Clear [ Name:%SMSSB Pattern Matching:Off Local Variables Only:Off ] 
	A18: End If 
```



## XML文件下载

https://raw.githubusercontent.com/feeshy/tasker_profiles_share/master/Profiles/SMS_Forward.xml
