MTTextkitViewModel 主要实现的功能是满足 美图秀秀社区 各种描述（富文本）渲染的需求

目前支持 http(s) , @ , # 三种链接类型，支持点击跳转。<br>
其中支持链接类型使用 enum 控制，支持任意组合或者禁用所有链接。



提供的功能 <br>
1. 支持多行显示 （展开、收起）。需要设置一个收起状态时的行数 <br>
2. 支持内容全部展示 <br>
3. 支持内容里面添加 attachment (view) 。 比如  ![image10](./images/image10.png)



主要的类

1. TYLabel  继承自 UIView .负责内容显示 
2.  TYAsyncLayer 作为 TYLabel layer
3.  TYTextRender  持有
    1. NSLayoutManager
    2. NSTextContainer
    3. NSTextStorage <br>
            负责真实的绘制
4.     MTTextKitViewModel 负责社区业务相关的逻辑封装
    1. 三种链接支持
    2. 展开、收起逻辑处理 
        1. 计算收起状态下的内容，比如三行：首先取出三行内容，然后逆序遍历最后一行字符，计算合适的宽度留给展开、收起使用。同时处理 @mention ， #topic# 被截断时，响应用户操作等




使用方法
    
1.   创建一个 TYLabel 对象

	```     
	- (TYLabel *)kolDescLabel {
	    if (!_kolDescLabel) {
	        _kolDescLabel = [[TYLabel alloc] init];
	        _kolDescLabel.numberOfLines = 0;
	    }
	    return _kolDescLabel;
	}
	```

2. 生成一个TYTextRender 对象

	```
	- (void)generateKOLTextRender {
	    
	    NSString *identifyDesc = [self.user identityDesc];
	    if ( ![identifyDesc isKindOfClass:[NSString class]] || !identifyDesc.length) {
	        return;
	    }
	    identifyDesc = [NSString stringWithFormat:@"  %@",identifyDesc];
	    
	    UIFont *identifyFont  = [UIFont mt_mediumSystemFontOfSize:10];
	    UIFont *descFont = [UIFont mt_regularSystemFontOfSize:12];
	    
	    MTKOLIdentifyPrefixView *prefixView = [[MTKOLIdentifyPrefixView alloc]
	                                           initWithFont:identifyFont
	                                           text:NSLocalizedString(@"认证", nil)
	                                           textColor:[UIColor whiteColor]
	                                           identifyType: [self.user identifyTypeOfUser]];
	    TYTextAttachment *attachmentView = [[TYTextAttachment alloc]init];
	    attachmentView.view = prefixView;
	    attachmentView.size = CGSizeMake(prefixView.frame.size.width, prefixView.frame.size.height);
	    attachmentView.baseline = -4;
	    
	    NSMutableAttributedString *text = [[NSMutableAttributedString alloc]initWithString:identifyDesc];
	    text.ty_font = descFont;
	    text.ty_color = RGBCOLOR(44, 46, 48);
	    text.ty_lineHeightMultiple = 1.0;
	    text.ty_maximumLineHeight = 20;
	    text.ty_minimumLineHeight = 20;
	    text.ty_lineSpacing = 2;
	    
	    NSMutableAttributedString *mAttrStr = [[NSMutableAttributedString alloc] initWithAttributedString:[NSAttributedString attributedStringWithAttachment:attachmentView]];
	    NSMutableAttributedString *tempAttrStr = text ;
	    text = mAttrStr;
	    [text appendAttributedString:tempAttrStr];
	    TYTextRender *textRender = [[TYTextRender alloc]initWithAttributedText:text];
	    textRender.lineBreakMode = NSLineBreakByCharWrapping;
	    CGFloat width = SCREEN_WIDTH - [UserHomeHeaderLayoutTool elementMargin] * 2;
	    textRender.size = CGSizeMake(width, [textRender textSizeWithRenderWidth:width].height+2);
	    self.textRender = textRender;
	    self.textRenderHeight = textRender.size.height;
	}
	
	```

3. 给 Label 设置 textRender 


	```
	_textLabel.textRender = self.textRender;   
	_textLabel.height = self.textRenderHeight;
	```

以上上个步骤可以实现带有 attachment view 的普通文本格式 

下面演示一下秀秀社区使用的富文本组件的使用方式

1. 创建一个 MTTextKitModel 或者实现 MTTextKitDataProtocol 协议的类。 提供三个方法的实现

	```
- (NSString *_Nullable)tk_text;
- (NSOrderedSet<MTTextLinkInfo *> *_Nullable)tk_textLinks;
- (NSString *_Nullable)tk_modelID;	
	```
	
	其中 最简单的情况可以只提供 `- (NSString *_Nullable)tk_text;` 的返回值，其他两个方法返回空。
	其中列表类需要实现 `- (NSString *_Nullable)tk_modelID;` ,可以根据  tk_modelID 来时间 TYTextRender 数据复用(业务放实现)。