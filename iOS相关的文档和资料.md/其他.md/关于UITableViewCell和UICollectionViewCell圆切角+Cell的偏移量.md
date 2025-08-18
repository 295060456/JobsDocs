# 关于UITableViewCell和UICollectionViewCell圆切角+Cell的偏移量

[toc]

## 一些公共的

```objective-c
@implementation UITableViewCell (BaseCellProtocol)
/// 获取这个UITableViewCell所承载的UITableView
-(UITableView *)jobsGetCurrentTableView{
    return (UITableView *)self.superview;
}
/// 获取当前的UITableViewCell对应的indexPath
-(NSIndexPath *)jobsGetCurrentIndexPath{
    return [(UITableView *)self.jobsGetCurrentTableView indexPathForCell:self];
}
/// 获取当前的UITableViewCell对应的section
-(NSInteger)jobsGetCurrentNumberOfSections{
    return [self.jobsGetCurrentTableView numberOfSections];
}
/// 获取当前的UITableViewCell对应的row
-(NSInteger)jobsGetCurrentNumberOfRowsInSection{
    return [self.jobsGetCurrentTableView numberOfRowsInSection:self.jobsGetCurrentIndexPath.section];
}

@end
```

```objective-c
@implementation UICollectionViewCell (UICollectionViewCellProtocol)
/// 获取这个UICollectionViewCell所承载的UICollectionView
-(UICollectionView *)jobsGetCurrentCollectionView{
    return (UICollectionView *)self.superview;
}
/// 获取当前的UICollectionViewCell对应的indexPath
-(NSIndexPath *)jobsGetCurrentIndexPath{
    return [(UICollectionView *)self.jobsGetCurrentCollectionView indexPathForCell:self];
}
/// 获取当前的UICollectionViewCell对应的section
-(NSInteger)jobsGetCurrentNumberOfSections{
    return [self.jobsGetCurrentCollectionView numberOfSections];
}
/// 获取当前的UICollectionViewCell对应的item
-(NSInteger)jobsGetCurrentNumberOfItemsInSection{
    return [self.jobsGetCurrentCollectionView numberOfItemsInSection:self.jobsGetCurrentIndexPath.section];
}

@end
```

```objective-c
/// 附加偏移量以后的大小
-(CGRect)dx:(CGFloat)dx dy:(CGFloat)dy{
    /**
     CGRectInset会进行平移和缩放两个操作;CGRectOffset做的只是平移
     CGRectInset(CGRect rect, CGFloat dx, CGFloat dy),
     三个参数:
     rect：待操作的CGRect；
     dx：为正数时，向右平移dx，宽度缩小2dx。为负数时，向左平移dx，宽度增大2dx；
     dy：为正数是，向下平移dy，高度缩小2dy。为负数是，向上平移dy，高度增大2dy。
     */
    return CGRectInset(self.bounds,dx,dy);/// 获取显示区域大小
}
```

* 这种方式，只能统一的设置同一种样式，并不能满足自定义需求

```objective-c
cell.contentView.layer.cornerRadius = cell.layer.cornerRadius = JobsWidth(8);
cell.contentView.layer.borderWidth = cell.layer.borderWidth = JobsWidth(1);
cell.contentView.layer.borderColor = cell.layer.borderColor = RGBA_COLOR(255, 225, 144, 1).CGColor;
cell.contentView.layer.masksToBounds = cell.layer.masksToBounds = YES;
```

## 1、关于UITableView.UITableViewCell

### 1.1、圆切角

#### 1.1.1、以section为单位，每个section的第一行和最后一行的cell圆角化处理【cell之间没有分割线】

