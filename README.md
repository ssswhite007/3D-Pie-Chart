# pie-3d

使用 Three.js 绘制 3D 饼图(pie)，其中饼图各项会根据比例 高低错落显示。

一共花了 1.5 天完成的，虽然代码并不是那么完美，但做出来了也小激动。

> 中间走了非常多的弯路，写这些文字时已经是半夜 2:20，呵。



<br>

![pie-3d](https://puxiao.com/temp/pie_3d.jpg)

**在线预览：**https://puxiao.com/threejs/pie-3d/

<br>

**源码地址：**https://github.com/puxiao/pie-3d/



<br>

#### 关于如何绘制 3D 的扇形

中间走了不少弯路：

1. 第1条弯路：想通过 Shape 画出各个扇形，然后通过挤压(ExtrudeGeometry) 得到立体的扇形

   失败原因：单个扇形是可以画出，但问题是他们之间的坐标 “互相独立且难易理解”，无法拼凑在一起。

2. 第2条弯路：想通过 CircleGeometry 得到各个扇形，然后通过某种方式让平面的扇形变成立体的

   失败原因：挤压(ExtrudeGeometry) 对象只能是 Shape，不可以是 CircleGeometry

以上方案都失败，最终选择通过 圆柱体(CylinderGeometry) 得到扇形。

**但问题是 扇形的圆柱体(CylinderGeometry) 两侧是镂空的，需要解决这个问题。**

又走了弯路：

1. 第1条弯路：想通过创建 2 个平面(PlaneGeometry) 来作为扇形两侧的面

   失败原因：实在写不出如何设置可以这 2 个面通过 旋转和位移 刚好贴合到扇形两侧

2. 第2条弯路：想通过先创建 1 个 目标扇形，然后再创建 1 个稍微大一点的 另外一个扇形，然后通过 CSG(three-csg-ts)，通过大扇形减去目标扇形，从而得到一个“两侧以封住”的扇形

   失败原因：CSG 它的 加减 是针对 顶点/法线/uv 的计算，并不会“自动”帮你封住原本就存在的缺口。

最终解决方案：

1. 第1步：先创建一个临时的扇形(圆柱体)，radialSegments 和 heightSegments 的值都为 1

2. 第2步：通过访问临时扇形的`.attributes.position.array`，得到全部的顶点信息(共30个数值)

3. 第3步：将顶点信息转换成顶点坐标(Vector3)，一共得到 10 个顶点坐标

   > 此时并不知道这 10 个坐标究竟他们各自对应的是哪个点

4. 第4步：通过不断试验，确定出了这 10 个坐标点分别对应的是哪个点

   > 我的具体做法是：从 10 个顶点坐标取出 2 个，然后将这 2 个坐标(Vector3) 组合成一个路径(Path)，将该路径渲染到场景中，不断组合观察，最终得出 10 个坐标他们依次对应扇形的顶点位置

   > 结论是：
   >
   > //points[0]和points[5]：扇形靠近屏幕的上方弧形的顶点位置
   >
   > //points[1]和points[6]：扇形远离屏幕的上方弧形的顶点位置
   >
   > //points[2]和points[8]：扇形靠近屏幕的下方弧形的顶点位置
   >
   > //points[3]和points[9]：扇形远离屏幕的下方弧形的顶点位置
   >
   > //point[4]：扇形圆心上方顶点的位置
   >
   > //point[7]：扇形圆心下方顶点的位置

5. 第5步：当我们已经知道具体 10 个点的位置后，通过组合得到 2 组值，每一组包含 4 个顶点坐标

6. 第6步：依次将这 2 组顶点坐标通过 'RectangleGeometry.ts' 得到对应的 “侧面”

7. 第7步：创建出真正的扇形(圆柱体)

   > 这个真正的扇形 与第1步中的 临时扇形主要区别在于 radialSegments 和 heightSegments 的值
   >
   > 1. 临时扇形的 radialSegments 和 heightSegments 值为 1
   >
   > 2. 真正扇形的 radialSegments 和 heightSegments 值可以较大一些，这样扇形才足够平滑
   >
   >    我在代码中分别设置 radialSegments 为 32，heightSegments 值为 1

8. 第8步：至此，我们已经有了 扇形 + 2个侧面，且侧面完全贴合扇形的 2 侧，那么就通过 自定义的`SectorMesh.ts` 得到了 “两侧封口，完完整整的扇形”

   > 至此，我们已经可以通过代码 创建出 饼图的扇形

9. 第9步：根据之前计算好的饼图各项的比例值、颜色，依次创建各个扇形

   > 所占比例越高，这个扇形的高度也越高

10. 第10步：根据各个扇形的高度，依次设置它们的 y 轴值，让它们底对齐

11. 第11步：创建渲染场景所需的各种对象，渲染器、场景、镜头、灯光等等，将前面步骤得到的饼图对象添加到场景中，最终实现了整个效果。



<br>

#### 目前不完美的地方

目前 扇形 + 2 个侧面都是相互独立的，他们对应 3 个 BufferGeometry。

这是因为目前 `RectangleGeometry.ts` 不够完善，没有提供 normal 和 uv 值。

假设日后 RectangleGeometry 完善了，可以提供 normal 和 uv 值，那么我们就可以通过 'three-csg-ts' 这样的库，这样就可以将 扇形和 2 个侧面合并成一个独立的 BBufferGeometry。



<br>

如果你有更好的实现方式，请告诉我。

1. 微信(同QQ)：78657141
2. 邮箱：yangpuxiao@gmail.com