MTTextKitView Model 实现思路

<font color="red">
富文本server 下发内容格式   
text [123230912309123] text @name #topic# text
</font>

1. 基于开源代码 [TYText](https://github.com/12207480/TYText) 实现
2. `MTTextKitViewModel` 封装具体的业务需求
	1. 	@ 、# 、link 处理
	2. 展开、收起 逻辑处理
	
3. `MTTextKitDataProtocol` 定义 MTTextKitViewModel 使用的数据源数据结构的通用定义
<pre><code>- (NSString *_Nullable)tk_text;
\- (NSOrderedSet \<MTTextLinkInfo *\> *_Nullable)tk_textLinks;
\- (NSString *_Nullable)tk_modelID;
</code></pre>


	MTTextLinkInfo 定义 link 节点的内容
	<pre><code>@interface MTTextLinkInfo (CoreDataProperties)
	@property (nullable, nonatomic, copy) NSString *linkId;
	@property (nullable, nonatomic, copy) NSString *title;
	@property (nullable, nonatomic, copy) NSString *url;
	@property (nullable, nonatomic, copy) NSString *type;
	@end</code></pre>
 
4. `MTTextKitModel` 是一个实现  `MTTextKitDataProtocol` 协议的类。主要避免一些场景下，直接传入业务类(strong)，外部改动影响到 textkit 内容
5. `MTTextKitViewModel` 目前统一定义富文本样式：
	1. 颜色（文本、链接、链接背景色)    
	2. 字体 
	3. 宽 （各业务场景 UI 宽度）
	4. iconType 每个业务场景链接前的图片   
	5. enableTagType enum控制三种链接的生效类型。没有启用的链接，可以解析，颜色使用文本颜色，无链接点击事件
	6. disableCollapse 是否启用收起效果 、maxCollapsedNumberOfLines 控制启用收起效果，折叠状态下的行数
	7. numberOfLines 禁用收起效果时，控制行数
	8. 段间距
	9. ... 

6. `TYLabel` UIView 子类
	1. 可以单独设置 text 、textStorage 、字体颜色等属性 （Propertys）
	2. 如果设置 `TYTextRender *textRender` 则会忽略单独设置的属性 
7. `TYAsyncLayer` `CALayer` 子类。设置给 `TYLabel` 的 `+ (Class)layerClass`.
 关键代码如下：
 <pre><code>\- (void)displayAsync:(BOOL)asynchronously {
    \_\_strong id\<TYAsyncLayerDelegate\> delegate = \_asyncDelegate;
    TYAsyncLayerDisplayTask *task = [delegate newAsyncDisplayTask];
    if (task.willDisplay) {
        task.willDisplay(self);
    }
    CGSize size = self.bounds.size;
    if (!task.displaying || size.width < 0.1 || size.height < 0.1) {
        self.contents = nil;
        if (task.didDisplay) {
            task.didDisplay(self, YES);
        }
        return;
    }
    BOOL opaque = self.opaque;
    CGFloat scale = self.contentsScale;
    UIColor *backgroundColor = (opaque && self.backgroundColor) ? [UIColor colorWithCGColor:self.backgroundColor] : nil;
    if (asynchronously) {
        unsigned int value = [_sentinel value];
        TYSentinel *sentinel = _sentinel;
        BOOL (^isCancelled)(void) = ^BOOL(void){
            return value != [sentinel value];
        };
        dispatch_queue_t asyncDisplayQueue = YYDispatchQueueGetForQOS(NSQualityOfServiceUtility);
        dispatch_async(asyncDisplayQueue, ^{
            if (isCancelled()) {
                return ;
            }
            UIGraphicsBeginImageContextWithOptions(size, opaque, scale);
            CGContextRef context = UIGraphicsGetCurrentContext();
            if (!context) {
                UIGraphicsEndImageContext();
                return;
            }
            if (opaque) {
                CGContextSaveGState(context);
                CGContextSetFillColorWithColor(context,backgroundColor.CGColor);
                CGContextAddRect(context, CGRectMake(0, 0, size.width * scale, size.height * scale));
                CGContextFillPath(context);
                CGContextRestoreGState(context);
            }
            task.displaying(context, size, YES, isCancelled);
            if (isCancelled()) {
                UIGraphicsEndImageContext();
                dispatch\_async(dispatch\_get\_main\_queue(), ^{
                    if (task.didDisplay) task.didDisplay(self, NO);
                });
                return ;
            }
            UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
            UIGraphicsEndImageContext();
            if (isCancelled()) {
                dispatch\_async(dispatch\_get\_main\_queue(), ^{
                    if (task.didDisplay) task.didDisplay(self, NO);
                });
                return;
            }
            if (isCancelled()) {
                dispatch\_async(dispatch\_get\_main\_queue(), ^{
                    if (task.didDisplay) task.didDisplay(self, NO);
                });
            } else {
//                https://stackoverflow.com/questions/7140465/new-calayer-contents-does-not-display-until-view-is-resized  
//                [CATransaction flush];
//                https://fabric.io/meitu/ios/apps/com.meitu.mtxx/issues/5b61b3ce6007d59fcd7460d4?time=last-seven-days
//                crash
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.contents = (\_\_bridge id)(image.CGImage);
                    if (task.didDisplay) task.didDisplay(self, YES);
                });
            }
        });
    } else {
        [_sentinel increase];
        UIGraphicsBeginImageContextWithOptions(size, opaque, scale);
        CGContextRef context = UIGraphicsGetCurrentContext();
        if (opaque) {
            CGContextSaveGState(context);
            CGContextSetFillColorWithColor(context,backgroundColor.CGColor);
            CGContextAddRect(context, CGRectMake(0, 0, size.width * scale, size.height * scale));
            CGContextFillPath(context);
            CGContextRestoreGState(context);
        }
        task.displaying(context, size, NO,^{return NO;});
        UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        self.contents = (\_\_bridge id)(image.CGImage);
        if (task.didDisplay) {
            task.didDisplay(self, YES);
        };
    }
}</code></pre>

