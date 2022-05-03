# Leetcode

## 盛水最多的容器

[盛水最多的容器](https://leetcode-cn.com/problems/container-with-most-water/)

双指针。

## 最长公共子序列

[link](https://leetcode-cn.com/problems/qJnOS7/)

```java
int processLongestCommonSubsequence(char[] str1, char[] str2, int i, int j) {
        // 处理到结尾了, 最多还能匹配0个
        if (i == str1.length || j == str2.length) {
            return 0;
        }
        // str1[i] == str2[j] i和j都右移一位, 且当前多匹配成功一个
        if (str1[i] == str2[j]) {
            return 1 + processLongestCommonSubsequence(str1, str2, i + 1, j + 1);
        } else {
            return Math.max(processLongestCommonSubsequence(str1, str2, i + 1, j),
                    processLongestCommonSubsequence(str1, str2, i, j + 1));
        }
}

	  // 记忆化搜索
		// f(i, j) = { 1 + f(i + 1, j + 1),             s[i] == s[j]
    //           { max(f(i + 1, j), f(i, j + 1))    s[i] != s[j]
    int processLongestCommonSubsequence(char[] str1, char[] str2, int i, int j,
                                        int[][] dp) {
        if (i == str1.length || j == str2.length) {
            return 0;
        }
        if (dp[i][j] != 0) {
            return dp[i][j];
        }
        if (str1[i] == str2[j]) {
            dp[i + 1][j + 1] = processLongestCommonSubsequence(str1, str2, i + 1, j + 1
                    , dp);
            return 1 + processLongestCommonSubsequence(str1, str2, i + 1, j + 1, dp);
        } else {
            dp[i + 1][j] = processLongestCommonSubsequence(str1, str2, i + 1, j, dp);
            dp[i][j + 1] = processLongestCommonSubsequence(str1, str2, i, j + 1, dp);
            return Math.max(
                    dp[i + 1][j],
                    dp[i][j + 1]);
        }

}


public int longestCommonSubsequence(String text1, String text2) {
        char[] str1 = text1.toCharArray();
        char[] str2 = text2.toCharArray();
        int[][] dp = new int[str1.length + 1][str2.length + 1];

        for (int i = str1.length - 1; i >= 0; i--) {
            for (int j = str2.length - 1; j >= 0; j--) {
                // str1[i] == str2[j] i和j都右移一位, 且当前多匹配成功一个
                if (str1[i] == str2[j]) {
                    dp[i][j] = 1 + dp[i + 1][j + 1];
                } else {
                    dp[i][j] = Math.max(dp[i + 1][j], dp[i][j + 1]);
                }
            }
        }

        return dp[0][0];
}
```

根据递归的版本可以画出下面的递推图。

![image-20220317190438068](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/dp_long_sub_str.png)

## 无重复字符的最长字串

[link](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/)

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        if ("".equals(s)) {
            return 0;
        }
        if (s.length() == 1) {
            return 1;
        }

        HashSet<Character> set = new HashSet<>();
        int maxLen = Integer.MIN_VALUE;
        int i = 0;
        int j = 1;
        char[] chars = s.toCharArray();
        set.add(chars[0]);
        int curLen = 1;

        while (i <= j && j < chars.length) {
            // 如果子串里没有 s[j] ，j 右移
            if (!set.contains(chars[j])) {
                set.add(chars[j]);
                j++;
                curLen++;
            } else { // 包含了说明有重复了 i 右移
                
                curLen--;
                set.remove(chars[i]);
                i++;       
            }
            maxLen = Math.max(maxLen, curLen);
        }
        return maxLen;
    }
}
```

## 两数之和

HashMap

## 三数之和

排序+双指针

## LRU缓存

[link](https://leetcode-cn.com/problems/lru-cache/)

```java
import java.util.HashMap;

public class LRUCache {
    final HashMap<Integer, DNode> map;
    final int capacity;
    int size;

    // 假头尾节点
    DNode head;
    DNode tail;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        map = new HashMap<>((int) (capacity / 0.75F));
        size = 0;

