本章介绍了渲染设计的关键概念；后续章节用特定的最佳实践与性能技术对此信息加以扩展。

<br />

## 如何看待OpenGL ES

本小节描述了针对OpenGL ES进行设计的两个方面：作为一个客户端-服务器架构，以及作为一条图形流水线。这两个方面对于规划和评估你的App都很有用。

<br />

### OpenGL ES作为一个客户端-服务器架构

下图将OpenGL ES视作为客户端-服务器架构。你的App与OpenGL ES客户端进行通信，访问传输状态的改变、纹理与顶点数据，以及渲染命令。客户端将此数据翻译为图形硬件可理解的一种格式，然后将它们传送给GPU。这些过程会对你App的图形性能增添负荷。

![Figure 6-1](https://github.com/zenny-chen/iOS-OpenGL-ES-Design-Guideline----Chinese-Translation/blob/master/figure6-1.png)

要达到相当好的性能需要仔细地管理此负荷。一款良好设计的App应当减少它对OpenGL ES的调用频率，使用适用于硬件的数据格式以最小化翻译成本，并且仔细管理App本身与OpenGL ES之间的数据流。

<br />

### OpenGL ES作为一条图形流水线

下图将OpenGL ES视作为一条图形流水线。你的App对此图形流水线进行配置，然后执行绘制命令，将顶点数据顺着此流水线向下发送。流水线的后继阶段运行了一个顶点着色器来处理顶点数据，将顶点装配成图元（***primitives***），然后将图元光栅化（***rasterize***）为片段（***fragments***），然后运行一个片段着色器对每一个片段计算颜色与深度值，最后将片段进行颜色混合，送到帧缓存（***framebuffer***）用于显示。

![Figure 6-2](https://github.com/zenny-chen/iOS-OpenGL-ES-Design-Guideline----Chinese-Translation/blob/master/figure6-2.png)

我们要将此流水线用作为一个标杆模型，以确定当前App执行了什么工作来生成一帧。你的渲染器设计由以下这些步骤构成：编写着色器程序以处理流水线的顶点与片段阶段，组织你给这些着色器程序所投送的顶点与纹理数据，配置OpenGL ES状态机来驱动流水线的固定功能阶段。
图形流水线中的独立的阶段（或称为“级”，***stage***）可以同时计算其各自的结果——比如，你的App可能在准备新的图元，而图形硬件的独立部分正在对之前所提交的几何执行顶点与片段计算。然而，后面的阶段依赖于前面阶段的输出。如果任一流水线阶段执行太多工作，或者执行得太慢，那么其他流水线阶段会处于空闲状态，一直坐等最慢的阶段完成其工作。一款良好设计的App应该根据图形硬件的能力与特征平衡好由每个流水线阶段所执行的工作。

**`注意：当你在调优你App的性能时，第一步往往是确定哪个阶段是性能瓶颈。`**

<br />

## OpenGL ES版本与渲染器架构

iOS支持三个OpenGL ES版本。更加新的版本提供更好的灵活性，允许你实现更高级的渲染算法，包括高质量的视觉效果而不用与性能进行妥协。

<br />

### OpenGL ES 3.0

OpenGL ES 3.0出在iOS 7系统上。你的App可使用OpenGL ES 3.0所引入的特征来实现高级图形编程技术，以获得更高的图形性能以及引人入胜的视觉效果，这些技术先前原本只能在桌面级硬件和游戏平台上使用。

<br />

#### OpenGL ES着色器语言版本3.0

GLSL ES 3.0增添了新的特性，诸如 **uniform blocks**，32位整数与新增的整数操作，用于在顶点着色器和片段着色器程序中执行更通用目的的计算任务。为了要在一个着色器程序中使用新语言，你的着色器源代码必须以`#version 300 es`指示符打头。OpenGL ES 3.0上下文仍然与OpenGL ES 2.0所写的着色器兼容。

<br />

#### 多重渲染目标

通过允许多重渲染目标，你可以创建同时写到多个帧缓存附件（***framebuffer attachments***）的片段着色器。
这个特征可允许你使用诸如***延迟着色***（***deferred shading***）这样的高级渲染算法。在这种方式下，你的App先渲染到一组纹理来存放几何数据，然后执行一个或多个着色遍用于从这些纹理中读取数据，然后执行光照计算输出最终的图像。因为这个方法预先计算了对光照计算的输入，所以用于对一个场景增加更大数量的光照所带来的执行开销的增长要小得多。延迟着色算法要求多个渲染目标的支持，如下图所示，以达到理想的性能。否则的话，对多个纹理的渲染需要对每个纹理执行一单独的绘制遍。

![屏幕快照 2018-11-11 下午11.25.01.png](https://upload-images.jianshu.io/upload_images/8136508-c3ab19598e488a25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你用一个额外的创建一个帧缓存对象的过程来设置多个渲染目标。不是说要对一个帧缓存创建一单个颜色附件，而是要创建若干个。然后，调用`glDrawBuffers`函数来指定哪些帧缓存附件用于渲染，如以下代码所示。

```c
/// 设置多个渲染目标

// Attach (previously created) textures to the framebuffer.
glFramebufferTexture2D(GL_DRAW_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D,  colorTexture, 0);
glFramebufferTexture2D(GL_DRAW_FRAMEBUFFER, GL_COLOR_ATTACHMENT1, GL_TEXTURE_2D,  positionTexture, 0);
glFramebufferTexture2D(GL_DRAW_FRAMEBUFFER, GL_COLOR_ATTACHMENT2, GL_TEXTURE_2D,  normalTexture, 0);
glFramebufferTexture2D(GL_DRAW_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_TEXTURE_2D,  depthTexture, 0);
 
// Specify the framebuffer attachments for rendering.
GLenum targets[] = {GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1, GL_COLOR_ATTACHMENT2};
glDrawBuffers(3, targets);
```

当你的App发布绘制命令时，你的片段着色器判定对于每个渲染目标中的每一像素，什么颜色（或非颜色数据）被输出。以下代码展示了一个基本的片段着色器渲染到多个目标，通过给输出变量进行赋值，这些变量的位置与上述代码所列出的匹配。

```glsl
/// 具有输出到多个渲染目标的片段着色器

#version 300 es
 
uniform lowp sampler2D myTexture;
in mediump vec2 texCoord;
in mediump vec4 position;
in mediump vec3 normal;
 
layout(location = 0) out lowp vec4 colorData;
layout(location = 1) out mediump vec4 positionData;
layout(location = 2) out mediump vec4 normalData;

void main(void)
{
    colorData = texture(myTexture, texCoord);
    positionData = position;
    normalData = vec4(normalize(normal), 1.0);
}
```

多重渲染目标对于其他高级图形技术也很有用，比如实时反射、屏幕空间的环境光遮蔽（*ambient occlusion*）、以及体积光照（*volumetric lighting*）。

<br />

#### 变换回馈

图形硬件使用一个高度并行的架构对向量处理进行优化。你可以用新增的变换回馈特性更好地利用此硬件。这个特性可以让你从一个顶点着色器捕获输出，然后再更新到GPU存储器中的缓存对象。你可以从一个渲染遍中捕获数据，然后在另一个渲染遍中使用，或者禁用图形流水线的某些部分，然后对通用目的计算使用变换回馈。
能从变换回馈只能中受益的一种技术是动画粒子效果。渲染一个粒子系统的一个通用架构由下图列出。首先，App设置粒子模拟的初始状态。然后，对于每一要渲染的帧，App运行一次其模拟的步骤，更新每个被模拟粒子的位置、方向与速度，然后绘制用于表示粒子当前状态的视觉资源。

![屏幕快照 2018-11-12 上午1.28.03.png](https://upload-images.jianshu.io/upload_images/8136508-08c54ec0986a897e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

传统上，实现粒子系统的App在CPU上运行其模拟，将模拟结果存储到一个顶点缓存中用于粒子渲染。然而，将顶点缓存的内容传输到GPU存储器是很耗费时间的。变换回馈通过优化现代GPU硬件可用的并行架构的威力来更有效地解决问题。
通过变换回馈，你可以设计你自己的渲染引擎来更高效地解决这个问题。下图大概展示了你的App可以如何配置OpenGL ES图形流水线来实现一个粒子系统动画。因为OpenGL ES将每个粒子以及其状态表示为一个顶点，所以GPU的顶点着色器阶段可以一次运行若干个粒子的模拟。因为包含粒子状态数据的顶点缓存在帧与帧之间重用，所以将那些数据传送到GPU存储器的耗时过程仅发生一次，即在初始化时。

![屏幕快照 2018-11-12 上午2.25.00.png](https://upload-images.jianshu.io/upload_images/8136508-ebe850ebf669d796.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 在初始化时，创建一个顶点缓存并用包含在此模拟中的所有粒子的初始状态的数据对它填充。
1. 在GLSL顶点着色器程序中实现你的粒子模拟，并且通过绘制包含粒子位置数据的顶点缓存的内容，每一帧运行此程序。
・为了在渲染过程中允许变换回馈，调用`glBeginTransformFeedback`函数。（在返回正常绘制之前调用`glEndTransformFeedback`函数。）
・使用`glTransformFeedbackVaryings`函数来指定哪些着色器输出应该被变换回馈捕获，并使用`glBindBufferBase`或`glBindBufferRange`函数，以及`GL_TRANSFORM_FEEDBACK_BUFFER`缓存类型来指定这些着色器输出将被捕获到哪个缓存中。
・通过调用`glEnable(GL_RASTERIZER_DISCARD)`禁用光栅化（以及图形流水线的后续阶段）。
1. 为了渲染此模拟结果用于展示，使用包含粒子位置的顶点缓存作为第二个绘制遍的输入，再一次允许光栅化（以及图形流水线的其余阶段），并使用适用于你App的视觉内容的顶点跟片段着色器。
1. 在下一帧，使用上一帧的模拟步骤所输出的顶点缓存作为下一模拟步骤的输入。

其他能从变换回馈获益的图形编程技术包括骨骼动画（*skeletal animation*）以及光线行进（*ray marching*）。

<br />

### OpenGL ES 2.0

OpenGL ES 2.0用可编程着色器提供了灵活的图形流水线，并且对当前所有iOS设备可用。许多在OpenGL ES 3.0规范里正式引入的特征可通过OpenGL ES 2.0扩展对iOS设备可用。因此，你可以实现许多高级图形编程技术而又能保留对大部分设备的兼容性。

<br />

### OpenGL ES 1.1

OpenGL ES 1.1仅仅提供了一个基本的固定功能流水线。iOS支持OpenGL ES 1.1主要用于向下兼容。

<br />

## 设计一款高性能的OpenGL ES App

总的来说，一款良好设计的OpenGL ES App需要以下两点：
・利用OpenGL ES流水线的并行性。
・在App与图形硬件之间管理数据流。

下图对一款App使用OpenGL ES执行动画用于显示的流程做了一个建议

![屏幕快照 2018-11-13 上午1.56.02.png](https://upload-images.jianshu.io/upload_images/8136508-e8a3304a9714bbc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当App启动时，它所做的第一件事就是初始化那些并不打算在整个App执行周期里进行改变的资源。理想情况下，App将这些资源封装为OpenGL ES对象。目标是创建任一对象，在App运行时（或App运行生命周期的某一部分，比如一部游戏中的某一关之中）保持不变，以增加初始化的时间来交换更好的渲染性能。应该用OpenGL ES对象来代替复杂的命令或状态改变，这样可以只需用一单次函数调用。例如，配置固定功能流水线可能需要数十条函数调用。取而代之的是，在初始化时编译一个图形着色器，然后在运行时用一单条函数调用切换到此着色器程序。（言下之意就是我们可以用OpenGL ES 2.0+的可编程着色器来代替OpenGL ES 1.1中的固定功能流水线，以减少函数调用次数，从而获得更好的性能。）需要花费较多时间、较多资源进行创建或修改的OpenGL ES对象应该几乎总是作为**静态对象**进行创建。

渲染循环处理你打算渲染到OpenGL ES上下文中的所有项目，然后将结果呈现到显示器上。在一个动画场景中，某些数据需要对每一帧进行更新。在内部渲染循环中，App在更新渲染资源（在此过程中创建或修改OpenGL ES对象）与提交使用那些资源的绘制命令之间不断交替。此内部循环的目标是平衡好工作负荷，使得CPU与GPU能并行工作，防止App与OpenGL ES同时访问同一资源。**在iOS上，当对一个OpenGL ES对象的修改没有在一帧的开始或结尾处执行时，这个修改可能是昂贵的。**

对于此内部循环的一个重要目标就是避免从OpenGL ES将数据拷贝回App。从GPU将结果拷贝到CPU可能会相当慢。如果所拷贝到数据稍后也要用于当前帧的渲染处理，正如中间渲染循环所展示的，那么你的App会被阻塞，直到此前所提交的绘制命令全都完成。

在App提交了渲染当前帧所需要的所有绘制命令之后，App将结果呈现到显示屏上。一款非可交互的App可以将最终图像拷贝到App存储器用于进一步的处理。

最后，当你的App准备退出时，或者当它完成了一个主要任务时，该App需要释放OpenGL ES对象，使得这些资源可再度使用。

下面概括一下本设计的重要特性：
・在每当实际需要的时候创建静态资源。
・内部渲染循环在修改动态资源与提交渲染命令之间交替进行。设法避免对动态资源的修改，除非在一帧的开头或结尾。
・避免将中间渲染结果读回到你的App中。

本章剩余部分提供了有用的OpenGL ES编程技术来实现此渲染循环的特征。后续章节描述了如何将这些通用技术应用到OpenGL ES的特定领域。

<br />

## 避免同步和冲刷操作

OpenGL ES规范没有要求立即执行命令的实现。通常，命令会被排队到一个命令缓存然后在稍后时间由硬件执行。OpenGL ES往往在App将命令发送到硬件之前会一直等待，直到App已经排队了足够多的命令进行发送——批处理通常更高效。然而，某些OpenGL ES功能必须立即冲刷命令缓存。其他还有些功能不仅要冲刷命令缓存，而且还会一直阻塞，直到之前所提交的命令在将控制返还给App之前完成。只有在必要的时候才去使用冲刷和同步命令。过度地使用冲刷和同步命令可能会导致你的App延迟，由于它需要等待硬件完成渲染。

以下这些情况要求OpenGL ES将命令缓存提交到硬件用于执行。
・函数`glFlush`将命令缓存发送到图形硬件。OpenGL ES一直阻塞，直到命令全都提交到硬件，但不会等待命令完成执行。
・函数`glFinish`冲刷命令缓存，然后等待所有之前所提交的命令在图形硬件上完成执行。
・重新获取帧缓存内容的函数（比如`glReadPixels`）也会等待所提交的命令完成。
・命令缓存满了。

<br />

### 有效地使用glFlush

在某些桌面OpenGL实现上，周期性地调用`glFlush`函数来高效平衡CPU与GPU的工作可能是有用的，但在iOS上却没有这种情况。由iOS图形硬件所实现的**基于区块的延迟渲染**（***Tile-based Deferred Rendering***）算法依赖于将所有顶点数据一次全都缓存在一个场景中，这样可以对隐藏面移除做最优处理。一般，只有两种情况OpenGL ES App应该调用`glFlush`或`glFinish`函数。
・当你的App移到后台时，你应该冲刷命令缓存，因为当你的App在后台时执行OpenGL ES命令会导致iOS终止你的App。
・如果你的App在多个上下文之间共享了OpenGL ES对象（比如顶点缓存和纹理），那么你应该调用`glFlush`函数来同步访问这些资源。比如，你应该在一个上下文中加载了顶点数据之后调用`glFlush`函数，以确保其内容准备好给另一个上下文获取。这个建议也应用于当OpenGL ES对象与其他iOS API进行共享时，比如Core Image。

<br />

### 避免查询OpenGL ES状态

调用`glGet*()`，包括`glGetError()`，可能要求OpenGL ES在获取任何状态变量之前执行先前的命令。这个同步迫使图形硬件与CPU作步调一致的运行，从而减少了并行的机会。为了避免这种情况的发生，维护对任何你想要查询的某一状态的一份拷贝，然后直接访问它，而不是调用OpenGL ES。
当错误发生时，OpenGL ES会设置一个错误标志。这些和其他错误会在Xcode中的OpenGL ES帧调试器中或Instruments中的OpenGL ES分析器中出现。你应该使用那些工具，而不是调用`glGetError`函数。如果频繁调用`glGetError`函数会降低性能。其他查询，诸如`glCheckFramebufferStatus()`、`glGetProgramInfoLog()`以及`glValidateProgram()`，同样一般也只有当开发和调试的时候才有用。你应当在你的App使用发布模式构建的时候省去对这些函数的调用。

<br />

## 使用OpenGL ES管理你的资源

许多OpenGL ES数据片段可以直接存储到OpenGL ES渲染上下文中。OpenGL ES实现可以几乎无开销地将数据转换为对于图形硬件最优的一种格式。这可以极大地提升性能，尤其对于不频繁改变的数据。你的App也可以给OpenGL ES提供暗示，关于让它打算如何使用数据。一个OpenGL ES实现可以使用这些暗示来更高效地处理数据。比如，静态数据可以放在图形处理器能准备获取的存储器中，或甚至存放在专门的图形存储器中。

<br />

## 使用双缓存来避免资源冲突

当你的App和OpenGL ES同时访问一个OpenGL ES对象时，就会发生资源冲突。当一个参与者企图修改正在被另一个参与者所使用的OpenGL ES对象时，它们可能会被阻塞，直到该对象不再处于使用中。一旦它们开始修改此对象，另一个参与者可能无法访问此对象，直到对该对象的修改完成。还有一种可替换的方式，OpenGL ES可以隐式地复制这个对象，使得两个参与者可以继续执行命令。这两种选择都是安全的，但这每一种都最终会对你的App产生瓶颈。下图展示了这个问题。在此例子中，有一单个纹理对象，该纹理对象OpenGL ES和你的App都想使用。当App企图改变纹理时，它必须等待，直到先前所提交的绘制命令完成——即CPU同步到GPU。
![屏幕快照 2018-11-16 下午6.10.53.png](https://upload-images.jianshu.io/upload_images/8136508-d675997e4d76e6e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了解决这个问题，你的App可以在改变对象与用它绘制之间执行额外的工作。但是，如果你的App没有额外的工作可以执行，则应该显式创建两个同样大小的对象；一个参与者读一个对象，而另一个参与者则修改另一个对象。下图展示了双缓存方法。当GPU在对一个纹理进行操作时，CPU则修改另一个。在开始启动之后，CPU与GPU都不会坐等。尽管下图主要针对纹理，但对于其他大部分类型的OpenGL ES对象也能起作用。
![屏幕快照 2018-11-20 下午6.47.15.png](https://upload-images.jianshu.io/upload_images/8136508-b5bb9b0c0b5d5cab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

双缓存对于大部分App都足够了，但它要求两个参与者在差不多相同时刻结束处理命令。为了避免阻塞，你可以添加更多的缓存；这种实现是传统的**生产者-消费者**模型。如果生产者在消费者结束处理命令之前结束，那么它会取一个空闲缓存然后继续处理命令。在这种情境下，生产者只有在消费者被严重拖到后面的时候才会发生闲置。
双缓存与三重缓存在消耗额外的存储器与防止流水线发生延迟之间进行权衡。对存储器的额外使用可能会导致你App其他部分的压力。在iOS设备上，存储器可能是稀缺的；你的设计可能需要对使用更多的存储器与其他App优化之间做成平衡。

<br />

## 请注意OpenGL ES状态

OpenGL ES实现维护了一套复杂的状态数据，包括你用`glEnable`和`glDisable`函数所设置的开关，当前的着色器程序以及其uniform变量，当前绑定的纹理单元，以及当前绑定的顶点缓存以及它们所允许的顶点属性。硬件只有一个当前状态，它被惰性编译并cache起来。状态切换是昂贵的，因此将你的应用设计成最少化状态切换是最佳的。

不要去设置已经做过同样设置的状态。一旦某一特征被允许，它不需要再被重新允许。比如，如果你用同样的实参多次调用`glUniform`函数，那么OpenGL ES可能不会查看同样的uniform状态是否已经被设置过了。它只会简单地去更新该状态值，即便该值与当前值相同。

通过使用专门的设置或关闭例程来避免做一次不必要的状态设置，而不是将这些调用放在一个绘制循环中。设置与关闭例程对于打开和关闭一些特征以实现一种特定的视觉效果也很有用。比如，对一个具有纹理贴图的多边形周围绘制一个线框轮廓。

<br />

### 用OpenGL ES对象封装状态

为了减少状态改变，要将创建需要收集多个OpenGL ES状态的对象 改变为 可以被约束到一次函数调用的一个对象。比如，顶点数组对象（**VAO**）将多个顶点属性的配置存放到一单个对象中。

<br />

### 组织绘制调用以最少化状态改变

改变OpenGL ES状态不会立即产生效果。取而代之的是，当你发布一条绘制命令时，OpenGL ES执行所必要的工作来以一组状态值进行绘制。你可以通过最少化状态改变来减少CPU在重新配置图形流水线时所花费的时间。比如，在你App中保存一个状态向量，然后只有当你的状态在两次绘制调用之间发生改变时，才去设置相应的OpenGL ES状态。另一个有用的算法是**状态排序**——保持跟踪你需要做的绘制操作，以及每次绘制操作所需要的状态改变的量，然后对它们进行排序以至于可使用相同的状态连续地执行绘制操作。

iOS的OpenGL ES实现可以cache它所需要的某些配置数据，从而可以高效地在状态之间切换，但是对于每个独立状态设置的初始配置会花费更长的时间。为了实现一致性的性能，你可以在一个设置例程期间来“预热”你计划使用的每个状态：
1、允许你计划使用的状态配置或着色器。
2、使用状态配置来绘制一个平凡个数的顶点。
3、冲刷OpenGL ES上下文，使得在此预热阶段绘制不被显示。