主要有两个关注点：   
1. asynchronously 异步绘制操作：设置为YES，可以在非主线程进行图片生成、然后切回主线程，进行图片显示.  
2.  `TYAsyncLayerDisplayTask *task = [delegate newAsyncDisplayTask];` 
获取 `TYLabel` 生成的渲染任务。其中 Task 对象只有三个 block 属性：  
<pre><code>@interface TYAsyncLayerDisplayTask : NSObject
@property (nullable, nonatomic, copy) void (^willDisplay)(CALayer *layer);
@property (nullable, nonatomic, copy) void (^displaying)(CGContextRef context,
 CGSize size, 
 BOOL isAsynchronously, 
 BOOL(^isCancelled)(void));
@property (nullable, nonatomic, copy) void (^didDisplay)(CALayer *layer, 
BOOL finished);
@end</code></pre>
3. `TYLabel: newAsyncDisplayTask` 实现如下
<pre><code>- (TYAsyncLayerDisplayTask *)newAsyncDisplayTask {
    __block TYTextRender *textRender = _textRender;
    __block NSTextStorage *textStorage = _textStorageOnRender;
    NSArray *attachments = _attachments;
    NSRange highlightRange  = _highlightRange;
    TYTextHighlight *textHighlight = _textHighlight;
    TYTextVerticalAlignment verticalAlignment = _verticalAlignment;
    NSInteger numberOfLines = _numberOfLines;
    NSLineBreakMode lineBreakMode = _lineBreakMode;
    __weak typeof(self) weakSelf = self;
    TYAsyncLayerDisplayTask *task = [[TYAsyncLayerDisplayTask alloc]init];
    // will display
    task.willDisplay = ^(CALayer * _Nonnull layer) {
        if (attachments) {
            NSSet *attachmentSet = textRender.attachmentViewSet;
            for (TYTextAttachment *attachment in attachments) {
                if (!attachmentSet || ![attachmentSet containsObject:attachment]) {
                    [attachment removeFromSuperView:self];
                }
            }
        }
        weakSelf.attachments = nil;
        weakSelf.textRenderOnDisplay = nil;
    };
    task.displaying = ^(CGContextRef  _Nonnull context, CGSize size, BOOL isAsynchronously, BOOL (^ _Nonnull isCancelled)(void)) {
        if (!textRender) {
            textRender = [[TYTextRender alloc]initWithTextStorage:textStorage];
            if (isCancelled()) return;
        }
        if (!textStorage) {
            return;
        }
        [self configureTextRender:textRender verticalAlignment:verticalAlignment numberOfLines:numberOfLines lineBreakMode:lineBreakMode];
        textRender.size = size;
        if (isCancelled()) return;
        [textRender setTextHighlight:textHighlight range:highlightRange];
        [textRender drawTextAtPoint:CGPointZero isCanceled:isCancelled];
    };
    task.didDisplay = ^(CALayer * _Nonnull layer, BOOL finished) {
        weakSelf.textRenderOnDisplay = textRender;
        NSArray *attachments = textRender.attachmentViews;
        if (!finished || !attachments) {
            if (attachments) {
                for (TYTextAttachment *attachment in attachments) {
                    [attachment removeFromSuperView:self];
                }
            }
            return ;
        }
        NSRange visibleRange = textRender.visibleCharacterRangeOnRender;
        NSRange truncatedRange = textRender.truncatedCharacterRangeOnRender;
        for (TYTextAttachment *attachment in attachments) {
            if (NSLocationInRange(attachment.range.location, visibleRange) && (truncatedRange.length == 0 || !NSLocationInRange(attachment.range.location, truncatedRange))) {
                if (textRender.maximumNumberOfLines > 0 && attachment.range.location != 0 && CGPointEqualToPoint(attachment.position, CGPointZero)) {
                    [attachment removeFromSuperView:self];
                }else {
                    CGRect rect = {attachment.position,attachment.size};
                    if (NSMaxRange(attachment.range) == NSMaxRange(visibleRange) && CGRectGetMaxX(rect) - CGRectGetWidth(self.frame) > 1) {
                        [attachment removeFromSuperView:self];
                    }else {
                        [attachment addToSuperView:self];
                        attachment.frame = rect;
                    }
                }
            }else {
                [attachment removeFromSuperView:self];
            }
        }
        weakSelf.attachments = attachments;
    };
    return task;
}</code></pre>



