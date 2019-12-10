# opencv

## 图片读取
    
    cv2.imread(path, mode)
    cv2.IMREAD_COLOR      不包括透明度。色彩值不变
    cv2.IMREAD_GRAYSCALE  灰度模式
    cv2.IMREAD_UNCHANGED  包括透明度值 和 色彩值
        alpha 通道 像素的透明度。 
            16bit标识像素的话， RGB 分别用5个bit表示。1bit表示alpha。
            32bit标识像素的话，RGB和透明度。都用8bit来表示


## 图片二值化处理
    
### 阈值固定
    函数例子:
        ret,thresh1 = cv2.threshold(grayImage, 127, 255, cv2.THRESH_BINARY)
    参数说明:
        grayImage   灰度图片
        127         灰度阈值
        255         目标灰度值
        cv2.THRESH_BINARY     类型

    以上边例子说明。更改不同二值化类型：
    cv2.THRESH_BINARY  低于阈值的像素点灰度值设置为0， 高于阈值的像素点灰度值设置为255。
    cv2.THRESH_BINARY_INV  低于阈值的像素点灰度值设置为 255， 高于阈值的像素点灰度值设置为0。

    cv2.THRESH_TRUNC   低于阈值的像素点灰度值不变。高于阈值的像素点灰度值设置为255.

    cv2.THRESH_TOZERO   低于阈值的像素点灰度值不变。 高于阈值的像素点灰度值置为0。 255 无用
    cv2.THRESH_TOZERO_INV   低于阈值的像素点灰度值置为0. 高于阈值的像素点灰度值不变。参数255 没有用到。

### 适应性阈值
    
    例子：
    th2 = cv2.adaptiveThreshold(img,255,cv2.ADAPTIVE_THRESH_MEAN_C , cv2.THRESH_BINARY,11,2)
    参数说明：
        img       原图片
        255       设置的值
        cv2.ADAPTIVE_THRESH_MEAN_C      阈值计算类型
        cv2.THRESH_BINARY               二值化类型
        11         11个像素点为一组计算阈值
        2          为常量
    阈值计算类型有俩个
        cv2.ADAPTIVE_THRESH_MEAN_C:    阀值取自相邻区域的平均值
        cv2.ADAPTIVE_THRESH_GAUSSIAN_C: 阀值取自相邻区域的加权和，权重为一个高斯窗口