```objective-c
/// 以section为单位，每个section的第一行和最后一行的cell圆角化处理【cell之间没有分割线】
-(__kindof CALayer *)roundedCornerFirstAndLastCellByTableView:(UITableView *)tableView
                                                    indexPath:(NSIndexPath *)indexPath
                                                  layerConfig:(JobsLocationModel *)layerConfig{
    // 关闭分割线
    tableView.separatorStyle = UITableViewCellSeparatorStyleNone;
    @jobs_weakify(self)
    if (jobsZeroSizeValue(layerConfig.roundingCornersRadii)) layerConfig.roundingCornersRadii = CGSizeMake(JobsWidth(10.0), JobsWidth(10.0));
    // 当前 section 的总行数
    NSInteger numberOfRows = tableView.rowsInSection(indexPath.section);
    // 初始化将要设置的圆角类型
    UIRectCorner corner = 0;
    if (numberOfRows == 1) {
        corner = UIRectCornerAllCorners;
    } else if (indexPath.row == 0) {
        corner = UIRectCornerTopLeft | UIRectCornerTopRight;
    } else if (indexPath.row == numberOfRows - 1) {
        corner = UIRectCornerBottomLeft | UIRectCornerBottomRight;
    } else {
        // 中间 cell，无圆角和边框
        self.layer.mask = nil;
        // 移除边框 Layer（避免复用残留）
        NSArray *sublayers = self.layer.sublayers.copy;
        for (CALayer *sublayer in sublayers) {
            if ([sublayer.name isEqualToString:@"rounded-border-layer"]) {
                [sublayer removeFromSuperlayer];
            }
        }return self.layer;
    }
    // 设置圆角路径
    UIBezierPath *maskPath = [UIBezierPath bezierPathWithRoundedRect:self.bounds
                                                   byRoundingCorners:corner
                                                         cornerRadii:layerConfig.roundingCornersRadii];
    self.layer.mask = jobsMakeCAShapeLayer(^(__kindof CAShapeLayer * _Nullable layer) {
        @jobs_strongify(self)
        layer.frame = self.bounds;
        layer.path = maskPath.CGPath;
    });
    // 添加边框 Layer（可选）
    if (layerConfig.layerBorderCor && layerConfig.borderWidth > 0) {
        // 移除之前旧的 border layer，避免复用叠加
        NSArray *sublayers = self.layer.sublayers.copy;
        for (CALayer *sublayer in sublayers) {
            if ([sublayer.name isEqualToString:@"rounded-border-layer"]) {
                [sublayer removeFromSuperlayer];
            }
        }
        self.layer.addSublayer(jobsMakeCAShapeLayer(^(__kindof CAShapeLayer * _Nullable borderLayer) {
            @jobs_strongify(self)
            borderLayer.frame = self.bounds;
            borderLayer.path = maskPath.CGPath;
            borderLayer.strokeColor = layerConfig.layerBorderCor.CGColor;
            borderLayer.fillColor = UIColor.clearColor.CGColor;
            borderLayer.lineWidth = layerConfig.borderWidth;
            borderLayer.name = @"rounded-border-layer";
        }));
    }return self.layer;
}
```

且不描边顶部

