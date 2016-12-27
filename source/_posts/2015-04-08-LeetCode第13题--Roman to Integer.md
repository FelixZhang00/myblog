title: LeetCode第13题--Roman to Integer（Java实现）
tag: leetcode
category: 算法

---

## [原题](https://leetcode.com/problems/roman-to-integer/)
Given a roman numeral, convert it to an integer.

Input is guaranteed to be within the range from 1 to 3999.

### 解题思路
#### 先弄明白什么是罗马数字：
这7个符号与10进制阿拉伯数字的对应关系是：<br/>
I=1；X=10；C=100；M=1000；　<br/>
V=5；L=50；D=500；<br/>

罗马数字编码规则如下：

	1.编码最短，左减右加；
	2.加减不能跨“数量级”；
	3.减不过1，加不过3；	
	
#### 算法思路
找到字符串中最大的字符，找到其对应的数字，递归地做左减右加。

### 算法实现
```java

	public class Solution {
    private static final Map<Character, Integer> map = new HashMap<Character, Integer>();
    static {
        map.put('I', 1);
        map.put('V', 5);
        map.put('X', 10);
        map.put('L', 50);
        map.put('C', 100);
        map.put('D', 500);
        map.put('M', 1000);
    }


    public int romanToInt(String s) {
        return romanToIntSelf(s, 0, s.length());

    }

    private int romanToIntSelf(String s, int start, int end) {
        if (start >= end) return 0;
         //特别注意：result就在递归，第一个递归函数的result用来回收左右递归所返回的值，并累加中间的值。
        int result = 0;
        int maxIndex = findMaxIndex(s, start, end);
        char max = s.charAt(maxIndex);
        result += map.get(max);
        result += romanToIntSelf(s, maxIndex + 1, end);
        result -= romanToIntSelf(s, start, maxIndex);
        return result;
    }

    /**
	 * 找到字符串中表示最大的罗马数字支付串所在的位置
     *
     * @param s
     * @param start inclusive
     * @param end   exclusive
     */
    private int findMaxIndex(String s, int start, int end) {
        char temp = s.charAt(start);
        int result = start;
        for (int i = start + 1; i < end; i++) {
            if (bigger(s.charAt(i), temp)) {
                result = i;
                temp = s.charAt(i);
            }

        }
        return result;
    }

    /**
     * is c1 in romandigit bigger than c2
     *
     * @return
     */
    private boolean bigger(char c1, char c2) {
        return map.get(c1) > map.get(c2);
    }

    class RomanDigit implements Comparable<RomanDigit> {

        public RomanDigit() {
        }

        @Override
        public int compareTo(RomanDigit r) {
            return 0;
        }
    }

    public static void main(String[] args) {
        Solution solution = new Solution();
        String s = "XCIX"; //99
        System.out.println(solution.findMaxIndex(s, 0, s.length()));
        System.out.println(solution.romanToInt(s));

    }
	}

```