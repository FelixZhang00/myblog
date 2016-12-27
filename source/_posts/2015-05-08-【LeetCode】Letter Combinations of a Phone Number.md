title: 【LeetCode】Letter Combinations of a Phone Number
tag: leetcode
category: 算法

---

## [题目](https://leetcode.com/submissions/detail/27236262/)


在手机九宫格键盘上输入一串数字，给出可能打印出来的字符串的集合。


## 分析

- 先做一个map将数字映射到键盘上相应的字母集合。
- 把按键顺序看成深度优先遍历的深度，每次dfs将深度d+1直到d=按键字符串的长度未知，此时即完成了一次按键可能的输出。


## 实现

```java

	static Map<Integer, Character[]> map = new HashMap<Integer, Character[]>();   //数字键--键上的字母集合

    static {
        map.put(1, new Character[]{'\0'});
        map.put(2, new Character[]{'a', 'b', 'c'});
        map.put(3, new Character[]{'d', 'e', 'f'});
        map.put(4, new Character[]{'g', 'h', 'i'});
        map.put(5, new Character[]{'j', 'k', 'l'});
        map.put(6, new Character[]{'m', 'n', 'o'});
        map.put(7, new Character[]{'p', 'q', 'r', 's'});
        map.put(8, new Character[]{'t', 'u', 'v'});
        map.put(9, new Character[]{'w', 'x', 'y', 'z'});
        map.put(0, new Character[]{' '});

    }

    public List<String> letterCombinations(String digits) {
        ArrayList<String> result = new ArrayList<String>();
        if (digits == null || digits.length() == 0) return result;

        dfs(digits, 0, "", result);
        return result;
    }

    /**
     * 深度优先遍历，每次将数字键上的字母拼接到s中，一旦到达底部，则将s放入结果集中，并返回
     */
    private void dfs(String digits, int d, String s, ArrayList<String> result) {
        if (d == digits.length()) {
            result.add(s);
            return;
        }
        Integer number = Integer.parseInt(digits.charAt(d) + "");  //得到输入数字串在深度d时的数字

        for (Character c : map.get(number)) {                      //遍历该数字对应的每一个字母
            dfs(digits, d + 1, s + c, result);
        }
    }

```