```objective-c
/// 以 section 为单位，仅对每个 section 的最后一行 cell 做圆角处理（cell 之间没有分割线），且不描边顶部
-(__kindof CALayer *)roundedCornerLastCellByTableView:(UITableView *)tableView
                                            indexPath:(NSIndexPath *)indexPath
                                          layerConfig:(JobsLocationModel *)layerConfig {

    tableView.separatorStyle = UITableViewCellSeparatorStyleNone;
    NSInteger numberOfRows = [tableView numberOfRowsInSection:indexPath.section];

    @jobs_weakify(self)

    if (jobsZeroSizeValue(layerConfig.roundingCornersRadii)) {
        layerConfig.roundingCornersRadii = CGSizeMake(JobsWidth(10.0), JobsWidth(10.0));
    }

    if (indexPath.row == numberOfRows - 1) {
        // 当前为该 section 的最后一个 cell，处理圆角遮罩
        UIBezierPath *maskPath = [UIBezierPath bezierPathWithRoundedRect:self.bounds
                                                       byRoundingCorners:(UIRectCornerBottomLeft | UIRectCornerBottomRight)
                                                             cornerRadii:layerConfig.roundingCornersRadii];

        self.layer.mask = jobsMakeCAShapeLayer(^(__kindof CAShapeLayer * _Nullable layer) {
            @jobs_strongify(self)
            layer.frame = self.bounds;
            layer.path = maskPath.CGPath;
        });

        // 处理边框
        if (layerConfig.layerBorderCor && layerConfig.borderWidth > 0) {
            // 移除旧的边框图层
            NSArray *sublayers = self.layer.sublayers.copy;
            for (CALayer *sublayer in sublayers) {
                if ([sublayer.name isEqualToString:@"rounded-border-layer"]) {
                    [sublayer removeFromSuperlayer];
                }
            }

            CGFloat width = CGRectGetWidth(self.bounds);
            CGFloat height = CGRectGetHeight(self.bounds);
            CGFloat radius = layerConfig.roundingCornersRadii.height;

            UIBezierPath *borderPath = [UIBezierPath bezierPath];

            // 左下圆弧起点
            [borderPath moveToPoint:CGPointMake(0, height - radius)];
            // 左下圆角
            [borderPath addArcWithCenter:CGPointMake(radius, height - radius)
                                  radius:radius
                              startAngle:M_PI
                                endAngle:M_PI_2
                               clockwise:NO];
            // 底边线到右下角
            [borderPath addLineToPoint:CGPointMake(width - radius, height)];
            // 右下圆角
            [borderPath addArcWithCenter:CGPointMake(width - radius, height - radius)
                                  radius:radius
                              startAngle:M_PI_2
                                endAngle:0
                               clockwise:NO];
            // 右边竖线往上
            [borderPath addLineToPoint:CGPointMake(width, 0)];

            // 👉 左边竖线补充（修复左边断线问题）
            [borderPath moveToPoint:CGPointMake(0, height - radius)];
            [borderPath addLineToPoint:CGPointMake(0, 0)];

            // 添加边框图层
            self.layer.addSublayer(jobsMakeCAShapeLayer(^(__kindof CAShapeLayer * _Nullable borderLayer) {
                @jobs_strongify(self)
                borderLayer.frame = self.bounds;
                borderLayer.path = borderPath.CGPath;
                borderLayer.strokeColor = layerConfig.layerBorderCor.CGColor;
                borderLayer.fillColor = UIColor.clearColor.CGColor;
                borderLayer.lineWidth = layerConfig.borderWidth;
                borderLayer.name = @"rounded-border-layer";
            }));
        }
    } else {
        // 非最后一个 cell，清除圆角遮罩和边框
        self.layer.mask = nil;
        NSArray *sublayers = self.layer.sublayers.copy;
        for (CALayer *sublayer in sublayers) {
            if ([sublayer.name isEqualToString:@"rounded-border-layer"]) {
                [sublayer removeFromSuperlayer];
            }
        }
    }return self.layer;
}
```

#### 1.1.2、除了最后一行以外，所有的cell的最下面的线的颜色为 layerConfig.layerBorderCor

```objective-c
/// 除了最后一行以外，所有的cell的最下面的线的颜色为：layerConfig.layerBorderCor
/// - Parameters:
///   - indexPath: indexPath
///   - bounds: bounds
///   - numberOfRowsInSection: 当前的UITableViewCell对应的section的的row个数
///   - layerConfig: layer的配置文件
-(void)tableViewMakesLastRowCellAtIndexPath:(NSIndexPath *_Nonnull)indexPath
                                     bounds:(CGRect)bounds
                      numberOfRowsInSection:(NSInteger)numberOfRowsInSection
                                layerConfig:(JobsLocationModel *)layerConfig{
    if (!indexPath) return;
    if (!layerConfig.layerBorderCor) layerConfig.layerBorderCor = JobsWhiteColor;
    if (indexPath.row != numberOfRowsInSection - 1) {
        /// 将图层添加到cell的图层中,并插到最底层
        [self.layer insertSublayer:jobsMakeCAShapeLayer(^(__kindof CAShapeLayer * _Nullable layer) {
            layer.borderWidth = layerConfig.borderWidth;
            layer.strokeColor = layerConfig.layerBorderCor.CGColor;
            layer.path = jobsMakeBezierPath(^(__kindof UIBezierPath * _Nullable data) {
                [data moveToPoint:CGPointMake(bounds.origin.x, bounds.size.height)];// 起点
                [data addLineToPoint:CGPointMake(bounds.origin.x + bounds.size.width, bounds.size.height)];// 其他点
            }).CGPath;
        }) atIndex:1];
    }
}
```