#### 处理点击事件 。 
`TYLabel` 实现 `- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event` 方法
<pre><code>- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    if (!\_textRenderOnDisplay || !\_textHighlight) {
        [self endLongPressTimer];
        if ([self.normalTextDelegate respondsToSelector:@selector(didTappedNormalTextInLabel:)]) {
            [self.normalTextDelegate didTappedNormalTextInLabel:self];
        }
        [super touchesEnded:touches withEvent:event];
        return;
    }
    UITouch *touch = touches.anyObject;
    CGPoint point = [touch locationInView:self];
    NSRange range = NSMakeRange(0, 0);
    if (\_delegateFlags.didTappedTextHighlight && _touchState == TYLabelTouchedStateTapped) {
        TYTextHighlight *textHighlight = [self textHighlightForPoint:point effectiveRange:&range];
        if (textHighlight == _textHighlight && NSEqualRanges(range, _highlightRange) ) {
            if (!textHighlight.disableUserInteraction) {
                \_textHighlight.userInfo = [self generateDictWithType:_textHighlight.userInfo];
                [_delegate label:self didTappedTextHighlight:_textHighlight];
            }
        }
    }
    [self endTouch];
}</code></pre> 

其中主要是 
`TYTextHighlight *textHighlight = [self textHighlightForPoint:point effectiveRange:&range];` 获取到点击 range 的 `TYTextHighlight`


<pre><code>//TYLabel
- (TYTextHighlight *)textHighlightForPoint:(CGPoint)point effectiveRange:(NSRangePointer)range {
	// 1.
    NSInteger index = [_textRenderOnDisplay characterIndexForPoint:point];
    if (index < 0) {
        return nil;
    }
    // 2.
    return [_textRenderOnDisplay.textStorage textHighlightAtIndex:index effectiveRange:range];
}

