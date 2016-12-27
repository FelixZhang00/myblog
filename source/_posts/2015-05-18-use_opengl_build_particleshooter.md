title: 用OpenGL构建粒子喷泉
tag: OpenGL
categotry: Android
---

## 效果展示
<img src="http://7viip0.com1.z0.glb.clouddn.com/particles-1.gif"/>

这是[《OpenGL ES应用开发实践指南》](http://book.douban.com/subject/25979507/)中的一个例子，写这篇blog简单总结下在Android上进行OpenGL ES开发的方法。


## 工作流程概述
	
	定义顶点着色器、片段着色器。
	
	
	
##在哪里画图
在Activity中设置ContentView为GLSurfaceView，在该控件上设置自定义渲染器Renderer完成OpenGL绘图。
Renderer接口定义的方法：
onSurfaceCreated(GL10 gl10, EGLConfig eglConfig)
在Surface被创建时调用。

onSurfaceChanged(GL10 gl10, int width, int height)
每次Suface尺寸变化时被调用，包括第一次刚创建时。

onDrawFrame(GL10 gl10)
当绘制一帧时会被调用，比如一秒钟会被调用执行60次。


##如何告诉GPU绘制信息
### 把内存从java堆复制到本地堆
图形有顶点和颜色构成，将这些信息存放在一个数组中，并且需要将java数组转移到本地数组中,可以使用这个工具类[VertexArray](https://github.com/FelixZhang00/My_Particles2/blob/master/app/src/main/java/me/felixzhang/example/my_particles2/data/VertexArray.java)
```java

/**
 * Created by felix on 15/5/19.
 * 负责将内存从java堆复制到本地堆。
 * 关联属性与顶点数据，告诉OpenGL去哪里找属性对应的数据。
 */
public class VertexArray {

    private final FloatBuffer floatBuffer;

    public VertexArray(float[] vertexData) {
        this.floatBuffer = ByteBuffer.allocateDirect(vertexData.length * BYTE_PER_FLOAT)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();
        floatBuffer.put(vertexData);
    }


    public void setVertexAttribPointer(int dataOffset, int attributeLocation, int componentCount, int stride) {
        floatBuffer.position(dataOffset);
        glVertexAttribPointer(attributeLocation, componentCount, GL_FLOAT, false, stride, floatBuffer);
        glEnableVertexAttribArray(attributeLocation);
        floatBuffer.position(0);
    }

    /**
     * 在原有数组的基础上更新指定范围的元素，如果全部复制的话速度太慢
     *
     * @param vertexData
     * @param start
     * @param count
     */
    public void updateBuffer(float[] vertexData, int start, int count) {
        floatBuffer.position(start);
        floatBuffer.put(vertexData, start, count);
        floatBuffer.position(0);
    }
}

```

###还需要着色器
告诉GPU如何绘制数据，数据在着色器这一管道中传递。

着色器中变量的解释

	uniform：会让每个顶点都使用同一个值，不需要对每个顶点设置，除非我们再次改变它。
	attribute：把顶点属性放进着色器的手段，每个顶点都要设置一次
	varying：不需要设置，共顶点着色器和片段着色器之间共享数据。
	
下面是一个顶点着色器

```c

	uniform mat4 u_Matrix;
	uniform float u_Time;                   //当前系统的时间

	attribute vec3 a_Position;
	attribute vec3 a_Color;
	attribute vec3 a_DirectionVector;
	attribute float a_ParticleStartTime;        //例子创建的时间


	varying float v_ElapseTime;
	varying vec3 v_Color;

	void main() {
    v_Color=a_Color;
    v_ElapseTime=u_Time-a_ParticleStartTime;
    vec3 currentPosition=a_Position+(a_DirectionVector*v_ElapseTime);

    gl_Position=u_Matrix*vec4(currentPosition,1.0);
    gl_PointSize=10.0;

	}
```

下面是一个片段着色器：
告诉GPU每个片段最终颜色是什么，对于基本图元的每个片段都会被调用一次。

```c

	precision mediump float;

	varying vec3 v_Color;
	varying float v_ElapseTime;

	void main() {

	gl_FragColor=vec4(v_Color/v_ElapseTime,1.0);

	}
```



###如何让OpenGL画图
当调用下面的方法时，OpenGL就会从缓冲区读数据，每读取完一组数据就会调用一次main方法，并把数据填到attribute对应的变量中。
```java
	glDrawArrays(GL_POINTS, 0, currentParticleCount);
```
着色器main方法中的gl_Position和gl_PointSize是OpenGL中的变量，也就是最终给GPU的信息。


##编译着色器
glsl文件需要编译链接成OpenGL的一个程序才能使用。
需要使用这几个[工具类](https://github.com/FelixZhang00/My_Particles2/tree/master/app/src/main/java/me/felixzhang/example/my_particles2/util)。
<img src="http://7viip0.com1.z0.glb.clouddn.com/opengl-particlesshooter-compile-tool.jpg"/>


##构建粒子系统
<img src="http://7viip0.com1.z0.glb.clouddn.com/particles-class-diagram.jpg"/>

[ParticlesRenderer](https://gist.github.com/FelixZhang00/bc6c7d4adc98319359b7)

[ParticlesShooter](https://gist.github.com/FelixZhang00/353d9d25a853341d6623)

[ParticlsSystem](https://gist.github.com/FelixZhang00/1d010a3f4af23f348b6c)

###向粒子系统中填充数据


```java

	 /**
     * 向系统中添加粒子，每次添加一个
     *
     * @param positionPoint 新加粒子的位置
     */
    public void addParticles(Point positionPoint, int color, Vector direction, float particleStrtTime) {

        final int particleOffset = nextParticleOffset * TOTAL_COMPONENT_COUNT;  //记住新粒子从数组的哪个编号开始
        int currentOffset = particleOffset;                       //记住新粒子的每个属性从哪里开始

        nextParticleOffset++;
        if (currentParticlesCount < maxParticlesCount) {
            currentParticlesCount++;
        }
        //当超出数组范围时，将下一个粒子放在数组的开头位置，达到回收的目的
        if (nextParticleOffset == maxParticlesCount) {
            nextParticleOffset = 0;
        }


        //把新粒子的数据写到数组中
        particles[currentOffset++] = positionPoint.x;
        particles[currentOffset++] = positionPoint.y;
        particles[currentOffset++] = positionPoint.z;

        particles[currentOffset++] = Color.red(color) / 255f;       //OpenGL需要[0,1)的颜色值
        particles[currentOffset++] = Color.green(color) / 255f;
        particles[currentOffset++] = Color.blue(color) / 255f;


        particles[currentOffset++] = direction.x;
        particles[currentOffset++] = direction.y;
        particles[currentOffset++] = direction.z;


        particles[currentOffset++] = particleStrtTime;

        vertexArray.updateBuffer(particles, particleOffset, TOTAL_COMPONENT_COUNT);
    }


```



通过发射器向粒子系统中添加数据
```java

	public class ParticlesShooter {

    //确定粒子发射器的位置，方向和颜色
    private final Point position;
    private final Vector direction;
    private final int color;

    public ParticlesShooter(Point position, Vector direction, int color) {
        this.position = position;
        this.direction = direction;
        this.color = color;
    }

    public void addParticles(ParticlsSystem particlsSystem, float currentTime, int count) {

        for (int i = 0; i < count; i++) {
            particlsSystem.addParticles(position,color,direction,currentTime);
        }
    }
	}

```




最后在ParticlesRenderer中加入一些调用统一管理这一切。

[项目地址](https://github.com/FelixZhang00/My_Particles2)