#### 1.1.3、除了第一行以外，所有的cell的最上面的线为：layerConfig.layerBorderCor

```objective-c
/// 除了第一行以外，所有的cell的最上面的线为：layerConfig.layerBorderCor
/// - Parameters:
///   - indexPath: indexPath
///   - bounds: bounds
///   - numberOfRowsInSection: 当前的UITableViewCell对应的section的的row个数
///   - layerConfig: layer的配置文件
-(void)tableViewMakesFirstRowCellAtIndexPath:(NSIndexPath *_Nonnull)indexPath
                                      bounds:(CGRect)bounds
                       numberOfRowsInSection:(NSInteger)numberOfRowsInSection
                                 layerConfig:(JobsLocationModel *)layerConfig{
    if (!indexPath) return;
    if (!layerConfig.layerBorderCor) layerConfig.layerBorderCor = JobsWhiteColor;
    if(indexPath.row){
        /// 将图层添加到cell的图层中,并插到最底层
        [self.layer insertSublayer:jobsMakeCAShapeLayer(^(__kindof CAShapeLayer * _Nullable layer) {
            layer.borderWidth = layerConfig.borderWidth;
            layer.strokeColor = layerConfig.layerBorderCor.CGColor;
            layer.path = jobsMakeBezierPath(^(__kindof UIBezierPath * _Nullable linePath) {
                [linePath moveToPoint:CGPointMake(bounds.origin.x, 0)];/// 起点
                [linePath addLineToPoint:CGPointMake(bounds.origin.x + bounds.size.width,0)];/// 其他点
            }).CGPath;
        }) atIndex:1];
    }
}
```

#### 1.1.4、调用示例

```objective-c
- (void)tableView:(UITableView *)tableView
  willDisplayCell:(UITableViewCell *)cell
forRowAtIndexPath:(NSIndexPath *)indexPath{
    NSInteger numberOfRows = [tableView numberOfRowsInSection:indexPath.section];
    if (indexPath.row == numberOfRows - 1) {
        /// 以 section 为单位，仅对每个 section 的最后一行 cell 做圆角处理（cell 之间没有分割线），且不描边顶部
        [cell roundedCornerLastCellByTableView:tableView
                                     indexPath:indexPath
                                   layerConfig:jobsMakeLocationModel(^(__kindof JobsLocationModel * _Nullable model) {
            model.roundingCornersRadii = CGSizeMake(JobsWidth(10.0), JobsWidth(10.0));
            model.borderWidth = JobsWidth(0.5f);
            model.layerBorderCor = JobsGrayColor;
        })];
    }else{
        /// 只描 UITableViewCell 的左右两边
        [cell leftAndRightLineCellByTableView:tableView
                                    indexPath:indexPath
                                  layerConfig:jobsMakeLocationModel(^(__kindof JobsLocationModel * _Nullable model) {
            model.borderWidth = JobsWidth(0.5f);
            model.layerBorderCor = JobsGrayColor;
        })];
    }
}
```

### 1.2、X、Y的偏移量

* 对单个cell的偏移：需要在cell的子类里面复写父类方法-(void)setFrame:(CGRect)frame

*详见@implementation UITableViewCell (Margin)*

```objective-c
// 需要在具体的子类去实现
// 如果在分类调用，则出现异常
-(void)setFrame:(CGRect)frame{
    NSLog(@"self.offsetXForEach = %f",self.offsetXForEach);
    NSLog(@"self.offsetYForEach = %f",self.offsetYForEach);
    frame.origin.x += self.offsetXForEach;
    frame.origin.y += self.offsetYForEach;
    frame.size.height -= self.offsetYForEach * 2;
    frame.size.width -= self.offsetXForEach * 2;
    [super setFrame:frame];
}
```

* 但是对于以section为单位基准的圆角处理

```
参见：1.1的调用示例
```

## 2、关于UICollectionView.UICollectionViewCell

### 2.1、圆切角

