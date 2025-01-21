---
title: "Android 相机开发中的尺寸和方向问题"
date: 2018-04-10T16:09:32+08:00
subtitle: ""
draft: false
categories: ["Android"]
tags: ["Camera"]
slug: "android-camera-aspect-ratio--and-orientation"
toc: true
 
--- 

> *本篇文章已授权微信公众号 guolin_blog （郭霖）独家发布


在 Android Camera 开发中，两个比较闹心的问题就是尺寸和方向了。


<!--more-->
 

其中尺寸指的是：

*	相机显示预览帧的尺寸
*	相机拍摄帧的尺寸
*	Android 显示相机预览内容的控件尺寸


而方向指的是

*	相机显示预览帧的方向
*	相机拍摄帧的方向
*	Android 手机自身的方向


在开发中要处理好这三个方向和三个尺寸各自的关系才行，这里以 Camera 1.0 版本的 API 作为示例，参考了 Google 的开源项目：[cameraview](https://github.com/google/cameraview)  和 [android-Camera2Basic](https://github.com/googlesamples/android-Camera2Basic) 。



## 尺寸
---

相机作为硬件设备，可以提供两类尺寸：

*	预览帧尺寸
*	拍摄帧尺寸



#### 预览帧尺寸


通过 `getSupportedPreviewSizes` 方法可以得到支持的预览帧的尺寸集合。

```java
		private final SizeMap mPreviewSizes = new SizeMap();
        mCamera = Camera.open(mCameraId);
        mCameraParameters = mCamera.getParameters();
        // Supported preview sizes
        mPreviewSizes.clear();
        for (Camera.Size size : mCameraParameters.getSupportedPreviewSizes()) {
            mPreviewSizes.add(new Size(size.width, size.height));
        }
```

而 `mPreviewSizes` 是 `SizeMap` 类型，查看源码，实际上就是在添加预览帧尺寸的长和宽时，还计算了他们的长宽比，并保存了起来，存储长宽比的结构可以是一对多的关系，也就是长宽比相同，长和宽的尺寸可以有多种，只要他们最后约分后的比例相同。

```java
    // 同一个长宽比，对应多个尺寸
    private final ArrayMap<AspectRatio, SortedSet<Size>> mRatios = new ArrayMap<>();
```

比如，尺寸是 1920 * 1080 和 1280 * 720 的长宽比都是 16 : 9，而尺寸是 800 * 600 和 640 * 480 的的长宽比都是 4 : 3。

在这里说法是 长宽比，也有说话是 宽高比 的。实际上都是相同的，都是将手机横放时的，较长的那一边比较短的那一边的值。

在计算长宽比时，需要求出宽和高数值的最大公约数，这样才能进行约分计算，根据欧几里得算法，又叫做`辗转相除法`：两个整数的最大公约数等于其中较小的那个数和两个数相除余数的最大公约数。转换成代码如下：


```java
	// a > b
    private static int gcd(int a, int b) {
        while (b != 0) {
            int c = b;
            b = a % b;
            a = c;
        }
        return a;
    }
```



#### 拍摄帧尺寸

通过 `getSupportedPictureSizes` 方法可以得到支持的拍摄帧的尺寸集合。

```java
        // Supported picture sizes;
        private final SizeMap mPictureSizes = new SizeMap();
        mPictureSizes.clear();
        for (Camera.Size size : mCameraParameters.getSupportedPictureSizes()) {
            mPictureSizes.add(new Size(size.width, size.height));
        }
```

存储结构和预览帧相似，在得到尺寸集合时，也计算了它们对应的长宽比。

而 Android 显示相机预览内容的控件尺寸，在控件对应的方法中可以拿到它的 Width 和 Height 。


#### 计算宽高比

有了这三类尺寸，接下来就是要如何处理了。

为了在预览和拍摄时，图像不会出现拉伸现象，预览帧的长宽比最好和显示控件的长宽比一致，并且拍摄帧的长宽比也和预览帧和显示控件的长宽比一致，总之三者的长宽比最好是一致的，才会有最好的预览和拍摄效果

因为手机预览控件的图像 是由 相机预览帧 根据 控件大小 **缩放**得来的，当长宽比不一致时必然会导致预览图像变形。而预览帧的长宽比和拍摄帧的长宽比不一致的话，又会导致拍摄的图片变形拉伸。


在 [cameraview](https://github.com/google/cameraview) 的源码中，首先设定了默认的宽高比为 4 : 3 。

```java
AspectRatio DEFAULT_ASPECT_RATIO = AspectRatio.of(4, 3);
```

根据这一长宽比，可以从预览帧的尺寸集合中得到那些符合的尺寸列表，再从那些尺寸列表中找到宽和高都刚好大于预览控件的宽高的。若是小于预览控件的宽高则会导致图像被拉伸了。


```java
        SortedSet<Size> sizes = mPreviewSizes.sizes(mAspectRatio);
        if (sizes == null) { // Not supported
            mAspectRatio = chooseAspectRatio();
            // 根据选定的长宽比得到对应的支持的尺寸集合
            sizes = mPreviewSizes.sizes(mAspectRatio);
        }
        // 和预览控件的尺寸相比较，从尺寸集合中找到合适的尺寸
        Size size = chooseOptimalSize(sizes);
        // 把找到的合适尺寸，设置给相机的预览帧
        mCameraParameters.setPreviewSize(size.getWidth(), size.getHeight());
```


具体找到合适的预览帧尺寸大小的代码如下：

```java
    private Size chooseOptimalSize(SortedSet<Size> sizes) {
        if (!mPreview.isReady()) { // Not yet laid out
            return sizes.first(); // Return the smallest size
        }
        int desiredWidth;
        int desiredHeight;
        // 预览界面的尺寸
        final int surfaceWidth = mPreview.getWidth();
        final int surfaceHeight = mPreview.getHeight();
        // 是否是横屏,若是横屏的话，宽和高相互调换
        if (isLandscape(mDisplayOrientation)) {
            desiredWidth = surfaceHeight;
            desiredHeight = surfaceWidth;
        } else {
            desiredWidth = surfaceWidth;
            desiredHeight = surfaceHeight;
        }

        // 从选定的长宽比支持的尺寸中，找到长和宽都大于或等于控件尺寸的
        Size result = null;
        for (Size size : sizes) { // Iterate from small to large
            if (desiredWidth <= size.getWidth() && desiredHeight <= size.getHeight()) {
                return size;
            }
            result = size;
        }
        // 实在没有符合条件的，选择支持尺寸中最大的返回。
        return result;
    }
```

注意到，当屏幕处于横屏模式式，预览控件的宽和高就发生变换了，要相互调换。

找到合适的预览帧的尺寸后，就可以设置给相机了。

而相机拍摄帧的尺寸，也是要根据长宽比来选定。

```java
        final Size pictureSize = mPictureSizes.sizes(mAspectRatio).last();
```

在宽高比一定的情况下，拍摄帧往往选择尺寸最大的，那样拍摄的图片更清楚，这也是为什么最后使用 `last` 方法。


这样一来，在确定好了宽高比的情况下，就可以设置对应的尺寸了。

在 Google 的  [android-Camera2Basic](https://github.com/googlesamples/android-Camera2Basic) 工程中，也有这样一段设置尺寸的代码，不同的它是根据拍摄的图片的最大尺寸确定好了长宽比，而不是默认选择普遍的 4 : 3 的比例，之后在此基础之上才进行设置。

```java
       Size largest = Collections.max(Arrays.asList(map.getOutputSizes(ImageFormat.JPEG)),
                        new CompareSizesByArea());
```

可以看到，在相机中设置宽高比还是非常重要的一个环节。

## 方向
---

搞定了尺寸问题，还剩下方向了。

相机有两种方向需要处理：

*	预览帧方向
*	拍摄帧方向

为了获得更好的相机体验，要处理好预览帧和拍摄帧的方向，保证通过手机屏幕看到的内容都是方向正常的。

首先要明确手机的自然方向：

*	当手机屏幕 `竖立时的自然方向`，此时，坐标原点位于左上角，向右为 X 轴正方向，向下为 Y 轴正方向，**宽比高短**。
*	当手机屏幕 `横放时的自然方向`，此时，坐标原点位于左上角，向右为 X 轴正方向，向下为 Y 轴正方向，**宽比高长**。


#### 预览帧方向

而相机的图像数据是来自相机硬件图像传感器的，传感器被固定在手机上后有一个默认的取景方向：坐标原点位于手机**逆时针横放**时的左上角，即与横屏应用的屏幕 X 方向一致。也就是与竖屏应用的屏幕 X 方向呈 90 度角。

这里盗图几张：


![back_camera_coordinate](https://image.glumes.com/images/2019/04/27/back_camera_coordiante.png)


所以，对于横屏应用来说，屏幕的自然方向和相机的图像传感器方向一致，因此看到的图像是正的。而对于竖屏应用来说，预览图像就侧过来了。需要将预览图像顺时针旋转 90 度角才可以正常预览图像。


横屏拍摄结果：

![landscape_camera](https://image.glumes.com/images/2019/04/27/landscape_camera_orientation.png)

竖屏拍摄结果：


![portrait](https://image.glumes.com/images/2019/04/27/portrait_camera_orientation.png)


关于相机的预览方向和屏幕自然方向存在 90 度角的偏差，在 Camera 的 `orientation`属性中也有说明：

![camera_orientation_description](https://image.glumes.com/images/2019/04/27/camera_orientation__description.png)

orientation 表示相机图像的方向。它的值是相机图像顺时针旋转到设备自然方向一致时的图像，它可能是 0、90、180、270 四种。

对于竖屏应用来说，后置相机传感器是横屏安装的，当你面向屏幕时，如果后置相机传感器顶边和设备自然方向的右边是平行的，那么后置相机的 orientation 是 90。如果是前置相机传感器顶边和设备自然方向的右边是平行的，则前置相机的 orientation 是 270 。

对于前置和后置相机传感器 orientation 是不同的，在不同的设备上也可能会有不同的值。

在没有限定 Activity 方向时，采用官方推荐的代码来设置方向：

```java
  public static void setCameraDisplayOrientation(Activity activity, int cameraId, android.hardware.Camera camera) {
        android.hardware.Camera.CameraInfo info =
                new android.hardware.Camera.CameraInfo();
        android.hardware.Camera.getCameraInfo(cameraId, info);
        int rotation = activity.getWindowManager().getDefaultDisplay()
                .getRotation();
        int degrees = 0;
        switch (rotation) {
            case Surface.ROTATION_0:
                degrees = 0;
                break;
            case Surface.ROTATION_90:
                degrees = 90;
                break;
            case Surface.ROTATION_180:
                degrees = 180;
                break;
            case Surface.ROTATION_270:
                degrees = 270;
                break;
        }

        int result;
        if (info.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
            result = (info.orientation + degrees) % 360;
            result = (360 - result) % 360;  // compensate the mirror
        } else {  // back-facing
            result = (info.orientation - degrees + 360) % 360;
        }
        camera.setDisplayOrientation(result);
    }
```
首先计算得到设备逆时针旋转的角度，对于后置摄像头传感器的计算：(info.orientation - degrees + 360) % 360 。

因为摄像头图像方向要恢复到自然方向需要顺时针旋转，而屏幕逆时针旋转正好抵掉了摄像头的旋转，所以两者相减，然后再加上 360 取模运算。

对于前置摄像头传感器，因为在使用前置摄像头时，从屏幕竖直方向看到的往往是一个镜像，这是因为摄像头硬件对图像做了水平翻转，也就是将图像内容对着竖直方向对调了，相当于预先旋转了 180 度。之后再只需要旋转 90 度就可以到自然方向了，只不过是个镜像，即左右翻转了。

需要注意的一点是，在 API 14 之前，调用 setDisplayOrientation 方法时要先关闭预览。

最后盗图更清晰明了一下：

![](https://image.glumes.com/images/2019/04/27/font_camera_orientation.png)


#### 拍摄帧方向

确定了预览时的方向，还需要确定拍摄时的方向。

通过 Camera.Parameters.setRotation 函数可以设置相机最终拍出的图片方向。

官方的推荐代码：

```java
    public void onOrientationChanged(int orientation) {
        if (orientation == ORIENTATION_UNKNOWN) {
            return;
        }
        android.hardware.Camera.CameraInfo info = new android.hardware.Camera.CameraInfo();
        android.hardware.Camera.getCameraInfo(cameraId, info);

        orientation = (orientation + 45) / 90 * 90;
        int rotation = 0;

        if (info.facing == CameraInfo.CAMERA_FACING_FRONT) {
            rotation = (info.orientation - orientation + 360) % 360;
        } else {  // back-facing camera
            rotation = (info.orientation + orientation) % 360;
        }
        mParameters.setRotation(rotation);
    }
```

计算旋转的方向，需要考虑到当前屏幕的方向和相机的方向。

OrientationEventListener 和 Camera.orientation 一起配合使用。当屏幕方向改变时，OrientationEventListener 会收到相应的通知，在 onOrientationChanged 的回调方法中去改变相机的拍摄方向，实际上在相机预览方向的改变也是在该回调方法中进行的。

onOrientationChanged 方法的返回值是从 0 ~ 359。而 setRotation 的值只能是 0、90、180、270。所以需要对屏幕方向的 orientation 做一个类似四舍五入的操作。

当然也可以在此回调中根据 Display 类的 getRotation 方法得到方向就行，总之就是有一个回调的通知，然后在此改变屏幕拍摄和预览的方向。

对于前置摄像头，摄像头的 orientation 和屏幕方向的 orientation 两个之差即为要旋转的角度；对于后置摄像头，两者之和即为要旋转的角度。


到这里，就对摄像头开发中的尺寸和方向设置有个更清晰的认识了。


## 参考
---

1. https://blog.csdn.net/Tencent_Bugly/article/details/53375311
2. https://blog.csdn.net/daiqiquan/article/details/40650055
3. https://zhuanlan.zhihu.com/p/20559606
4. http://javayhu.me/blog/2017/09/25/camera-development-experience-on-android/