// TYRender
- (NSInteger)characterIndexForPoint:(CGPoint)point{
    CGRect textRect = _textRectOnRender;
    if (!CGRectContainsPoint(textRect, point) && !_editable) {
        return -1;
    }
    CGPoint realPoint = CGPointMake(point.x - textRect.origin.x, point.y - textRect.origin.y);
    CGFloat distanceToPoint = 1.0;
    NSUInteger index = [_layoutManager characterIndexForPoint:realPoint inTextContainer:_textContainer fractionOfDistanceBetweenInsertionPoints:&distanceToPoint];
    return distanceToPoint < 1 ? index : -1;
}


//NSLayoutManger
- (NSUInteger)characterIndexForPoint:(CGPoint)point 
inTextContainer:(NSTextContainer *)container 
fractionOfDistanceBetweenInsertionPoints:(nullable CGFloat *)partialFraction;


// 2. 
// NSTextStorage 的扩展方法
- (TYTextHighlight *)textHighlightAtIndex:(NSUInteger)index 
 effectiveRange:(nullable NSRangePointer)range {
    return [self ty_attribute:TYTextHighlightAttributeName 
    atIndex:index
     longestEffectiveRange:range];
}
</code></pre>


#### MTTextKitViewModel 实现

1. 初始化
<pre><code>\- (instancetype)initWithModel:(id<MTTextKitDataProtocol>)mediaDetail
                        label:(TYLabel *\_Nullable)contentLabel
                   styleModel:(MTTextKitStyleModel *\_Nullable)styleModel {
	    if (self = [super init]) {
	        self.mediaDetail = mediaDetail;
	        self.styleModel = styleModel;
	        //adjust ，如果style model 没有设置一些属性， 提供默认值：
	        // 比如颜色、字号、链接类型 是否允许展开、收起
	        [self generateTextRender];
	        // 触发setTextRender
	        self.contentLabel = contentLabel;
	    }
	    return self;
}</code></pre>
	
	调用时可以参考  `MTContentLabelHelper.swift` 里面 `public func fetchTextKitViewModel(for mediaDetail: MTTextKitDataProtocol, date: Date?) -> MTTextKitViewModel?` ,主要根据`MTTextKitDataProtocol` 中的 `- (NSString *_Nullable)tk_modelID;`  字段 ，进行缓存。

2. 设置 contentLabel 内部实现给 TYLabel 设置 render ，显示内容
	<pre><code>- (void)setContentLabel:(TYLabel *)contentLabel {
	    if (\_contentLabel  != contentLabel) {
	        \_contentLabel = contentLabel;
	    }
	    //修复 MTXX-27594 。 加在 if 里面，会有cell 重用问题，造成 _isExpand 状态污染 
	   	\_contentLabel.normalTextDelegate = self;
	    //禁止收起、展开， 使用配置的 numberOfLines
	    if (self.styleModel.disableCollapse && self.styleModel.numberOfLines) {
	        \_contentLabel.numberOfLines = self.styleModel.numberOfLines;
	        \_contentLabel.displaysAsynchronously = YES;
	    } else {
	        \_contentLabel.displaysAsynchronously = NO;
	    }
	    if (self.isExpand) {
	        if (self.styleModel.disableCollapse && self.styleModel.numberOfLines) {
	            \_contentLabel.textRender = self.numberOfLinesTextRender;
	        } else {
	            \_contentLabel.textRender = self.expandedTextRender;
	        }
	    } else {
	        \_contentLabel.textRender = self.collapsedTextRender;
	    }
}</code></pre>