#### 2.1.1、对UICollectionView上的每一组的第一个和最后一个UICollectionViewCell进行圆切角

* 对**UICollectionView**上的每一组的第一个和最后一个**UICollectionViewCell**进行圆切角
* 要求切第一个**UICollectionViewCell**的左上+右上，最后一个**UICollectionViewCell**的左下和右下

* 作用域 ：**UICollectionViewCell**子类的` - (void)drawRect:(CGRect)rect`
* 注意：⚠️不能写在`UICollectionViewDelegate`的 `-(void)collectionView:(UICollectionView *)collectionView  willDisplayCell:(UICollectionViewCell *)cell forItemAtIndexPath:(NSIndexPath *)indexPath`方法里面，因为此时-`(NSIndexPath *)jobsGetCurrentIndexPath；`返回nil

```objective-c
-(void)cutFirstAndLastCollectionViewCellWithBackgroundCor:(UIColor *_Nullable)cellBgCor
                                           cellOutLineCor:(UIColor *_Nullable)cellOutLineCor
                                            bottomLineCor:(UIColor *_Nullable)bottomLineCor
                                              borderWidth:(CGFloat)borderWidth
                                         cornerRadiusSize:(CGSize)cornerRadiusSize
                                                       dx:(CGFloat)dx
                                                       dy:(CGFloat)dy{
    if(!cellBgCor) cellBgCor = JobsGreenColor;
    if(!borderWidth) borderWidth = 1.0f;
    if (!bottomLineCor) bottomLineCor = UIColor.whiteColor;
    CGRect bounds = [self dx:dx dy:dy];
    
    NSIndexPath *indexPath = self.jobsGetCurrentIndexPath;
//    NSInteger numberOfSections = self.jobsGetCurrentNumberOfSections;
    NSInteger numberOfItemsInSection = self.jobsGetCurrentNumberOfItemsInSection;

    {
        // 绘制曲线
        UIBezierPath *bezierPath = nil;
        {
            if(numberOfItemsInSection <= 1){/// 一个section里面只有一个item = 四个角都为圆角
                bezierPath = [UIBezierPath bezierPathWithRoundedRect:bounds
                                                   byRoundingCorners:UIRectCornerAllCorners
                                                         cornerRadii:cornerRadiusSize];
            }else{/// 一个section里面有多个item
                if(indexPath.row == 0){/// 首行 = 左上、右上角为圆角
                    bezierPath = [UIBezierPath bezierPathWithRoundedRect:bounds
                                                       byRoundingCorners:(UIRectCornerTopLeft | UIRectCornerTopRight)
                                                             cornerRadii:cornerRadiusSize];
                }else if (indexPath.row == numberOfItemsInSection - 1){/// 末行 = 左下、右下角为圆角
                    bezierPath = [UIBezierPath bezierPathWithRoundedRect:bounds
                                                       byRoundingCorners:(UIRectCornerBottomLeft | UIRectCornerBottomRight)
                                                             cornerRadii:cornerRadiusSize];
                }else{/// 中间的都为矩形
                    bezierPath = [UIBezierPath bezierPathWithRect:bounds];
                }
            }
        }
        
        {
            CAShapeLayer *layer1 = CAShapeLayer.layer;/// 新建一个图层
            layer1.borderWidth = borderWidth;/// 线宽
            layer1.path = bezierPath.CGPath;/// 图层边框路径
            layer1.fillColor = cellBgCor.CGColor;/// 图层填充色,也就是cell的底色
            layer1.strokeColor = cellOutLineCor.CGColor;/// 图层边框线条颜色
            [self.layer insertSublayer:layer1 atIndex:0];/// 将图层添加到cell的图层中,并插到最底层
        }
        /// 除了最后一行以外，所有的cell的最下面的线为bottomLineCor
        [self makeBottomLineWithIndexPath:indexPath
                                   bounds:bounds
                   numberOfItemsInSection:numberOfItemsInSection
                              borderWidth:borderWidth
                            bottomLineCor:bottomLineCor];
        /// 除了第一行以外，所有的cell的最上面的线为bottomLineCor
        [self makeTopLineWithIndexPath:indexPath
                                bounds:bounds
                numberOfItemsInSection:numberOfItemsInSection
                           borderWidth:borderWidth
                         bottomLineCor:bottomLineCor];
    }
}
```

