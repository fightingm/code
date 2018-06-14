# 使用vr-panorama生成一个vr全景漫游系统(一)

## 前言

本文是用来记录个人项目vr-panorama的实现过程及一些技术难点，阅读本文需要对threejs和空间几何有一些了解。

[项目地址](https://github.com/fightingm/vrpano)

demo地址:

[前端展示](http://jsrun.net/gigKp/embedded/all/light/)

[录制数据](http://jsrun.net/figKp/embedded/all/light/)

## 主要功能

- 全景图浏览
- vr眼镜模式切换
- 不同场景之间跳转
- 懒加载图片资源
- 不支持webgl的情况下使用css3d完成

## 使用threejs做一个全景图

目前使用threejs完成全景图大概有以下几个步骤(本文代码均为大概实现，具体实现可在github中查看)：

### 创建相机和场景

```js
// vrView.js
  init() {
    // 添加场景
    this.scene = new Scene();

    // 添加光源
    const light = new AmbientLight( 0xffffff );
    this.scene.add( light );

    // 设置相机
    const width = this.container.clientWidth,
          height = this.container.clientHeight,
          fov = 90;
    // 创建相机
    let camera = new PerspectiveCamera( fov, width / height, 1, 1000 );
    this.camera = camera;

    // 创建渲染器
    let renderer;
    this.painter = new GlPainter(this);
    renderer = this.painter.renderer;
    renderer.setClearColor(0xEEEEEE, 1.0);
    renderer.setSize( width, height );
    this.renderer = renderer;
  }
```

### 纹理贴图

创建完场景和相机之后我们需要将全景图贴到一个球面上，并且添加到场景中
```js
// glPainter.js
  loadThumb(url, cb) {
    let loader = new TextureLoader();
    loader.crossOrigin = '*';
    loader.load(url, (texture) => {
      texture.minFilter = LinearFilter;
      texture.magFilter = LinearFilter;
      const widthSegments = 64,
            heightSegments = 64;

      const geometry = new SphereGeometry( 10, widthSegments, heightSegments ),
            materials = [new MeshBasicMaterial({map: texture})];

      geometry.scale(-1, 1, 1);
      const sphere = new Mesh( geometry, materials );

      this.viewer.scene.add( this.sphere );
      cb();
    });
  }
```

如果你看过其他threejs完成全景图的文章，你会发现本文与它们不同的地方在于这里的materials是一个数组，这也是实现图片懒加载的需要，因为我们后面会加载每一张碎片图，然后放到这个materials数组中，再渲染到球面的对应位置上。

### 让全景图动起来

目前我们能看到的只是全景图的一部分，我们需要通过鼠标的拖拽来观察到全景图的任意角度，这里做了一个mouseControl来专门处理鼠标事件：

```js
// mouseControl.js
    // 鼠标移动的时候计算出横向和纵向的移动角度，然后传给viewer处理
  handleMouseMove(e) {
    // 移动端缩放
    this.moving = true;
    const x = e.clientX,
        y = e.clientY;
    //这里相当于鼠标移动5px,场景旋转1deg,显示更加平滑
    const curX = ((this.initPos.x - x)/5 + this.startManulRotation[0]) %360,
        curY = ((y - this.initPos.y)/5 + this.startManulRotation[1]) %360;
    this.lastX = x;
    this.lastY = y;
    this.viewer.handleMouseMove(curX, curY);
  }
```

我们需要一个render函数来让动画持续渲染更新

```js
// vrView.js
  render() {
    this.curRenderer.render(this.scene, this.camera);
    requestAnimationFrame(this.render.bind(this));
  }
```

### 场景之间跳转

要想实现场景图的跳转，我们需要在球面上添加一个覆盖物，点击的时候跳转到其他的场景。

### 创建overlay

根据overlays数组生成dom元素，添加到vr容器中。

```js
// vrTravller.js
renderOverlays(overlays) {
    this.viewer.clearOverlay();
    for (let i = 0; i < overlays.length; i++) {
      const dom = this.createOverlay(overlays[i]);
      const overlay = new Overlay(dom, overlays[i]);
      overlay.setTraveller(this);
      this.viewer.addOverlay(overlay);
    }
  }
```

### overlay的坐标

如何确定overlay在全景图中的位置是一个繁琐的过程，我们先看一下最终需要的overlays数组是什么样的:

```json
    "overlays": [{
      "title": "洗手间",
      // 导航的位置，x:经度，y:纬度
      "x": 4.6720072719141,
      "y": -0.52291666726088,
      // 导航的跳转场景标识
      "next_photo_key": "2"
    },
    {
      "title": "厨房",
      "x": 4.6720072719141,
      "y": 0.52291666726088,
      "next_photo_key": "2"
    }]
```

这里的x, y就表示当前overlay的位置信息，这里的x, y并不是表示x坐标和y坐标，因为现在我们是在一个3d坐标系中，仅凭x,y是无法确定一个点的，这里的x表示经度信息0 < x < 2π,y表示纬度信息-2/π < y < 2/π。通过经纬度我们可以确定空间中的一个点。关于球体的经纬度信息可以参考[这篇文章](http://www.hiwebgl.com/?p=339)。

首先，我们添加一个overlay，默认它会显示在屏幕中心位置，然后我们拖动这个overlay把它放到我们想要的位置：

```js
  handleOverlayMouseMove(e) {
    e.cancelBubble = true;
    e.preventDefault();
  
    const diffX = e.clientX - this.mouse.mouseDownX;
    const diffY = e.clientY - this.mouse.mouseDownY;
  
    const x = this.mouse.initX + diffX;
    const y = this.mouse.initY + diffY;
  
    const angles = this.viewer.pixelToAngle(x, y);
    this.curOverlay.setDirAngle(angles.lg, angles.lt);
  }
```

物体跟随鼠标拖动这个效果很容易实现，网上有很多实现方式，但是这里我们不仅需要让overlay跟随鼠标拖动到指定的位置，更最重要的是我们需要在拖动全景图的时候让ovelay也能在正确的位置显示。比如说我们的全景图中有一个楼梯，我想在楼梯的位置添加一个导航，点击之后跳转到楼上，现在我们通过旋转看到了楼梯，它在屏幕中的坐标可能是右下角(100px, 100px)的位置，我们添加了一个dom，它的right值是100，bottom值也是100，然后我们旋转相机，发现这个导航始终在屏幕右下角，这不是我们要的效果。

想要实现这个功能，我们需要将平面2d坐标转换成空间3d坐标，然后设置overlay定位的时候再将空间3d坐标再转换成平面2d坐标赋值给ovelay的样式。

为什么用这种方法能够实现呢。我们整理一下：现在有三个值 overlay在屏幕的位置，overlay在空间的位置，相机的位置。

当我们拖动全景图的时候，实际上我们改变的是相机的位置，但是overlay在空间中的位置是不会发生变化的，当我们拖动overlay的时候，改变的是overlay在空间中的位置，相机的位置此时并不会发生变化。所以如果我们想要在拖动全景图的时候让overlay能够跟随旋转，我们需要找到它们之间的关系。在图形学中，有一个投影的概念，我们将3d的坐标映射到2d坐标的过程就是投影，所以只要在相机旋转的时候使用相机的投影功能获取到overlay的3d坐标投影到平面上2d坐标，然后更新overlay的定位，就能实现跟随旋转了。


所以我们首先需要拿到overlay的3d坐标，我们需要一个函数pixelToAngle，在我们拖动overly的时候把2d坐标转换成3d坐标:

```js
// utils.js
  pixelToAngle(x, y) {
    // 1.将2d坐标转换成3d坐标
    const raycaster = new Raycaster();
    const mouseVector = new Vector2();
    // 把鼠标坐标转换成webgl坐标，webgl的原点在中心，屏幕坐标的原点在左上角
    if(x!== undefined && y !== undefined) {
      mouseVector.x = 2 * (x / this.container.clientWidth) - 1;
      mouseVector.y = - 2 * (y / this.container.clientHeight) + 1;
    }else {
      // 如果没有传x,y默认渲染在页面中心位置
      mouseVector.x = 0;
      mouseVector.y = 0;
    }
    raycaster.setFromCamera(mouseVector, this.camera);
    const intersects = raycaster.intersectObjects([this.painter.sphere]);
    if(intersects.length > 0) {
      const { point } = intersects[0];
      const theta = Math.atan2(point.x, -1.0 * point.z);
      const phi = Math.atan2(point.y, Math.sqrt(point.x * point.x + point.z * point.z)); 
      // 这里的3pi/2,是通过测试log推测出来的
      return {lg: (theta + 3*Math.PI/2) % (Math.PI * 2), lt: phi};
    }
    return {lg: 0, lt: 0};
  }
```

这里使用了threejs的[Raycaster对象](https://threejs.org/docs/index.html#api/core/Raycaster)来实现，首先将屏幕坐标转换成webgl坐标系中的坐标，沿着相机发射一条射线，然后判断与球体的交点，这里的每一个交点就包含了x，y，z坐标，可以直接使用，但是为了兼容经纬度的数据，还需要将3d坐标转换成经纬度。

然后在旋转相机的时候，我们获取到overlay经过相机的投影2d坐标，然后赋值给overlay的dom元素：

```js
// overlay.js
  updatePosition(camera) {
    if(utils.isOffScreen(this.tagMesh, camera)) {
        this.dom.style.display = "none";
    }else{
        this.dom.style.display = "block";
        const position = utils.toScreenPosition(this.tagMesh, camera, this.container);
        // 向下看的时候导航指向z轴，向上看导航指向y轴
        this.dom.style.transform = 'translate3d('+ position[0] +'px, '+ position[1] +'px, 0) rotateZ('+ 0 +'deg)';
    }
    
  }
```

这里的tagMesh保存了我们的overlay在空间中的坐标，它会在每次创建和移动overlay的时候被调用。

```js
 setMesh() {
    let tagMesh = new Mesh();
    tagMesh.position.copy(utils.lglt2xyz(this.dirAngle.x, -this.dirAngle.y + Math.PI/2, 10));
    this.tagMesh = tagMesh;
  }
```

这里有一个lglt2xyz函数是用来把经纬度转换成3d坐标的，如果在pixelToAngle你并没有将3d坐标转换成经纬度信息，就可以直接使用3d坐标，不需要转换的这一步。

然后这里还使用了一个toScreenPosition，用来得到overlay经过相机的投影2d坐标，这个函数相当于pixelToAngle的逆运算:

```js
// utils.js
 toScreenPosition (obj, camera, container){
      let vector = new Vector3();
      let size ={
        width: container.clientWidth,
        height: container.clientHeight
      };

      obj.updateMatrixWorld();
      vector.setFromMatrixPosition(obj.matrixWorld);
      vector.project(camera);

      let target = {
        x: (vector.x + 1) * size.width / 2,
        y: (-vector.y + 1) * size.height / 2
      };
      vector.x = target.x;
      vector.y = target.y;
      return [vector.x, vector.y];
  }
```

### overlay出现两次的bug

这个时候我们的overlay可以跟随我们的全景图一起旋转了，但是在旋转的过程中会遇到一个问题：同一个overlay会在我们的全景图中出现两次，这是因为平面坐标没有z方向，所以空间上的一个点通过相机投影到平面上会产生两个点。

相机是有可视区域的，我们可以通过判断overlay的空间坐标是不是在相机的可视区域内，如果不在可视区域内，我们就把overlay隐藏起来。下面是isOffScreen方法的具体实现：

```js
  isOffScreen (obj, camera){
      let frustum = new Frustum(); //Frustum用来确定相机的可视区域
      let cameraViewProjectionMatrix = new Matrix4();
      cameraViewProjectionMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse); //获取相机的法线
      frustum.setFromMatrix(cameraViewProjectionMatrix); //设置frustum沿着相机法线方向

      return !frustum.intersectsObject(obj);
  }
```

这里使用threejs的Frustum对象(视锥体)，设置视锥体沿着相机法线的方向，然后判断与overlay空间坐标的包围球是否相交，如果相交，则在当前相机视角内。

最后，别忘了在我们的render函数中调用updatePosition方法。

这篇文章就介绍到这里，下篇文章将介绍碎片图的按需加载实现过程。