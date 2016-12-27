title: Programming Assignment 2 Seam Carving 暴力实现
tag: 算法
categotry: 算法
---
Robert Sedgewick教授在Coursera上开了一门[算法课](https://class.coursera.org/algs4partII-005)，这是图论中的一道[编程作业题](https://class.coursera.org/algs4partII-005/assignment/view?assignment_id=9)。

## 问题概述
图像由像素构成，可以看成是一张二维数组，其中的存储着Color，这样每个位置都有相应的颜色，就可以表示一张图片了。
这道题目的目的是resize图像，每次删除一行或一列颜色值最不明想的像素。
		
图像在二维数组中的表示 ：

	  (255,101,51)  	  (255,101,153)  	  (255,101,255)  
	  (255,153,51)  	  (255,153,153)  	  (255,153,255)  
      (255,203,51)  	  (255,204,153)  	  (255,205,255)  
      (255,255,51)  	  (255,255,153)  	  (255,255,255)  


##解题思路

### 能量函数
如何界定某个像素是否明显，可以被删除呢？
		
		是否明显是由周围的像素决定的，基于此有公式
		pixel（x,y）的能量函数表示为：
		Δx2(x, y) + Δy2(x, y)
		其中，Δx2(x, y) = Rx(x, y)2 + Gx(x, y)2 + Bx(x, y)2
		Rx、Gx、Bx分别为为pixel(x+1,y)与pixel(x-1,y)对应RGB的差值。
		Δy2(x, y)同理。

### 最短路径
我想到的暴力求解的笨办法是，将图像的位置映射成唯一的整型索引，并算出其能量函数的值，加上起始终点两个虚拟位置（它门的能量值都为0）以此构造一张加权有向图。找出起始点到终点的最短路径。

<img src="http://7viip0.com1.z0.glb.clouddn.com/Seam-Carving-picture_seam.png"  width="250" height="250"  style="margin-left: 0px"/>

### 顶点的权重
加权有向图的权重是指边的权重，而上面构造的图形的权重值是在顶点中表示的，这需要转化为边的权重。这很简单，只需要将将一条边的两个顶点的权重相加表示成边的权重即可。



##算法实现
### 能量函数
	```java
    private Picture mPicture; 
    private static final double BORDER_ENERGY = 255.0 * 255 * 3;
    /**
     * energy of pixel at column x and row y
     */
    public double energy(int x, int y) {
        if (x < 0 || x >= width() || y < 0 || y >= height()) throw new IndexOutOfBoundsException();
        if (isBorder(x, y)) return BORDER_ENERGY;

        return energyFun(mPicture.get(x - 1, y), mPicture.get(x + 1, y))
                + energyFun(mPicture.get(x, y - 1), mPicture.get(x, y + 1));

    }
      private double energyFun(Color color1, Color color2) {

        int r1 = color1.getRed();
        int g1 = color1.getGreen();
        int b1 = color1.getBlue();

        int r2 = color2.getRed();
        int g2 = color2.getGreen();
        int b2 = color2.getBlue();

        int delta = square(r1 - r2) + square(g1 - g2) + square(b1 - b2);

        return delta;
    }

    private int square(int i) {
        return i * i;
    }

    private boolean isBorder(int x, int y) {
        if (x == 0 || x == mPicture.width() - 1) return true;
        else if (y == 0 || y == mPicture.height() - 1) return true;
        else return false;

    }
    ````
    
###找到垂直方向的最短路径
	```java
    /**
     * sequence of indices for vertical seam
     *
     * @return
     */
    public int[] findVerticalSeam() {
        int[] result = new int[height()];

        //解法一：构造图
        EdgeWeightedDigraph verticalG = buildVerticalGraph(width(), height());
        AcyclicSP sp = new AcyclicSP(verticalG, 0);
        Iterable<DirectedEdge> edges = sp.pathTo(verticalG.V() - 1);
        int len = 0;
        for (DirectedEdge e : edges) {
			//StdOut.println(e); //调试 打印路径

            int v = e.from();
            if (v != 0 && v != (verticalG.V() - 1)) {
                result[len++] = convertToX(v);
            }
        }

        return result;
    }
    
     /**
     * 将图中标示的点映射到相应的x值
     *
     * @param v
     * @return
     */
    private int convertToX(int v) {
        return (v - 1) % width();
    }
    
    
        /**
     * //构造图，将二维矩阵转化为唯一标示的整数作为图的顶点
     * //顶点的权重转化为边的权重：一条边两个顶点的权重之和
     * //上下两个虚拟点的energy为0，这样把最终算出来的总权重之和除以2就是原来最短路径的顶点的权重之和了
     *
     * @param width
     * @param height
     */
    private EdgeWeightedDigraph buildVerticalGraph(int width, int height) {

        EdgeWeightedDigraph G = new EdgeWeightedDigraph(width * height + 2);
        for (int i = 0; i < G.V() - 1; i++) {  //上方的起始虚拟点
            if (i == 0) {
                for (int j = 1; j <= width; j++) {
                    G.addEdge(new DirectedEdge(0, j, 0 + energy(j - 1, 0)));
                }
            } else if (i >= G.V() - 1 - width) {  //最下方的所有点连接 下方的终点虚拟点
                G.addEdge(new DirectedEdge(i, G.V() - 1, energy((i - 1) % width, height - 1) + 0));
            } else {
                int fromX = (i - 1) % width;
                int fromY = (i - 1) / width;
                int toX = fromX; //正下方
                int toY = fromY + 1;
                if ((i - 1) % width == 0) { //最左边的点(以排除最下方的最左侧的点)
                    toX = fromX; //正下方
                    toY = fromY + 1;
                    double fromWeight = energy(fromX, fromY);
                    double toWeight = energy(toX, toY);
                    G.addEdge(new DirectedEdge(i, i + width, fromWeight + toWeight));

                    toX = fromX + 1; //右下方
                    toY = fromY + 1;
                    toWeight = energy(toX, toY);
                    G.addEdge(new DirectedEdge(i, i + width + 1, fromWeight + toWeight));
                } else if ((i - 1) % width == (width - 1)) {//最右边的点（(以排除最下方的点）
                    toX = fromX; //正下方
                    toY = fromY + 1;
                    double fromWeight = energy(fromX, fromY);
                    double toWeight = energy(toX, toY);
                    G.addEdge(new DirectedEdge(i, i + width, fromWeight + toWeight));

                    toX = fromX - 1; //左下方
                    toY = fromY + 1;
                    toWeight = energy(toX, toY);
                    G.addEdge(new DirectedEdge(i, i + width - 1, fromWeight + toWeight));
                } else { //一般的点都有3个有向边发出（(以排除最下方的）
                    toX = fromX; //正下方
                    toY = fromY + 1;
                    double fromWeight = energy(fromX, fromY);
                    double toWeight = energy(toX, toY);
                    G.addEdge(new DirectedEdge(i, i + width, fromWeight + toWeight));

                    toX = fromX - 1; //左下方
                    toY = fromY + 1;
                    toWeight = energy(toX, toY);
                    G.addEdge(new DirectedEdge(i, i + width - 1, fromWeight + toWeight));

                    toX = fromX + 1; //右下方
                    toY = fromY + 1;
                    toWeight = energy(toX, toY);
                    G.addEdge(new DirectedEdge(i, i + width + 1, fromWeight + toWeight));
                }
            }


        }
        return G;
    }
    ```
### 从图像中删除垂直最短路径（即最不明显的像素）
	```java
	 /**
     * remove vertical seam from current picture
     *
     * @param seam
     */
    public void removeVerticalSeam(int[] seam) {
        if (height() <= 1)
            throw new java.lang.IllegalArgumentException();
        Picture pic = new Picture(width() - 1, height());
        for (int h = 0; h < pic.height(); h++) {

            for (int w = 0; w < seam[h]; w++) {
                pic.set(w, h, mPicture.get(w, h));
            }

            for (int w = seam[h] + 1; w < width(); w++) {
                pic.set(w - 1, h, mPicture.get(w, h));

            }
        }
        this.mPicture = pic;
    }
		````

##后记
提交了n次，虽然通过了正确性测试，但是timing的5个测试全部没有通过，看样子此算法还是太过复杂了。