* 除了最后一行以外，所有的cell的最下面的线为bottomLineCor

```
-(void)makeBottomLineWithIndexPath:(NSIndexPath *)indexPath
                            bounds:(CGRect)bounds
            numberOfItemsInSection:(NSInteger)numberOfItemsInSection 
                       borderWidth:(CGFloat)borderWidth
                     bottomLineCor:(UIColor *_Nullable)bottomLineCor{
    if (indexPath.row != numberOfItemsInSection - 1) {
        UIBezierPath *linePath = UIBezierPath.bezierPath;
        /// 起点
        [linePath moveToPoint:CGPointMake(bounds.origin.x, bounds.size.height)];
        /// 其他点
        [linePath addLineToPoint:CGPointMake(bounds.origin.x + bounds.size.width, bounds.size.height)];
        /// 新建一个图层
        CAShapeLayer *layer2 = CAShapeLayer.layer;
        layer2.borderWidth = borderWidth;
        layer2.path = linePath.CGPath;
        layer2.strokeColor = bottomLineCor.CGColor;
        /// 将图层添加到cell的图层中,并插到最底层
        [self.layer insertSublayer:layer2 atIndex:1];
    }
}
```

* 除了第一行以外，所有的cell的最上面的线为bottomLineCor

```
-(void)makeTopLineWithIndexPath:(NSIndexPath *)indexPath
                         bounds:(CGRect)bounds
         numberOfItemsInSection:(NSInteger)numberOfItemsInSection
                    borderWidth:(CGFloat)borderWidth
                  bottomLineCor:(UIColor *_Nullable)bottomLineCor{
    if(indexPath.row){
        UIBezierPath *linePath = UIBezierPath.bezierPath;
        /// 起点
        [linePath moveToPoint:CGPointMake(bounds.origin.x, 0)];
        /// 其他点
        [linePath addLineToPoint:CGPointMake(bounds.origin.x + bounds.size.width,0)];
        /// 新建一个图层
        CAShapeLayer *layer3 = CAShapeLayer.layer;
        layer3.borderWidth = borderWidth;
        layer3.path = linePath.CGPath;
        layer3.strokeColor = bottomLineCor.CGColor;
        /// 将图层添加到cell的图层中,并插到最底层
        [self.layer insertSublayer:layer3 atIndex:1];
    }
}
```

#### 2.1.2、利用UIBezierPath，对 UICollectionViewCell 描边 + 切角

* 作用域 ：UICollectionViewCell子类的 - (void)drawRect:(CGRect)rect

```objective-c
-(void)outlineByBezierPath:(UIBorderSideType)borderSideType
         cellBackgroundCor:(UIColor *)cellBackgroundCor
               borderColor:(UIColor *)borderColor
               borderWidth:(CGFloat)borderWidth
          cornerRadiusSize:(CGSize)cornerRadiusSize
           roundingCorners:(UIRectCorner)roundingCorners{
    if(!borderColor) borderColor = JobsRedColor;
    if(!cellBackgroundCor) cellBackgroundCor = JobsGreenColor;
    if(!borderWidth) borderWidth = 1.0f;
     // 创建一个贝塞尔曲线
    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:self.bounds
                                               byRoundingCorners:roundingCorners
                                                     cornerRadii:cornerRadiusSize];
    // 设置填充颜色为背景颜色，以保留原始背景
    [cellBackgroundCor setFill];
    [path fill];
     
    // 设置边框的属性
    [borderColor setStroke];// 设置边框颜色
    path.lineWidth = borderWidth; // 设置边框宽度
     
    // 添加路径到上下文中并绘制
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextAddPath(context, path.CGPath);
    CGContextDrawPath(context, kCGPathStroke);
}
```

#### 2.1.3、利用CALayer，对 UICollectionViewCell 只描边、不切角

* 作用域 ：**UICollectionViewCell**子类的` - (void)drawRect:(CGRect)rect`

