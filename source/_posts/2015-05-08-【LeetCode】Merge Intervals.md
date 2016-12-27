title: 【LeetCode】Merge Intervals
tag: leetcode
category: 算法

---

## [题目](https://leetcode.com/problems/merge-intervals/)

	Given a collection of intervals, merge all overlapping intervals.

	For example,
	Given [1,3],[2,6],[8,10],[15,18],
	return [1,6],[8,10],[15,18].
	
	
## 分析
先对intervals集合按start从小到大排序，last变量用于保存可能插入到结果集中的元素，遍历每一个集合中的元素，如果符合合并的条件，将last和当前元素合并,并重新赋值给last，此时last仍然具有合并的潜力；如果不符合合并的条件，则将last放入结果集中，并把当前元素赋值给last，成为一个新的潜在具有合并性的元素。


### 自定义比较类

```java

	 //      Definition for an interval
    public class Interval {
        int start;
        int end;

        Interval() {
            start = 0;
            end = 0;
        }

        Interval(int s, int e) {
            start = s;
            end = e;
        }
    }

  	public static final Comparator<Interval> BY_START = new ByStart();

    private static class ByStart implements Comparator<Interval> {
        @Override
        public int compare(Interval o1, Interval o2) {
            return o1.start - o2.start;
        }
    }

```


### 主方法

```java

 		public List<Interval> merge(List<Interval> intervals) {
        ArrayList<Interval> result = new ArrayList<Interval>();

        if (intervals == null || intervals.size() == 0) {
            return result;
        }

        //按Interval的start对intervals排序
        Collections.sort(intervals, BY_START);

        Interval last = intervals.get(0);
        for (int i = 1; i < intervals.size(); i++) {
            Interval temp = intervals.get(i);
            if (canMerge(last, temp)) {
                if (last.end <= temp.end) {
                    last = new Interval(last.start, temp.end);
                }
                //另外一种情况last保持不变
            } else {
                result.add(last);
                last = intervals.get(i);
            }

        }
        result.add(last);
        return result;
    }

    private boolean canMerge(Interval item1, Interval item2) {
        if (item1.end >= item2.start) {
            return true;
        }
        return false;
    }

```