3. 生成 TYTextRender
	<pre><code>-(void)generateTextRender {
    if (!self.mediaText) {
        return;
    }
    self.contentAttrString = [self.mediaText
                              mt\_parseUrlParam:self.mediaDetail.tk\_textLinks
                              imageNamed:[self urlIconImageNameForType:self.styleModel.iconType]];
    self.contentAttrString = [self configAttrStr:self.contentAttrString];
    NSAttributedString *foldSuffix = [MTTextKitViewModel foldTruncationStringWithFont:self.styleModel.font];
    NSAttributedString *unfoldSuffix = [MTTextKitViewModel unfoldTruncationStringWithFont:self.styleModel.font];
    BOOL isCollpased = false;
    NSInteger lineCount = 0;
    self.collapsedText = [MTTextKitViewModel getCollapsedTextForText:self.contentAttrString
                                                              suffix:unfoldSuffix
                                                           textLinks:self.mediaDetail.tk_textLinks
                                                            maxLines:self.styleModel.maxCollapsedNumberOfLines
                                                                font:self.styleModel.font
                                                               width:[self getWidth]
                                                         isCollapsed:&isCollpased
                                                  collapsedLineCount:&lineCount];    
    self.numberofLinesCollapsedTextLines = lineCount;
    if (isCollpased) {
        self.disableNormalTextTap = NO;
    } else {
        self.disableNormalTextTap = YES;
    }
    if (self.styleModel.disableNormalTextTap) {
        self.disableNormalTextTap = self.styleModel.disableNormalTextTap;
    }
    self.expandedText = [MTTextKitViewModel
                         getExpandedTextForText:self.contentAttrString
                         suffix:self.styleModel.disableCollapse ? nil: foldSuffix
                         textLinks:self.mediaDetail.tk_textLinks
                         maxLines:self.styleModel.maxCollapsedNumberOfLines
                         font:self.styleModel.font
                         width:[self getWidth]];
    self.hasSuffix = isCollpased;
    NSString *collapsedText = [self generateOriginFormatString:self.collapsedText.string
                                                        suffix:unfoldSuffix
                                                      isExpand:NO
                                                     hasSuffix:isCollpased];
    if ([collapsedText hasSuffix:@"@"] && self.hasSuffix) {
        collapsedText = [collapsedText substringToIndex:collapsedText.length-@"@".length];
    }
    
    //三个case 会不会遇到 bug？
    NSString *truncationTokenString = @"\u2026";
    if ([collapsedText hasSuffix:truncationTokenString]) {
        collapsedText = [collapsedText substringToIndex:collapsedText.length - truncationTokenString.length];
    }
    if ([collapsedText hasSuffix: @"\n"]) {
        collapsedText = [collapsedText substringToIndex:collapsedText.length - @"\n".length];
    }
    
    if ([collapsedText hasSuffix: @"\r"]) {
        collapsedText = [collapsedText substringToIndex:collapsedText.length - @"\r".length];
    }
    
    [self generateCollapsedTextRender:collapsedText];
    NSString *expandedText = [self generateOriginFormatString:self.expandedText.string
                                                       suffix:foldSuffix
                                                     isExpand:YES
                                                    hasSuffix:isCollpased];
    [self generateExpandedTextRender:expandedText];
    if (self.styleModel.disableCollapse && self.styleModel.numberOfLines) {
        if (self.styleModel.numberOfLinesSystemAddDots) {
            [self generateNumberofLinesTextRender: self.mediaText];
        } else {
            NSMutableDictionary *attributes = [NSMutableDictionary dictionary];
            attributes[NSForegroundColorAttributeName] = self.styleModel.textColor;
            attributes[NSFontAttributeName] = self.styleModel.font;
            NSMutableAttributedString *suffixString = [[NSMutableAttributedString alloc] initWithString:@"…" attributes:attributes];
            NSAttributedString *dotSuffix = suffixString;
            BOOL dotCollapsed = false;
            BOOL numberOfLinesIsCollapsed = NO;
            NSInteger numberOfLinesRealLine = 0;
            self.numberofLinesText = [MTTextKitViewModel
                                      getNumberOfLinesTextForText:self.contentAttrString
                                      suffix:nil
                                      textLinks:self.mediaDetail.tk_textLinks
                                      maxLines:self.styleModel.maxCollapsedNumberOfLines
                                      font:self.styleModel.font
                                      width:[self getWidth]
                                      isCollapsed:&numberOfLinesIsCollapsed
                                      collapsedLineCount:&numberOfLinesRealLine];
            NSString *dotText = [self generateOriginFormatString:self.numberofLinesText.string
                                                          suffix:dotSuffix
                                                        isExpand:NO
                                                       hasSuffix:dotCollapsed];
            if ([dotText hasSuffix:@"@"]) {
                dotText = [dotText substringToIndex:dotText.length - @"@".length];
            }
            [self generateNumberofLinesTextRender:dotText];
        }
    }
}</code></pre>