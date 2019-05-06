---
layout: post
title:  自动适配多种尺寸占位图（带缓存）
description: 自动适配多种尺寸占位图（带缓存）
catalog: true
tags:
    - iOS
    - 占位图
---

```
#import "UIImage+Placeholder.h"
#import "MyCommon.h"
#define SCLoading @"SCLoading"

static NSMutableDictionary *mutableImageDic;

@implementation UIImage (Placeholder)

+ (UIImage *)loadingImage:(CGSize)size{
    //初始化
    if (!mutableImageDic) {
         mutableImageDic = [NSMutableDictionary new];
        [mutableImageDic setObject:[UIImage imageNamed:SCLoading] forKey:SCLoading];
    }

    if (CGSizeEqualToSize(size, CGSizeZero)) {//如果是自动布局没有size，则三倍面积
        UIImage *img = (UIImage *)[mutableImageDic objectForKey:SCLoading];
        size = CGSizeMake(img.size.width * 3, img.size.height * 3);
    }

    //如果缓存里面，马上返回
    NSString *imageSizeKey = NSStringFromCGSize(size);
    UIImage *cacheImage = [mutableImageDic objectForKey:imageSizeKey];

    if (cacheImage) {
        return cacheImage;
    }

    //没有就绘制
    UIImage *img = [mutableImageDic objectForKey:SCLoading];
    UIGraphicsBeginImageContextWithOptions(size, NO, [UIScreen mainScreen].scale);
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextSetFillColorWithColor(context, UIColorFromRGB(0xDBDBDB).CGColor);
    CGContextFillRect(context, CGRectMake(0, 0, size.width, size.height));
    if (size.width < img.size.width || size.height < img.size.height) {
        [img drawInRect:CGRectMake(0, 0, size.width, size.height)];
    }else{
        [img drawInRect:CGRectMake((size.width - img.size.width)/2, (size.height - img.size.height)/2, img.size.width, img.size.height)];
    }

    UIImage* retImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    //绘制后添加到到字典内
    [mutableImageDic setObject:retImage forKey:imageSizeKey];
    return retImage;
}

@end

```