```objective-c
-(void)outlineByLayer:(UIBorderSideType)borderSideType
          borderColor:(UIColor *)borderColor
          borderWidth:(CGFloat)borderWidth{
    
    if(!borderColor) borderColor = JobsRedColor;
    if(!borderWidth) borderWidth = 1.0f;
    
    CALayer *topBorder = nil;
    CALayer *bottomBorder = nil;
    CALayer *leftBorder = nil;
    CALayer *rightBorder = nil;
    
    if(borderSideType & UIBorderSideTypeTop || borderSideType & UIBorderSideTypeAll){
        topBorder = CALayer.layer;
        topBorder.borderColor = borderColor.CGColor; // 选择你想要的颜色
        topBorder.borderWidth = borderWidth;// 选择边框的宽度
        // 设置边框的位置和大小
        topBorder.frame = CGRectMake(0,
                                     0,
                                     self.contentView.frame.size.width,
                                     borderWidth);
    }
    
    if(borderSideType & UIBorderSideTypeBottom || borderSideType & UIBorderSideTypeAll){
        bottomBorder = CALayer.layer;
        bottomBorder.borderColor = borderColor.CGColor; // 选择你想要的颜色
        bottomBorder.borderWidth = borderWidth;// 选择边框的宽度
        // 设置边框的位置和大小
        bottomBorder.frame = CGRectMake(0,
                                        self.contentView.frame.size.height - borderWidth,
                                        self.contentView.frame.size.width,
                                        borderWidth);
    }
    
    if(borderSideType & UIBorderSideTypeLeft || borderSideType & UIBorderSideTypeAll){
        leftBorder = CALayer.layer;
        leftBorder.borderColor = borderColor.CGColor; // 选择你想要的颜色
        leftBorder.borderWidth = borderWidth;// 选择边框的宽度
        // 设置边框的位置和大小
        leftBorder.frame = CGRectMake(0,
                                      0,
                                      borderWidth,
                                      self.contentView.frame.size.height);
    }
    
    if(borderSideType & UIBorderSideTypeRight || borderSideType & UIBorderSideTypeAll){
        rightBorder = CALayer.layer;
        rightBorder.borderColor = borderColor.CGColor; // 选择你想要的颜色
        rightBorder.borderWidth = borderWidth;// 选择边框的宽度
        // 设置边框的位置和大小
        rightBorder.frame = CGRectMake(self.contentView.frame.size.width - borderWidth,
                                       0,
                                       borderWidth,
                                       self.contentView.frame.size.height);
    }
    
    if(topBorder) [self.contentView.layer addSublayer:topBorder];
    if(bottomBorder) [self.contentView.layer addSublayer:bottomBorder];
    if(leftBorder) [self.contentView.layer addSublayer:leftBorder];
    if(rightBorder) [self.contentView.layer addSublayer:rightBorder];
}
```

## 3、指定对某个View的某个角进行圆切角

* 在这个View的具体子类里面，复写系统方法-(**void**)layoutSubviews;

```objective-c
-(void)layoutSubviews{
    [super layoutSubviews];
    /// 内部指定圆切角
    [self layoutSubviewsCutCnrByRoundingCorners:UIRectCornerTopLeft | UIRectCornerTopRight
                                    cornerRadii:CGSizeMake(JobsWidth(8), JobsWidth(8))];
}
```

## 4、对某个View的4个角无差别进行圆切角

```objective-c
[View cornerCutToCircleWithCornerRadius:JobsWidth(8)];
[View layerBorderCor:RGBA_COLOR(255, 225, 144, 1) andBorderWidth:JobsWidth(0.5f)];
```

## 5、其他

* 隐藏每个分区最后一个cell的分割线:系统分割线,移到屏幕外

```objective-c
 - (void)tableView:(UITableView *)tableView
   willDisplayCell:(UITableViewCell *)cell
 forRowAtIndexPath:(NSIndexPath *)indexPath{
 	 _tableView.separatorColor = HEXCOLOR(0xeeeeee);//改变分割线颜色
	 cell.separatorInset = UIEdgeInsetsMake(0, 15, 0, 15);//改变分割线长度
 }
```