        head = new DNode();
        tail = new DNode();

        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        DNode node = map.get(key);
        if (node != null) {
            // 当前不是头才移动,否则不用移动
            if (head.next != node) {
                moveToHead(node);
            }
            return node.val;
        }
        return -1;
    }

    public void put(int key, int value) {
        DNode node = map.get(key);
        if (node != null) { // key存在
            node.val = value;
            moveToHead(node);
        } else {
            // 新建一个,放到队列和map中
            node = new DNode(key, value);
            map.put(key, node);
            addToHead(node);
            ++size;
            if (size > capacity) { // 删除
                DNode last = removeTail();
                map.remove(last.key);
                --size;
            }
        }
    }

    void removeNode(DNode node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    void addToHead(DNode node) {
        node.next = head.next;
        node.prev = head;

        node.next.prev = node;
        head.next = node;
    }

    void moveToHead(DNode node) {
        removeNode(node);
        addToHead(node);
    }

    DNode removeTail() {
        DNode t = tail.prev;
        removeNode(t);

        return t;
    }


    static class DNode {
        int val;
        int key;

        DNode prev;
        DNode next;

        public DNode() {
        }

        public DNode(int key, int val) {
            this.val = val;
            this.key = key;
        }
    }



}
```

## [电话号码的字母组合](https://leetcode-cn.com/problems/letter-combinations-of-a-phone-number/)

```java
public class T17 {

    static HashMap<Character, String> map = new HashMap<>();

    static {
        map.put('2', "abc");
        map.put('3', "def");
        map.put('4', "ghi");
        map.put('5', "jkl");
        map.put('6', "mno");
        map.put('7', "pqrs");
        map.put('8', "tuv");
        map.put('9', "wxyz");
    }

    public List<String> letterCombinations(String digits) {
        List<String> ans = new ArrayList<>();

        if ("".equals(digits)) {
            return ans;
        }
        process(digits.toCharArray(), 0, ans);
        return ans;
    }

    void process(char[] chars, int i, List<String> res) {
        if (i == chars.length) {
            String val = String.valueOf(chars);
            res.add(val);
            return;
        }
        char cur = chars[i];

        String mapper = map.get(cur);
        for (char c : mapper.toCharArray()) {
            chars[i] = c;
            process(chars, i + 1, res);
        }
        chars[i] = cur;
    }
}
```

## 括号生成

```java
public class T22 {
    public static void main(String[] args) {
        System.out.println(new T22().generateParenthesis(3));
    }

    public List<String> generateParenthesis(int n) {
        List<String> ans = new ArrayList<>();
        char[] chars = new char[2 * n];
        process(0, 0, chars, ans, n);
        return ans;
    }

    // 处理
    void process(int left, int right, char[] chars, List<String> ans, int n) {
        // left == right  == n 找到一种
        if (left == n && right == n) {
            ans.add(String.valueOf(chars));
        } else if (left == n) { // 只能生成右括号
            chars[left + right] = ')';
            process(left, right + 1, chars, ans, n);
        } else if (left == right) { // 只能生成左括号
            chars[left + right] = '(';
            process(left + 1, right, chars, ans, n);
        } else {
            // 生成左括号
            chars[left + right] = '(';
            process(left + 1, right, chars, ans, n);
            chars[left + right] = ')';
            process(left, right + 1, chars, ans, n);
        }

    }

}
```

## 最大子数组和



## [合并K个升序链表](https://leetcode-cn.com/problems/merge-k-sorted-lists/)

```java
public ListNode mergeKLists(ListNode[] lists) {
        if (lists.length == 0) {
            return null;
        }
        PriorityQueue<ListNode> queue = new PriorityQueue<>(new Comp());

        for (ListNode node : lists) {
            while (node != null) {
                queue.offer(node);
                node = node.next;
            }
        }
        ListNode ans = queue.poll();
        if (ans == null) {
            return null;
        }
        ListNode tmp = ans;
        while (!queue.isEmpty()) {
            tmp.next = queue.poll();
            tmp = tmp.next;
        }
        tmp.next = null;

        return ans;
    }

    static class Comp implements Comparator<ListNode> {

        @Override
        public int compare(ListNode o1, ListNode o2) {
            return o1.val - o2.val;
        }
    }
```

## 最大子数组和



```java
class Solution {
    public int maxSubArray(int[] nums) {
        int[] dp = new int[nums.length];
        dp[0] = nums[0];
        int curMax = dp[0];

        for (int i = 1; i < nums.length; i++) {
            dp[i] = Math.max(nums[i], dp[i - 1] + nums[i]);
            if (dp[i] > curMax) {
                curMax = dp[i];
            }
        }


        return curMax;
    }
}
```

