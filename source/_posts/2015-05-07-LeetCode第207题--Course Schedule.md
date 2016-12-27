title: LeetCode第207题--Course Schedule
tag: leetcode
category: 算法

---

## [原题](https://leetcode.com/problems/course-schedule/)
给出课程总数，用不同整数编号表示不同课程，用一个二维数组表示多组先修课程的顺序对。

比如：有2门课，要学课程1必须先学课程0，这是有效的。

	2, [[1,0]]
	
如果有2门课，要学课程1必须先学课程0，要学课程0必须先学课程1，这是无效的。

	2, [[1,0],[0,1]]
	
	
需要完成的就是这个方法：

	```java
	
	public boolean canFinish(int numCourses, int[][] prerequisites) {
	
	}	
	```
	
### 测试数据：

	8, [[1,0],[2,6],[1,7],[5,1],[6,4],[7,0],[0,5],[5,1],[6,4]]
		
	
### 解题思路
由课程的编号构成一幅有向图，方向为先修顺序，构造完成后只要判断图是否有环。


### 有向图数据结构

初始化时构造一张图，但只有顶点个事，顶点之间没有边相连。

用邻接表表示有向图。

调用`addEdge()`方法可以将课程之间先修关系写到图中。

```java
	
	
    private class Digraph {

        private final int V;
        private ArrayList<Integer>[] adj;

        public Digraph(int V) {
            this.V = V;
            adj = (ArrayList<Integer>[]) new ArrayList[V];
            for (int v = 0; v < V; v++) {
                adj[v] = new ArrayList<Integer>();
            }
        }

        public void addEdge(int v, int w) {
            adj[v].add(w);
        }

        public int V() {
            return V;
        }

        public Iterable<Integer> adj(int v) {
            return adj[v];
        }

    }
	
	
```
	
	
### 判断有向图中是否有环

用一个数组`boolean[] onStack`保存递归调用期间栈上的所有顶点.

onStack[v]=true,记录顶点v出现在这次dfs中，在这次dfs结束后，是onStack[v]=false

在递归执行dfs的过程中，记录当前下当前顶点在递归调用栈中，这样以后的递归调用栈只要判断它的相连点是否在之前的递归调用栈中出现过，就能判断是否有环。


```java


    private class DirectedCycle {
        private boolean[] marked;        // marked[v] 顶点v是否被访问过
        private boolean[] onStack;       //保存递归调用期间栈上的所有顶点。 onStack[v] = ？顶点v是否在栈中
        private boolean hasCycle;       // 有向图中是否环

        public DirectedCycle(Digraph G) {
            marked = new boolean[G.V()];
            onStack = new boolean[G.V()];
            hasCycle = false;
            for (int v = 0; v < G.V(); v++)   //对图中每一个没有被访问过的点做深度优先遍历
                if (!marked[v]) dfs(G, v);
        }


        /**
         * 从v顶点开始做深度优先遍历
         */
        private void dfs(Digraph G, int v) {
            onStack[v] = true;
            marked[v] = true;
            for (int w : G.adj(v)) {        //遍历所有顶点v的相连点
                if (hasCycle) return;       //如果已经找到一个环就不再dfs
                else if (!marked[w]) {      //对每一个未访问过的点继续dfs
                    dfs(G, w);
                } else if (onStack[w]) {    //顶点w在之前的递归调用栈中，并且已经被访问过，构成环
                    hasCycle = true;
                }
            }
            onStack[v] = false;             //顶点v所有的相连点遍历结束,顶点v退出当前调用栈
        }

        public boolean hasCycle() {
            return hasCycle;
        }
    }


```


### 主方法

```java

	 public boolean canFinish(int numCourses, int[][] prerequisites) {
        Digraph G = new Digraph(numCourses);
        for (int i = 0; i < prerequisites.length; i++) {
            for (int j = 0; j < prerequisites[i].length; j++) {
                G.addEdge(prerequisites[i][j], prerequisites[i][++j]);
            }
        }
        DirectedCycle dag = new DirectedCycle(G);

        return !dag.hasCycle();

    }

```
	
