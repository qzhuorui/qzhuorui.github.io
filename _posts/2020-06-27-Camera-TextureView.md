---
layout: post
title: 'Camera简单使用并通过TextureView预览'
date: 2020-06-27
author: qzhuorui
color: rgb(98,170,255)
tags: 音视频
---

> 音视频系列的所有总结内容，均可以在本人GitHub找到对应代码实现。

# Camera简单使用并通过TextureView预览

## 一、Camera使用流程：

### 1.权限：

```
<uses-permission android:name = "android.permission.CAMERA" />
<uses-feature android:name = "android.hardware.camera" />
<uses-feature android:name = "android.hardware.camera.autofocus" />
//5.0以上注意动态申请
```

### 2.打开Camera：

`mCamera = Camera.open(mCameraID); ` 

### 3.预览Camera：

1. 设置camera接收预览的输出，可以使surfaceTexture或surfaceHolder

   1. ```
      mCamera.setPreviewTexture(surfaceTexture);
      mCamera.setPreviewDisplay(surfaceHolder);
      ```

2. 设置camera方向，因为camera默认水平。如果固定portrait可以直接旋转90°，但可以先找出屏幕方向再设置

   1. ```java
      mOrientation = getCameraPreviewOrientation(activity, mCameraID);
      mCamera.setDisplayOrientation(mOrientation); 
      ```

   2. ```java
      public static int getCameraPreviewOrientation(Activity activity, int cameraId) {
          if (mCamera == null) {
              throw new RuntimeException("mCamera is null");
          }
          Camera.CameraInfo info = new Camera.CameraInfo();
          Camera.getCameraInfo(cameraId, info);
          int result;
          int degrees = getRotation(activity);
          //前置
          if (info.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
              result = (info.orientation + degrees) % 360;
              result = (360 - result) % 360;
          }
          //后置
          else {
              result = (info.orientation - degrees + 360) % 360;
          }
          return result;
      }
          
      public static int getRotation(Activity activity) {
          int rotation = activity.getWindowManager().getDefaultDisplay().getRotation();
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
          return degrees;
      }
      ```

### 4.为camera设置预览尺寸：

1. 问题1：预览图像拉伸变形，camera提供很多个预览尺寸，如何选择最合适的？一种是选择与textureview宽高最接近的；另一种是选择分辨率最大的。第一种效果不好，第二种后置可以，但前置仍有拉伸感。
2. 问题2：camera.size的width也是大于height，同size进行转化需要注意
3. **始终注意一点**：camera获取的预览尺寸列表都是：宽 > 高，所以在获取选择合适的分辨率时候，都选择把textureview大的边设为宽，小的设为高，再进行选择比较。 

### 5.打开相机，预览的时机：

textureVIew回调中

```java
@Override
    public void onSurfaceTextureAvailable(SurfaceTexture surfaceTexture, int width, int height) {
        mPreviewSize = new Size(width, height);
        startPreview();
    }

    @Override
    public void onSurfaceTextureSizeChanged(SurfaceTexture surfaceTexture, int width, int height) {
        focus(width/2, height/2, true); //自动对焦
    }

    @Override
    public boolean onSurfaceTextureDestroyed(SurfaceTexture surfaceTexture) {
        releasePreview();
        return false;
}

```



## 二、手动对焦与自动对焦

### 1.手动对焦

首先需要给textureView设置一个触摸监听 `textureView.setOnTouchListener(this);` 传入点击的坐标值。

`focus((int) event.getX(), (int) event.getY(), false);`  接着需要把坐标值转化为Point。具体代码如下

```java
//focusPoint:点击区域；screenSize屏幕宽高Size对象
QzrCameraManager.getInstance().activeCameraFocus(focusPoint, screenSize, new Camera.AutoFocusCallback() {
            @Override
            public void onAutoFocus(boolean success, Camera camera) {
                isFocusing = false;
                if (!autoFocus) {
                    focusViewWidget.endFocus(success);
                }
            }
        });
```

接下来在CameraManager中进行处理

```java
    public boolean activeCameraFocus(Point focusPoint, Size screenSize, Camera.AutoFocusCallback focusCallback) {
        if (mCamera == null) {
            Log.e(TAG, "activeCameraFocus: camera is null");
        }
        //点击坐标转化为camera对焦坐标
        Point cameraFocusPoint = CameraUtil.convertToCameraPoint(screenSize, focusPoint);
        Rect cameraFocusRect = CameraUtil.convertToCameraRect(cameraFocusPoint, 100);

        Camera.Parameters parameters = mCamera.getParameters();

        if (Build.VERSION.SDK_INT > 14) {
            if (parameters.getMaxNumFocusAreas() <= 0) {
                //不支持
                return focus(focusCallback);
            }
            clearCameraFocus();
            List<Camera.Area> focusAreas = new ArrayList<>();
            //100是权重
            focusAreas.add(new Camera.Area(cameraFocusRect, 100));
            parameters.setFocusAreas(focusAreas);
            //设置感光区域
            parameters.setMeteringAreas(focusAreas);
            try {
                mCamera.setParameters(parameters);
            } catch (Exception e) {
                Log.e(TAG, "activeCameraFocus: error");
                e.printStackTrace();
                return false;
            }
        }
        return focus(focusCallback);
    }
```

```java
    private void clearCameraFocus() {
        if (mCamera == null) {
            throw new RuntimeException("mCamera is null");
        }
        mCamera.cancelAutoFocus();
        Camera.Parameters parameters = mCamera.getParameters();
        parameters.setFocusAreas(null);
        parameters.setMeteringAreas(null);
        try {
            mCamera.setParameters(parameters);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private boolean focus(Camera.AutoFocusCallback autoFocusCallback) {
        if (mCamera == null) {
            return false;
        }
        mCamera.cancelAutoFocus();
        mCamera.autoFocus(autoFocusCallback);
        return true;
    }
```

### 2.自动对焦

同理手动对焦。只不过需要调用加速度传感器。

`focus(textureView.getWidth() / 2, textureView.getHeight() / 2, true);` 