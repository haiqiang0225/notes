# 算法

## leetcode&zcy

### 给定一个有序数组Arr，代表数轴上从左到右的n个点arr[0] arr[1]，给定一个正整数L，代表绳子的长度，求最多能覆盖几个点

```java

```



### 买苹果，只有大小为6的袋子和为8的袋子



### 有先手跟后手两只动物吃草，每只只能吃4^i数量的草，给定数字N代表共有多少草，谁先吃完草谁赢

```java
public String winner(int n) {
        // 0 1 2 3 4 5
        if (n < 5) {
            return (n == 0 || n == 2) ? "后手" : "先手";
        }
        // n >= 5
        int base = 1;
        while (base <= n) {
            if (winner(n - base).equals("后手")) {
                return "先手";
            }
            if (base > n / 4) {
                break;
            }
            base *= 4;
        }
        return "后手";
    }
```

### 染色，有红色R跟绿色G两种颜色，G要比R更靠近左侧，请返回最少的染色数量

1) 左侧0位，右侧n位，需要将右侧的R全部染成G，返回右侧有多少R
2) 左侧1位，右侧n-1位，需要将左侧的R染成G，右侧的G染成R，返回左侧有多少R+右侧有多少G
3) 左侧i位，右侧n-i位，同2

预处理，用两个数组保存左侧i个中R的数量和右侧i个中G的数量。

```java
public int minPathTest(String s) {
        char[] str = s.toCharArray();
        int n = str.length;
        int[] lCountsR = new int[str.length];
        int[] rCountsG = new int[str.length];
        lCountsR[0] = str[0] == 'R' ? 1 : 0;
        rCountsG[0] = str[n - 1] == 'G' ? 1 : 0;

        for (int i = 1; i < str.length; i++) {
            if ('R' == str[i]) {
                lCountsR[i] = lCountsR[i - 1] + 1;
            } else {
                lCountsR[i] = lCountsR[i];
            }
        }

        int minVal = Integer.MAX_VALUE;
        for (int i = 0; i <= str.length; i++) {
            if (i == 0) {
                // 只统计i右侧有多少G
                minVal = Math.min(minVal, rCountsG[i]);
            } else if (i == str.length) {
                // 只统计i左侧
                minVal = Math.min(minVal, lCountsR[i - 1]);
            } else {
                // 既统计i左侧,又统计i右侧
                minVal = Math.min(minVal, minVal);
            }
        }

        return minVal;
    }
```





### 给定一个N*N的矩阵，只有0和1两种值，返回边框全是1的最大正方形的边长长度。

right[] -> 保存每个点右侧有多少个连续的1

```java
public int maxAllOneBorder(int[][] matrix) {
        if (matrix.length == 0 || matrix[0].length == 0) {
            return 0;
        }

        int nRow = matrix.length;
        int nCol = matrix[0].length;

        int maxVal = Integer.MIN_VALUE;

        for (int row = 0; row < nRow; row++) {
            for (int col = 0; col < nCol; col++) {
                int lenMin = Math.min(nCol - col,  nRow - row);
                for (int len = 1; len < lenMin; len++) {
                  // 枚举
                }
            }public int maxAllOneBorder(int[][] matrix) {
        if (matrix.length == 0 || matrix[0].length == 0) {
            return 0;
        }

        int nRow = matrix.length;
        int nCol = matrix[0].length;

        int maxVal = Integer.MIN_VALUE;

        for (int row = 0; row < nRow; row++) {
            for (int col = 0; col < nCol; col++) {
                int lenMin = Math.min(nCol - col,  nRow - row);
                for (int len = 1; len < lenMin; len++) {
                  // 枚举
                }
            }
        }
        return maxVal;
    }
        }
        return maxVal;
    }


public int maxAllOneBorder(int[][] matrix) {
        if (matrix.length == 0 || matrix[0].length == 0) {
            return 0;
        }

        int nRow = matrix.length;
        int nCol = matrix[0].length;
  			
  			// 当前点右侧有多少个连续的1
  			int[][] maxRight = new int[nRow][nCol];
  		  // 当前点下方有多少个连续的1
  		  int[][] maxDown = new int[nRow][nCol];

        int maxVal = Integer.MIN_VALUE;

        for (int row = 0; row < nRow; row++) {
            for (int col = 0; col < nCol; col++) {
                // 判断当前点右侧和下侧，最多有多少个连续的1，求他俩里的最小值m
                // 找到当前点的右侧移动m点，看它下方有多少，下方同样
            }
        }
        return maxVal;
    }
```



### 给定一个函数f，可以返回a~b之间等概率的数字。请加工出一个可以返回c~d之间等概率数字的函数。

先能生成0-1的数字，再

```java
```

### 给定一个非负整数n，代表二叉树的结点个数。返回能生成多少种不同的二叉树结构。

f(n):返回n个节点形成多少个二叉树。

- 没有节点：形成一种结构，空树
- 一个节点：一种结构，就一个点
- 2个节点： 2种结构
- n个节点：
  - 左树没有：形成f(n-1)种
  - 左树有一个，形成f(n-2)种
  - 左树有两个，形成f(2)* f(n-3)
  - ...f(i)*f(n-i-1)

```java
public class TreeCount {
    // 给定n个结点, 返回能生成二叉树的个数
    static int treeCount(int n) {
        if (n < 0) {
            return 0;
        }
        if (n == 0) {
            return 1;
        }
        if (n == 1) {
            return 1;
        }
        if (n == 2) {
            return 2;
        }
        int ans = 0;
        for (int leftNum = 0; leftNum <= n - 1; leftNum++) {
            int leftWays = treeCount(leftNum);
            int rightWays = treeCount(n - leftNum - 1);
            ans += leftWays * rightWays;
        }

        return ans;
    }

    static int treeCountDp(int n) {
        if (n < 2) {
            return 1;
        }
        int[] dp = new int[n + 1];
        dp[0] = 1;


        // 节点总数
        for (int i = 1; i <= n; i++) {
            // 左侧节点数为leftNum, 右侧节点数为 i - left - 1
            for (int leftNum = 0; leftNum < i; leftNum++) {
                dp[i] += dp[leftNum ] * dp[i - leftNum - 1];
            }
        }

        return dp[n];
    }

    public static void main(String[] args) {
        long timeBegin = System.currentTimeMillis();
        treeCount(100);
        long timeEnd = System.currentTimeMillis();
        System.out.println(timeEnd - timeBegin);

        timeBegin = System.currentTimeMillis();

        treeCountDp(100);

        timeEnd = System.currentTimeMillis();
        System.out.println(timeEnd - timeBegin);

    }
}
```



 ### 完整的括号字符串

- 空字符是完整的
- 如果s是完整的，则(s)也是完整的
- 如果s和t都是完整的，则将它们连起来也是完整的

牛牛有一个括号字符串s，现在需要在其中任意位置尽量少的添加括号，将其转化为一个完整的括号字符串。请问牛牛至少要添加多少个括号。

思路：遍历一遍，如果出现负数就说明前面需要一个左括号，那么就加上一个。最后left的值就是有多少个左括号没匹配，因此最后需要再加上左括号的值。

#### 括号匹配

使用一个left变量来记录左括号出现的次数，每次遇到左括号就加一，遇到右括号就减一。如果中途出现负数，说明右括号出现在了左括号前面，如果遍历完left！=0，也说明不匹配。

### 给定一个数组arr，求差值为k的去重数字对

 例子：

```[3， 2， 5， 7， 0, 0]```, ```k=2```

返回：`[0, 2]`,`[3, 5]`,`[5, 7]`

思路：Hash表

### 给定包含n个元素的集合a，m个元素的集合b，最多进行多少次magic操作

magic操作：从一个集合里拿出某个数，放到另一个集合里，使得两个集合的平均值都变大。

- 不可以把集合取空
- 取出来的数要确保在另一个集合里不存在

 

### 合法的括号匹配序列

![image-20220320183250303](/Users/hq/Documents/md_image/algorithm_zcy/1_1_11.png)

left，遇到(  left ++ 否则 left--  left最大值就是答案。

### 最长有效括号子串

字串要求连续。

**遇到连续字串/子数组问题，求以每个位置为结尾的情况下，答案是多少。**

- 如果当前是左括号，那么以当前位置结尾的有效括号长度为0
- 如果当前是右括号，前一个字符有可能是左括号，有可能是右括号
  - 如果前一个是左括号，那么当前位置的长度为0 + 2
  - 如果前一个是右括号，那么当前位置的长度需要根据前一个有效匹配了的字串的前一个位置。

```java
package algorithm.leetcode;

public class T32 {
    public int longestValidParentheses(String s) {
        char[] str = s.toCharArray();
        int[] dp = new int[str.length];
        int curMax = 0;

        int pre = 0;
        for (int i = 1; i < str.length; i++) {
            if (str[i] == ')') {
                pre = i - dp[i - 1] - 1;
                if (pre >= 0 && str[pre] == '(') {
                    dp[i] += dp[i - 1] + 2 + (pre > 0 ? dp[pre - 1] : 0);
                }
            }
            curMax = Math.max(curMax, dp[i]);
        }

        return curMax;
    }
}
```



### 只使用一个额外栈，将另一个栈排序

互相弹，维护辅助栈的顺序。

### 将给定的数转换为字符串

1对应a，2对应b...，26对应z。

给定一个字符串，求出可以转换的不同字符串的个数。 

```java
int process(char[] str, int i) {
        if (i == str.length) {
            return 1;
        }

        if (str[i] == '0') {
            return 0;
        }

        // 让当前字符自己转换
        int ans = process(str, i + 1);

        // 没有后续字符则只有一种转换
        if (i == str.length - 1) {
            return ans;
        }

        if (((str[i] - '0') * 10 + (str[i - 1] - '0')) <= 26) {
            ans += process(str, i + 2);
        }

        return ans;
}

public int numDecodings(String s) {
        if (s.length() <= 0) {
            return 0;
        }

        char[] str = s.toCharArray();
        int n = str.length;
        int[] dp = new int[n + 1];

        dp[n] = 1;
        dp[n - 1] = str[n - 1] == '0' ? 0 : 1;

        for (int i = n - 2; i >= 0; i--) {
            if (str[i] == '0') {
                dp[i] = 0;
            } else {
                dp[i] = dp[i + 1]
                        + ((((str[i] - '0') * 10 + (str[i + 1] - '0')) <= 26) ?
                        dp[i + 2] : 0);

            }
        }

        return dp[0];
}
```



### 二叉树每个结点都有一个int型权值，给定一个二叉树，要求计算出从根节点到叶结点的所有路径中，权值和最大的值为多少？

```java
public int getMaxPath(Node root) {
  	// 已经到达叶节点，直接返回值
    if (root.left == null && root.right == null) {
        return root.val;
    }
  	int ans = 0;
  	if (root.left != null) {
      	ans += getMaxPath(root.left);
    }
  	if (root.right != null) {
      	ans += Math.max(ans, getMaxPath(root.right));
    }
  	return ans;
}
```



### 判断某个数是否在非负整数二维数组中

![image-20220321085604547](/Users/hq/Documents/md_image/algorithm_zcy/1_1_16.png)

 [leetcode](https://leetcode-cn.com/problems/sorted-matrix-search-lcci/)

【思路一】：从右上角开始出发找

- 如果下方的数大于aim，则aim肯定不在下方所有位置
- 如果左侧的数小于aim，则左侧不可能有aim

```java
public boolean searchMatrix(int[][] matrix, int target) {
        if (matrix.length == 0 || matrix[0].length == 0) {
            return false;
        }
        int nRow = matrix.length;
        int nCol = matrix[0].length;
        int x = 0;
        int y = nCol - 1;

        while (y >= 0 && x <= nRow - 1) {
            if (matrix[x][y] > target) {
                y--;
            } else if (matrix[x][y] < target) {
                x++;
            } else {
                return true;
            }
        }
        return false;
}
```



### [超级洗衣机](https://leetcode-cn.com/problems/super-washing-machines/)

![image-20220321095101176](/Users/hq/Documents/md_image/algorithm_zcy/1_1_17.png)



衣服总数%洗衣机数量 ！= 0，一定没法分，结束。

对于i位置的洗衣机：

- 如果左侧为-x，右侧为-y，表示左侧需要x件，右侧需要y件才能平衡。因此需要从i位置给出衣服，i位置每次能给1件，所以至少需要x+y轮
- 如果左侧为x，右侧为y，表示左侧多了x件，右侧多了y件。至少需要max(x, y)次，因为左侧右侧都能给出衣服。
- 左负-x右正y，至少需要max(x, y)次。
- 左正右负，同上

求出每个位置至少需要移动衣服的次数，总的轮数等于该位置的轮数。

```java
public int findMinMoves(int[] machines) {
        if (machines == null || machines.length == 0) {
            return -1;
        }
        int sum = 0;
        int size = machines.length;
        for (int i : machines) {
            sum += i;
        }
        // 一定凑不出来
        if (sum % size != 0) {
            return -1;
        }
        int average = sum / size;

        int maxMove = -1;
        int leftSum = 0;
        for (int i = 0; i < size; i++) {
            int leftNeed = leftSum - i * average;
            int rightNeed = (sum - leftSum - machines[i]) - (size - i - 1) * average;
            if (leftNeed < 0 && rightNeed < 0) {
                maxMove = Math.max(maxMove, Math.abs(leftNeed + rightNeed));
            } else {
                maxMove = Math.max(maxMove, Math.max(Math.abs(leftNeed),
                        Math.abs(rightNeed)));
            }
            leftSum += machines[i];
        }

        return maxMove;
}
```

### 用zigzag方式打印矩阵

![image-20220321105920454](/Users/hq/Documents/md_image/algorithm_zcy/1_1_18.png)

宏观调度打印。

使用两个点，(x0, y0)，(x1, y1)，第一个点一直向右走只要不越界，第二个点一直往下走，只要不越界。如果要越界了，改变移动方向。这两个点就能确定打印的斜线。

[Z 字形变换](https://leetcode-cn.com/problems/zigzag-conversion/)

```java
public String convert(String s, int numRows) {
    if (numRows == 1 || s.length() < numRows) {
        return s;
    }
    StringBuilder[] stringBuilders = new StringBuilder[numRows];
    char[] str = s.toCharArray();
    int cur = 0;
    boolean up = false;
    for (char c : str) {
        if (stringBuilders[cur] == null) {
            stringBuilders[cur] = new StringBuilder();
        }
        stringBuilders[cur].append(c);
        if (up && cur == 0) {
            up = false;
        } else if (!up && cur == numRows - 1) {
            up = true;
        }
        if (up) {
            cur--;
        } else {
            cur++;
        }
    }
    StringBuilder ans = new StringBuilder();
    for (StringBuilder sb : stringBuilders) {
        ans.append(sb.toString());
    }
    return ans.toString();
}
```

[螺旋矩阵 III](https://leetcode-cn.com/problems/spiral-matrix-iii/)

```java


```

### 用螺旋的方式打印矩阵

```java
public List<Integer> spiralOrder(int[][] matrix) {
    if (matrix == null || matrix.length == 0 || matrix[0].length == 0) {
        return new ArrayList<>();
    }
    List<Integer> ans = new ArrayList<>(matrix.length * matrix[0].length);

    int x0 = 0;
    int y0 = 0;
    int x1 = matrix.length - 1;
    int y1 = matrix[0].length - 1;
    while (x0 <= x1 && y0 <= y1) {
        print(matrix, x0++, y0++, x1--, y1--, ans);
    }

    return ans;
}

void print(int[][] matrix, int x0, int y0, int x1, int y1, List<Integer> ans) {
    if (x0 == x1) {
        for (int i = y0; i <= y1; i++) {
            ans.add(matrix[x0][i]);
        }
    } else if (y1 == y0) {
        for (int i = x0; i <= x1; i++) {
            ans.add(matrix[i][y0]);
        }
    } else {
        int curRow = x0;
        int curCol = y0;
        while (curCol != y1) {
            ans.add(matrix[x0][curCol]);
            curCol++;
        }
        while (curRow != x1) {
            ans.add(matrix[curRow][y1]);
            curRow++;
        }
        while (curCol != y0) {
            ans.add(matrix[x1][curCol]);
            curCol--;
        }
        while (curRow != x0) {
            ans.add(matrix[curRow][y0]);
            curRow--;
        }
    }
}
```

### 只用几个有限变量，旋转矩阵90°

```java
// n 行  n 列
public void rotate(int[][] matrix) {
    if (matrix == null || matrix.length == 0 || matrix[0].length == 0) {
        return;
    }
    int x0 = 0;
    int y0 = 0;
    int x1 = matrix.length - 1;
    int y1 = matrix[0].length - 1;
  
  	// 方阵，x0 != x1
    while (x0 < x1) {
        process(matrix, x0++, y0++, x1--, y1--);
    }
}

void process(int[][] matrix, int x0, int y0, int x1, int y1) {
  	// 分成 x1 - x0 组
    for (int i = 0; i < x1 - x0; i++) {
      	// 每组四个点分别为
      	// 第一行 [x0, y0 + i]
      	// 第二列 [x0 + i, y1]
      	// 第二行 [x1, y1 - i]
      	// 第一列 [x1 - i, y0]
        int tmp = matrix[x0][y0 + i];
        matrix[x0][y0 + i] = matrix[x1 - i][y0];
        matrix[x1 - i][y0] = matrix[x1][y1 - i];
        matrix[x1][y1 - i] = matrix[x0 + i][y1];
        matrix[x0 + i][y1] = tmp;
    }
}
```

### 最小操作步数

![image-20220321143256392](/Users/hq/Documents/md_image/algorithm_zcy/1_1_21.png)

```java

```



[只有两个键的键盘](https://leetcode-cn.com/problems/2-keys-keyboard/)



### 给定一个字符串类型的数组arr，求其中出现次数最多的前k个

Hash表+大根堆

Hash表统计每个字符串出现的次数，堆根据出现的次数排序

```java

```

### 最小栈

【思路一】两个栈

【思路二】存差值



### 如何仅用队列实现栈结构，如何仅用栈结构实现队列

【1】两个队列，一个队列进，当需要弹出时，把除最后一个其余的放到另一个队列，弹出最后一个，重复。

【2】两个栈，一个用于存入，一个用于删除。

问：现在让你实现图的深度优先遍历，但是只能用队列。

实现图的广度优先遍历，只能用栈。

### 动态规划空间压缩技巧

给你一个二维数组，其中每个数都是正数，要求从左上角走到右下角，每一步只能向右或者向下，沿途经过的数字要累加起来。请返回最小路径和。

```java
// 从i j 位置走,之后的最小的和
    int process(int i, int j, int[][] grid) {
        // 走完了
        if (i == grid.length && j == grid[0].length) {
            return 0;
        }
        int ans = Integer.MAX_VALUE;
        // 还能往下走
        if (i < grid.length - 1)
            ans = process(i + 1, j, grid);
        if (j < grid[0].length - 1)
            ans = Math.min(ans, process(i, j + 1, grid));

        if (ans == Integer.MAX_VALUE)
            ans = 0;

        return ans + grid[i][j];
    }

public int minPathSum(int[][] grid) {
        if (grid == null || grid.length == 0 || grid[0].length == 0) {
            return 0;
        }
        int m = grid.length;
        int n = grid[0].length;
        int[][] dp = new int[m + 1][n + 1];

        for (int j = n - 1; j >= 0; j--) {
            for (int i = m - 1; i >= 0; i--) {
                int ans = Integer.MAX_VALUE;
                // 还能往下走
                if (i < grid.length - 1)
                    ans = dp[i + 1][j];
                if (j < grid[0].length - 1)
                    ans = Math.min(ans, dp[i][j + 1]);
                if (ans == Integer.MAX_VALUE)
                    ans = 0;
                dp[i][j] = ans + grid[i][j];
            }
        }
        return dp[0][0];
//        return process(0, 0, grid);
}

```



###  最长公共子序列

```java
// 从i j 位置开始，之后还能匹配多少个
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



### 子序列的数目



### [接雨水](https://leetcode-cn.com/problems/trapping-rain-water/)

【思路一】单调栈

```java
class Solution {
     public int trap(int[] height) {
        // 最少要三个块才能接住雨水
        if (height.length < 3) {
            return 0;
        }
        int count = 0;
        LinkedList<Integer> stack = new LinkedList<>();
        int n = height.length;

        for (int i = 0; i < n; i++) {
            // 当前元素入栈之前,先判断是否需要弹出元素
            // 栈不为空且当前元素大于栈顶元素则需要弹出并处理
            while (!stack.isEmpty() && height[i] > height[stack.peekFirst()]) {
                int top = stack.pop();
                if (stack.isEmpty()) {
                    break;
                }
                int left = stack.peekFirst();
                int curWidth = i - left - 1;
                int currHeight = Math.min(height[left], height[i]) - height[top];
                count += currHeight * curWidth;
            }
            stack.push(i);
        }


        return count;

    }
}
```

【思路二】双指针

[i] = max{min{左侧Max,右侧Max} - arr[i], 0} 

```java
int trap(int[] height) {
        // 最少要三个块才能接住雨水
        if (height.length < 3) {
            return 0;
        }
        int count = 0;
        int left = 0;
        int right = height.length - 1;
        int leftMax = height[left++];
        int rightMax = height[right--];

        while (left <= right) {
            // 根据左侧max决定
            if (leftMax <= rightMax) {
                count += Math.max(leftMax - height[left], 0);
                leftMax = Math.max(leftMax, height[left++]);
            } else {
                count += Math.max(rightMax - height[right], 0);
                rightMax = Math.max(rightMax, height[right--]);
            }
        }

        return count;
}
```

### 给定一个数组arr长度为N，你可以把任意长度大于0且小于N的前缀作为左部分

![image-20220321201726271](/Users/hq/Documents/md_image/algorithm_zcy/1_1_29.png)

【思路1】预处理，两个数组，一个数组保存i位置切分时左侧部分的最大值，另一个保存i位置右侧的最大值。

 ```java
 ```

【思路2】

先找到全局Max

- 如果把全局Max切分到左侧，那么切分时，要让右侧Max最小
- 如果把全局Max切分到右侧，那么切分时，要让左侧Max最小

切分时要让一边的最大值尽量小，只有一种切分方式，那就是只包含边界。

2 | 4 1 5 3 9 6，其中|不管往右如何移动，最小的最大值就是2，右侧同理。

```java
 全局Max - Math.min(arr[0], arr[arr.len - 1]);
```

### 互为旋转词

![image-20220321203100951](/Users/hq/Documents/md_image/algorithm_zcy/1_1_30.png)

```java
public boolean rotateString(String s, String goal) {
        if (s == null) {
            return goal == null;
        }
        if (s.length() != goal.length()) {
            return false;
        }
        String str = s + s;
        
        return str.contains(goal);
}
```

### 咖啡杯问题

- 给一个正数数组arr，每个数代表当前位置的咖啡机冲一杯咖啡需要的时间 ，每台咖啡机一次只能冲一杯咖啡。

- 给定一个正数n，代表有n个人等待喝咖啡。每个人需要选择在某个咖啡机前排队。
- 给定一个正数a，代表一个洗杯子的机器，经过a的时间后洗好一个杯子
- 给定一个正数b，代表一个杯子不洗，经过b的时间后变干

问从所有人喝完咖啡到所有杯子被洗干净或变干，最少需要多少时间。

- 先得到最短的冲泡的时间 --》 使用小根堆，存放策略\<可用时间, 冲泡时间>，排序规则是可用时间+冲泡时间越小的排在上方，因为可用时间+冲泡时间就是下一次该咖啡机可用的时间，因此意味着越早能用的越先取。
- 根据上一步中所有人得到的时间，求解洗杯子+等杯子变干所需要的最短时间。

```java
static class Machine {
        int timeBegin;
        int consumeTime;

        public Machine(int timeBegin, int consumeTime) {
            this.timeBegin = timeBegin;
            this.consumeTime = consumeTime;
        }
    }

    static class MachineComparator implements Comparator<Machine> {

        @Override
        public int compare(Machine o1, Machine o2) {
            return o1.timeBegin + o1.consumeTime - o2.timeBegin - o2.consumeTime;
        }
    }

    static int minTime(int[] arr, int n, int a, int b) {
        PriorityQueue<Machine> queue = new PriorityQueue<>(new MachineComparator());
        for (int j : arr) {
            queue.add(new Machine(0, j));
        }
        int[] drinkTime = new int[n];

        for (int i = 0; i < n; i++) {
            Machine cur = queue.poll();
            assert cur != null;
            cur.timeBegin += cur.consumeTime;
            drinkTime[i] = cur.timeBegin;
            queue.add(cur);
        }
        return process(drinkTime, a, b, 0, 0);
    }

    // 用机器洗需要时间a, 自然干时间是b, avaTime 是洗碗机什么时候可用
    // 假设机器在avaTime可用, 要洗完[i ... n]的所有杯子,
    // 返回最早的结束时间
    static int process(int[] drinkTime, int a, int b, int i, int avaTime) {
        // 只剩一个咖啡杯 2种选择
        if (i == drinkTime.length - 1) {
            // 用咖啡机洗, 如果avaTime 大于 drinkTime[i], 则需要等到时间才能洗
            int machine = Math.max(avaTime, drinkTime[i]) + a;
            // 不用咖啡机洗
            int notMac = drinkTime[i] + b;
            return Math.min(machine, notMac);
        }

        // 使用机器洗的情况
        int useMachineTime = Math.max(avaTime, drinkTime[i]) + a;
        int finTime = process(drinkTime, a, b, i + 1, useMachineTime);
        finTime = Math.max(useMachineTime, finTime);

        // 不使用机器洗的情况
        int notMachineTime = drinkTime[i] + b;
        int finTime2 = process(drinkTime, a, b, i + 1, avaTime);
        finTime2 = Math.max(finTime2, notMachineTime);

        return Math.min(finTime, finTime2);
    }
```

### 调整数组arr，使得arr中任意两个相邻的数字相乘是4的倍数，如果能，返回true，否则false。

遍历数组，统计奇数个数a、只含有一个2因子的数个数b、包含4因子的数个数c。

- 如果b==0， 奇4奇4奇4...，
  - a = 1， c >=1
  - a = 2， c >= 1
  - a = 3， c >= 2
  - a = i，  c >= i - 1
- b != 0，2222224奇4奇...
  - 比上述情况多要一个4因子数
  - a = 1， c >= 1
  - a = 2， c >= 2
  - a = 3， c >= 3
  - a = i，  c >= i

```java
 
```

### [最小覆盖子串](https://leetcode-cn.com/problems/minimum-window-substring/)

【思路】滑动窗口

```java
public String minWindow(String s, String t) {
        char[] chars = s.toCharArray(), chart = t.toCharArray();
        int n = chars.length, m = chart.length;

        int[] hash = new int[128];
        for (char ch : chart) {
            hash[ch]--;
        }

        String ans = "";
        for (int right = 0, left = 0, cnt = 0; right < n; right++) {
            hash[chars[right]]++;
            if (hash[chars[right]] <= 0) {
                // 只统计有效字符
                cnt++;
            }

            // 已经满足条件了才移动左指针
            while (cnt == m && hash[chars[left]] > 0) {
                hash[chars[left++]]--;
            }
            // 满足情况下,只取最短的
            if ((cnt == m) && (ans.equals("") || ans.length() > right - left + 1)) {
                ans = s.substring(left, right + 1);
            }
        }
        return ans;
}
```



### 达标字符串

![image-20220322091046083](/Users/hq/Documents/md_image/algorithm_zcy/1_1_34.png)

打表-斐波那契数列。2 4 8

F(i)：前一个字符一定为1的情况下，有多少种填法

子问题

- 1xxxxx     = F(i - 1)
- 01xxxx     = F(i - 2)
- 001xxx     = F(i - 3)
- 00000x    = F(1) = 1

F(i) = F(i - 1) + F(i - 2)

【除初始项有严格递归的公式的，可以优化为O(logN)】

![image-20220322091611925](/Users/hq/Documents/md_image/algorithm_zcy/1_1_34_2.png)

根据递推式化简。

10^75快速计算。 =》 快速求矩阵的n次方  =》 将指数转为二进制，有1的位才乘到结果里。

【快速幂】



生牛问题：f(N)=f(N-1)+f(N-3)

### 小明最少去掉多少根木棍

![image-20220322094603010](/Users/hq/Documents/md_image/algorithm_zcy/1_1_35.png)

斐波那契数列。F(n)=F(n-1)+F(n-2)，故组不成三角形



### 牛牛准备参加学校组织的春游

![image-20220322100328887](/Users/hq/Documents/md_image/algorithm_zcy/1_1_36.png)

动态规划表，最后一行代表所有物品都处理的情况下，某个背包剩余容量下的解的方法数。这种情况下，每个物品的价值都是1。

暴力递归改的动态规划表是第0行，因为第0行代表处理完所有物品。



### 工作难度和报酬

![image-20220322102756976](/Users/hq/Documents/md_image/algorithm_zcy/1_1_37.png)

【贪心】选能做的工作里，报酬最高的。-》 先按难度排，再按报酬排，同样难度的，只保留报酬高的；难度高，报酬低的同样删掉。



### 字符串是否符合人们日常书写的形式

![image-20220322104616858](/Users/hq/Documents/md_image/algorithm_zcy/1_1_38.png)

- 开头如果有负号，只会出现在开头且仅出现一次，负号后面必须要有数字字符，且不能为0，之后全是数字

- 开头为0，后面不能接数字



### 实现Linux，tree命令

![image-20220322105620989](/Users/hq/Documents/md_image/algorithm_zcy/1_1_39.png)

【前缀树 + 深度优先遍历】

### 双向链表节点结构和二叉树节点结构是一样的，如果把prev认为是left，next认为是right的话。给定一个搜索二叉树的头节点head，请转换成一条有序的双向链表，并返回链表头节点。

【二叉树套路】左侧返回自己的头和尾，右侧返回自己的头和尾。把自己的prev指向左侧尾，把自己的next指向右侧的头。。。

[二叉搜索树与双向链表](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-yu-shuang-xiang-lian-biao-lcof/)

```java
public Node treeToDoublyList(Node root) {
    if (root == null) {
        return null;
    }

    Info info = process(root);
    info.head.left = info.tail;
    info.tail.right = info.head;
    return info.head;
}

Info process(Node root) {
    if (root == null) {
        return new Info(null, null);
    }
    Info lInfo = process(root.left);
    Info rInfo = process(root.right);
    if (lInfo.tail != null) {
        lInfo.tail.right = root;
    }
    if (rInfo.head != null) {
        rInfo.head.left = root;
    }
    root.left = lInfo.tail;
    root.right = rInfo.head;

    return new Info(lInfo.head != null ? lInfo.head : root,
            rInfo.tail != null ? rInfo.tail : root);
}

static class Info {
    Node head;
    Node tail;

    public Info(Node head, Node tail) {
        this.head = head;
        this.tail = tail;
    }
}
```

### 找到一棵二叉树中，最大的二叉搜索子树，返回最大搜索二叉子树的节点个数/返回子树的根节点

最大二叉搜索子树：节点数最多的那棵子树

【二叉树递归套路】

给定某个节点X

- 最大二叉搜索子树与节点X有关
- 最大二叉搜索子树与节点X无关
  - 左树右树

子树：最大二叉子树的头 ，最大值最小值，是否是BST，二叉搜索子树的大小

```java
public static Info process(Node root) {
        if (root == null) {
            return null;
        }

        Info left = process(root.left);
        Info right = process(root.right);

        int min = root.val;
        int max = root.val;

        if (left != null) {
            min = Math.min(left.min, min);
            max = Math.max(left.max, max);
        }
        if (right != null) {
            min = Math.min(right.min, min);
            max = Math.max(right.max, max);
        }

        int maxBSTSize = 0;
        Node maxBSTRoot = null;
        if (left != null) {
            maxBSTSize = left.maxBSTSize;
            maxBSTRoot = left.maxBSTRoot;
        }
        if (right != null && right.maxBSTSize > maxBSTSize) {
            maxBSTSize = right.maxBSTSize;
            ;
            maxBSTRoot = right.maxBSTRoot;
        }

        boolean isBST = false;
        if ((left == null || left.isBST) && (right == null || right.isBST)) {
            if ((left == null || left.max < root.val) &&
                    (right == null || right.min > root.val)) {

                isBST = true;
                maxBSTSize = 1 + (left != null ? left.maxBSTSize : 0) +
                        (right != null ? right.maxBSTSize : 0);

                maxBSTRoot = root;
            }
        }
        return new Info(isBST, maxBSTRoot, min, max, maxBSTSize);

    }

    static class Info {
        public boolean isBST;
        public Node maxBSTRoot;
        public int min;
        public int max;
        public int maxBSTSize;

        public Info(boolean isBST, Node maxBSTRoot, int min, int max, int maxBSTSize) {
            this.isBST = isBST;
            this.maxBSTRoot = maxBSTRoot;
            this.min = min;
            this.max = max;
            this.maxBSTSize = maxBSTSize;
        }
    }
```

### 招聘信息质量问题

![image-20220322161249990](/Users/hq/Documents/md_image/algorithm_zcy/1_1_42.png)

[最大子数组和](https://leetcode-cn.com/problems/maximum-subarray/)

```java
public int maxSubArray(int[] nums) {
        if (nums == null || nums.length == 0) {
            return 0;
        }

        int cur = 0;
        int max = Integer.MIN_VALUE;

        for (int i : nums) {
            cur += i;
            max = Math.max(cur, max);
            if (cur < 0) {
                cur = 0;
            }
        }

        return max;
}

public int maxSubArrayDp(int[] nums) {
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
```

先假设找到了答案，然后分析。

\|  pre   |     target     |      next    \|

- target从左侧开始，加到中间任意部分肯定大于0，否则就不需要这一部分了
- pre这一块加一起肯定小于0，否则就需要把这一块也加上才能得到最大值。

### 给定一个整形矩阵，返回子矩阵的最大累加和

0：-1  2  3  4                0 ： -1  2  3  4

1： 3  3  6  8      	 0~1  ： 2   5  9  12        求累加和

2： 2  1  2  3           0~2  :    4   6  11  11  



### 路灯问题

![image-20220322170748001](/Users/hq/Documents/md_image/algorithm_zcy/1_1_44.png)

【动态规划】

【贪心】

对于第i个位置

- 如果是x，不用给灯
- 如果是. ，判断i + 1位置
  - 如果i+1是x，则i位置必须给一栈灯，后跳两步
  - 如果i+1是. ，那么在i + 1位置放灯，下面后跳三步

### 已知二叉树中没有重复节点，给定了先序遍历数组和中序遍历数组，请返回后序遍历数组

### 二叉树最近公共祖先

【递归思路】

对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。

有以下几种条件：

- 临界条件1：如果当前节点为空，返回空
- 临界条件2：当前节点为p，直接返回p，不需要向下查了
- 临界条件3：当前节点为q，直接返回q，不需要继续递归了
- 根据以上临界条件，问题变为了以root为根的树上是否有p节点和q节点的问题，如果有就返回p节点或者q节点，否则返回null。

[二叉树最近公共祖先](https://zhuanlan.zhihu.com/p/269251274)

【思路2】

用一个HashMap，遍历时保存每个节点的父节点是谁。

弄完以后，用两个HashSet，一个保存p节点的所有祖先节点，另一个保存q节点所有的祖先节点，第二个添加时需要判断是否当前节点已经存在在p节点对应的set里了。

```java
```



### 二叉树两个节点之间的最大路径

- 当前节点参与最大路径，maxDistance= 左树高度 + 右树高度
- 当前节点不参与最大路径，maxDistance=左树的最大distance+右树的最大distance

```c
public Info maxDistance(TreeNode root) {
        if (root == null) {
            return new Info(0, 0);
        }
        Info leftInfo = maxDistance(root.left);
        Info rightInfo = maxDistance(root.right);

        int curDistance = leftInfo.height + rightInfo.height + 1;
        int maxDistance = Math.max(curDistance, Math.max(leftInfo.maxDistance,
                rightInfo.maxDistance));
        int height = Math.max(leftInfo.height, rightInfo.height) + 1;

        return new Info(maxDistance, height);

    }

    static class Info {
        int maxDistance;
        int height;

        public Info(int maxDistance, int height) {
            this.maxDistance = maxDistance;
            this.height = height;
        }
    }

//改进的版本
int HeightOfBinaryTree(BinaryTreeNode* pNode, int &nMaxDistance){
	if (pNode == NULL)
		return -1;   //空节点的高度为-1
	//递归
	int nHeightOfLeftTree = HeightOfBinaryTree(pNode->m_pLeft, nMaxDistance) + 1;   //左子树的的高度加1
	int nHeightOfRightTree = HeightOfBinaryTree(pNode->m_pRight, nMaxDistance) + 1;   //右子树的高度加1
	int nDistance = nHeightOfLeftTree + nHeightOfRightTree;    //距离等于左子树的高度加上右子树的高度+2
	nMaxDistance = nMaxDistance > nDistance ? nMaxDistance : nDistance;            //得到距离的最大值
	return nHeightOfLeftTree > nHeightOfRightTree ? nHeightOfLeftTree : nHeightOfRightTree;
}
```

### 带指向父节点指针的二叉树找后继节点

找结点x的后继节点

- 如果x有右树，则后继是x右树上的最左节点
- 如果x没有右树，则向上找，如果当前节点是上一个节点的右树，那么继续找；如果当前节点是上一个节点的左树，那么上一个节点就是x的后继



### 二叉树的序列化与反序列化

序列化：按照某种遍历顺序，打印行为改为拼接字符串的行为，注意在每个结束位置添加分隔符

反序列化：先将序列化后的二叉树按分隔符分割成代表每个节点的字符串，再按照原来的遍历方式恢复原树。



### 找到数组中，比左边所有数字都大，比右边所有数字都小的数字，复杂度O(n)

当我们遍历到某个值i

- 该值需要比左侧最大的还要大，比右侧最小的还小

【思路1】

- 预处理一下，先求出每个位置左侧的最大值是多少或者某个位置右侧的最小值是多少

【思路2】

- 单调栈

### 比较版本号

两个版本号：

”2.1.1“和”1.1.1“比较它俩的大小。



### [买卖股票的最佳时机](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock/)

```java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices == null || prices.length == 0) {
            return 0;
        }
        int[] maxRight = new int[prices.length];
        int curMax = prices[prices.length - 1];
        maxRight[prices.length - 1] = prices[prices.length - 1];
        // 记录每个数右侧的最大值
        for (int i = prices.length - 2; i >= 0; --i) {
            maxRight[i] = curMax;
            if (prices[i] > curMax) {
                curMax = prices[i];
            }
        }
        int max = Integer.MIN_VALUE;
        for (int i = 0; i < prices.length; ++i) {
            if (maxRight[i] - prices[i] > max) {
                max = maxRight[i] - prices[i];
            }
        }
        return max;
    }
}

class Solution {
    public int maxProfit(int[] prices) {
        if (prices == null || prices.length == 0) {
            return 0;
        }
        int ans = 0;
        int minPrice = Integer.MAX_VALUE;

        for (int i = 0; i < prices.length; ++i) {
            if (prices[i] < minPrice) {
                minPrice = prices[i];
            } else if (prices[i] - minPrice > ans) {
                ans = prices[i] - minPrice;
            }
        }
        return ans;
    }
}
```

### [买卖股票的最佳时机 II](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-ii/)

```java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices == null || prices.length == 0) {
            return 0;
        }

        int ans = 0;
        for (int i = 1; i < prices.length; ++i) {
            if (prices[i] - prices[i - 1] > 0) {
                ans += prices[i] - prices[i - 1];
            }
        }
        return ans;
    }
}
```

### [掷骰子的N种方法](https://leetcode-cn.com/problems/number-of-dice-rolls-with-target-sum/)

```java
static final int MOD = 1_000_000_007;

    public int numRollsToTarget(int n, int k, int target) {

        int[][] dp = new int[n + 1][target + 1];
        dp[n][0] = 1;

        for (int i = n - 1; i >= 0; i--) {
            for (int t = 1; t <= target; t++) {
                int ans = 0;
                for (int j = 1; j <= k; j++) {
                    if (t - j >= 0) {
                        ans += dp[i + 1][t - j];
                        ans %= MOD;
                    }
                }
                dp[i][t] = ans;
            }
        }

        return dp[0][target];
}


    int process(int i, int n, int k, int target) {
        if (i == n) {
            return target == 0 ? 1 : 0;
        }
        // 第 i 个 骰子有 1 ~ k 种可能
        int ans = 0;
        for (int j = 1; j <= k; j++) {
            ans += process(i + 1, n, k, target - j);
            ans %= MOD;
        }
        return ans;
}

		static final int MOD = 1_000_000_007;
		// 空间压缩
    public int numRollsToTarget(int n, int k, int target) {
        if (target == 0 || n == 0 || k == 0 || n * k < target) {
            return 0;
        }    

        int[][] dp = new int[2][target + 1];

        if ((n - 1 & 1) == 0) {
            dp[1][0] = 1;
        } else {
            dp[0][0] = 1;
        }

        for (int i = n - 1; i >= 0; i--) {
            int row = i & 1; // 偶数用0行
            int last = ~i & 1;

            for (int t = 1; t <= target; t++) {

                int ans = 0;
                for (int j = 1; j <= k; j++) {
                    if (t - j >= 0) {
                        ans += dp[last][t - j];
                        ans %= MOD;
                    }
                }
                dp[row][t] = ans;
            }
          	// last的用不到了，置0，否则会出错
            Arrays.fill(dp[last], 0);
        }
        return dp[0][target];
    }
```

### [二叉树的层序遍历](https://leetcode-cn.com/problems/binary-tree-level-order-traversal/)

【思路】当我们访问root时，队列的宽度就是该层的节点个数。同理，当我们访问左儿子时（出队之前），队列的长度应该是当前层节点的个数，因此我们只需要在for循环里处理完这一层再进行下一步操作就可以了。

```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        if (root == null) {
            return new ArrayList<>();
        }
        Queue<TreeNode> queue = new LinkedList<>();
        List<List<Integer>> ans = new LinkedList<>();
        queue.add(root);
        while (!queue.isEmpty()) {
            int curLevelSize = queue.size();
            List<Integer> curList = new ArrayList<>(curLevelSize);
            for (int i = 0; i < curLevelSize; i++) {
                TreeNode cur = queue.poll();
                curList.add(cur.val);
                if (cur.left != null) {
                    queue.add(cur.left);
                }
                if (cur.right != null) {
                    queue.add(cur.right);
                }
            }
            ans.add(curList);
        }
       return ans;
    } 

}
```

### 求比数n大的最小回文数的字符串形式

题意要求，数N可达10000位，只能用字符串数组存储，对于回文数，只要求前半部分就可写出后半部分，大致分两种情况，一种是前半部分小于后半部分，前半部分有要+1的，这时候就要判断是否有9，有9存在要增加一位的情况，如999/199/191。另一种简单，因为前半部分大于后半部分，所以，只要把前半部分替换后半部分即可，如321/291.



### [验证二叉树的前序序列化](https://leetcode-cn.com/problems/verify-preorder-serialization-of-a-binary-tree/)

【思路1】边的数量

- 除了头结点外，每个节点都需要消耗一个边
- 头结点和非空节点可以生成两个边
- 貌似只有前序才能这样做

```java
public boolean isValidSerialization(String preorder) {
        String[] preS = preorder.split(",");
  			// 头结点不需要消耗边，而我们下面的流程中，头结点是消耗了一个边的，故初值设为1
        int edges = 1;
        for (String cur : preS) {
            edges -= 1;
            if (edges < 0) {
                return false;
            }
            if (!"#".equals(cur)) {
                edges += 2;
            }
        }
        return edges == 0;
}
```

【思路2】用栈模拟前序遍历



【思路3】利用反序列化的思想

```java
    public boolean isValidSerializationStack(String preorder) {
        String[] preS = preorder.split(",");
        Queue<String> queue = new LinkedList<>();
        Collections.addAll(queue, preS);
        return checkValid(queue) && queue.isEmpty();
    }

    public boolean checkValid(Queue<String> queue) {
      	// base case, 所有节点都遍历了，还没有检查成功，返回false
        if (queue.isEmpty()) {
            return false;
        }
      	// 先序遍历
        String cur = queue.poll();
      	// 检查到NIL节点没问题的话，至此没问题
        if ("#".equals(cur)) {
            return true;
        }
      	// 递归的检查
        return checkValid(queue) && checkValid(queue);
    }
```

### [二叉树展开为链表](https://leetcode-cn.com/problems/flatten-binary-tree-to-linked-list/)

```java
public void flatten(TreeNode root) {
        process(root);
    }

    private Info process(TreeNode root) {
        if (root == null) {
            return new Info(null, null);
        }

        Info lcInfo = process(root.left);
        Info rcInfo = process(root.right);

        // 转换
        if (lcInfo.head != null) {
            root.right = lcInfo.head;
        }

        if (lcInfo.tail != null) {
            lcInfo.tail.right = rcInfo.head;
        }
        root.left = null;

        TreeNode curTail = rcInfo.tail != null ? rcInfo.tail : lcInfo.tail != null ?
                lcInfo.tail : root;
        return new Info(root, curTail);

    }

    private static class Info {
        TreeNode head;
        TreeNode tail;

        public Info(TreeNode head, TreeNode tail) {
            this.head = head;
            this.tail = tail;
        }
    }
```

### 查找第K大的数

【思路一】利用小根堆

```java
public int findK(int[] arr, int k) {
  	if (arr == null) {
      	return -1;
    }
  
  	PriorityQueue<Integer> queue = new PriorityQueue<>();
  	for (int n : arr) {
      	queue.add(n);
      	if (queue.size() > k) {
          	queue.poll();
        }
    }
  	return queue.poll();
}
```

【思路二】快排-》快速查找

```java

public int quickFindK(int[] arr, int k) {
  	int targetIdx = k - 1, left = 0, right = arr.length - 1;
    int pivotIdx = partition(arr, left, right);
  	while (pivotIdx != targetIdx) {
      	if (pivotIdx > targetIdx) {
          	right = pivotIdx - 1;
        } else {
          	left = pivotIdx + 1;
        }
      	pivotIdx = partition(arr, left, right);
    }
  	return arr[pivotIdx];

}


private int partition(int[] arr, int left, int right) {
  	int pivot = arr[left];
  	while(left < right) {
      	// 先从右侧找需要移动到左边的
      	while (left < right && arr[right] >= pivot) {
          	right--;
        }
      	// 找到了，移过去
      	arr[left] = arr[right];
        // 同理
      	while (left < right && arr[left] <= pivot) {
          	left++;
        }
      	arr[right] = arr[left];
    }
  	// 将基准放到合适位置
  	arr[left] = pivot;
  	return left;
}
```

### [岛屿的最大面积](https://leetcode-cn.com/problems/max-area-of-island/)

【思路】深度优先遍历，遍历时记录岛的数量

```java
public class T695 {
    public int maxAreaOfIsland(int[][] grid) {
        if (grid == null || grid.length == 0 || grid[0].length == 0) {
            return 0;
        }
        int ans = 0;
        for (int i = 0; i < grid.length; i++) {
            for (int j = 0; j < grid[0].length; j++) {
                if (grid[i][j] == 1) {
                    ans = Math.max(ans, process(i, j, grid));
                }
            }
        }
        return ans;
    }

    // 返回数量
    public int process(int x, int y, int[][] grid) {
        // 越界情况下,数量为0
        if (x < 0 || y < 0 || x == grid.length || y == grid[0].length || grid[x][y] == 0) {
            return 0;
        }

        // 标记一下访问过了
        grid[x][y] = 0;
        int num = 1;
        num += process(x + 1, y, grid);
        num += process(x, y + 1, grid);
        num += process(x - 1, y, grid);
        num += process(x, y - 1, grid);

        return num;
    }
}
```

### [礼物的最大价值](https://leetcode-cn.com/problems/li-wu-de-zui-da-jie-zhi-lcof/)

【思路1】暴力递归

对于当前位置的礼物，之后有两种选法，一个是选右边的，另一个是选下方的，最大的礼物价值肯定是选下面两种选法中价值最大的。

```java
class Solution {
    public int maxValue(int[][] grid) {
        return process(0, 0, grid);
    }

    public int process(int x, int y, int[][] grid) {
        if (x == grid.length || y == grid[0].length) {
            return 0;
        }
        if (x == grid.length - 1 && y == grid[0].length - 1) {
            return grid[x][y];
        }

        int right = process(x, y + 1, grid);
        int down = process(x + 1, y, grid);

        return Math.max(right, down) + grid[x][y];
    }
}

// 动态规划
public int maxValue(int[][] grid) {
        if (grid == null || grid.length == 0 || grid[0] == null || grid[0].length == 0) {
            return 0;
        }
        int xLen = grid.length;
        int yLen = grid[0].length;

        int[][] dp = new int[xLen + 1][yLen + 1];
        dp[xLen - 1][yLen - 1] = grid[xLen - 1][yLen - 1];

        for (int y = grid[0].length - 1; y >= 0; y--) {
            for (int x = grid.length - 1; x >= 0; x--) {
                if (x == grid.length - 1 && y == grid[0].length - 1) {
                    dp[x][y] = grid[x][y];
                }
                int right = dp[x][y + 1];
                int down = dp[x + 1][y];

                dp[x][y] = Math.max(right, down) + grid[x][y];
//                return Math.max(right, down) + grid[x][y];
            }
        }

        return dp[0][0];
    }
```



### [用 Rand7() 实现 Rand10()](https://leetcode-cn.com/problems/implement-rand10-using-rand7/)

【拒绝采样】

rand7() + rand7()不是等概率的生成。范围是[2, 14]，其中2只有1+1这种情况是可以得到的，而14却有很多情况可以得到，比如1+13，2+12，3+11...，因此生成的数据并不是等概率的，而原题要求等概率。

那么有没有其它方法生成等概率的数字呢？

假设我们有rand5()，那么我们可以很容易的得到等概率的[10, 20, 30, 40, 50]，如果我们现在有能生成0~9的函数fun()，我们可以很容易的生成[10, 11, 12, ..., 59]这样的等概率数据。

类似的我们有rand7()，怎么只用rand7()生成类似上面的连续的数据呢？我们观察上面的序列，大的序列之间的“空槽位”刚好等于fun()生成的数据的数量。因为我们现在rand7()只能生成[1~7]的数据，因此我们这样生成`(rand7() - 1) * 7 + rand7()`。这样的话，     `(rand7() - 1)* 7`能够生成[0, 7, 14, 21, 28, 35, 42]，而`rand7()`可以生成[1~7]。这样相加生成的数据就是[1~7, 8 ~14, 15 ~ 28, ...]，因此每个数据是等概率的。我们只采样其中的40个，后面多余的9个丢掉。

```java
public int rand10() {
    int ans = 41;
    while (ans > 40) {
        ans = (rand7() - 1) * 7 + rand7();
    }
    return ans % 10 + 1;
}
```

更一般的，(randX() - 1) * Y + randY()，可以等概率的生成[1~XY]区间的数。



### [无重复字符的最长子串](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/)

【思路一】滑动窗口

窗口向右扩张，直到遇到了重复的字符，就开始移动窗口左侧。使用HashSet保存窗口里的元素。

```java
public int lengthOfLongestSubstring(String s) {
        if (s == null || s.length() == 0) {
            return 0;
        }
        if (s.length() == 1) {
            return 1;
        }

        char[] str = s.toCharArray();
        HashSet<Character> set = new HashSet<>();
        int left = 0;
        int right = 1;
        set.add(str[left]);
        int curLen = 1;
        int maxLen = Integer.MIN_VALUE;
        while (left <= right && right < str.length) {
            if (!set.contains(str[right])) { // r右移
                set.add(str[right++]);
                curLen++;
            } else { 	// 否则left右移
                set.remove(str[left++]);
                curLen--;
            }
            maxLen = Math.max(maxLen, curLen);
        }

        return maxLen;
}
```



### [LRU 缓存](https://leetcode-cn.com/problems/lru-cache/)

```java
package algorithm.leetcode;

import java.util.HashMap;

public class LRUCache<K, V> {

    private final int capacity;

    private int size;


    private final DNode<K, V> fakeHade;
    private final DNode<K, V> fakeTail;

    private final HashMap<K, DNode<K, V>> map;

    public LRUCache(int capacity) {
        this.capacity = capacity;

        fakeHade = new DNode<>();
        fakeTail = new DNode<>();

        map = new HashMap<>((int) (capacity / .75F));
    }

    public V get(K key) {
        DNode<K, V> node = map.get(key);
        if (node != null) {
            if (fakeHade.next != node) {
                removeNode(node);
                addToHead(node);
            }
            return node.val;
        }
        return null;
    }

    public void put(K key, V value) {
        DNode<K, V> node = map.get(key);
        if (node != null) {
            node.val = value;
            removeNode(node);
            addToHead(node);
        } else {
            node = new DNode<>(key, value);
            map.put(key, node);
            size++;
            if (size > capacity) {
                node = removeNode(fakeTail.prev);
                map.remove(node.key);
            }
        }
    }

    private void addToHead(DNode<K, V> node) {
        node.prev = fakeHade;
        node.next = fakeHade.next;

        fakeHade.next = node;
        node.next.prev = node;
    }

    private DNode<K, V> removeNode(DNode<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;

        return node;
    }


    private static class DNode<K, V> {
        K key;
        V val;

        DNode<K, V> prev;
        DNode<K, V> next;

        public DNode() {
        }

        public DNode(K key, V val) {
            this.key = key;
            this.val = val;
        }
    }
}

```



### [K 个一组翻转链表](https://leetcode-cn.com/problems/reverse-nodes-in-k-group/)



### [剑指 Offer 28. 对称的二叉树](https://leetcode-cn.com/problems/dui-cheng-de-er-cha-shu-lcof/)

- 对于遍历到的一个节点，需要左树和右树都是对称的
- 如果该节点左树右树都为空，那么它是对称的
- 如果该节点左树右树只有一个为空，或者左儿子右儿子都不为空但是值不一样，那么它是非对称的。
- 接下来我们只需要递归的判断，左树的左儿子是否和右树的右儿子对称，且左树的右儿子要和右树的左儿子对称

```java
class Solution {
    public boolean isSymmetric(TreeNode root) {
        return root == null ? true : process(root.left, root.right);
    }

    public boolean process(TreeNode left, TreeNode right) {
        if (left == null && right == null) {
            return true;
        }
        if (left == null || right == null || left.val != right.val) {
            return false;
        }

        return process(left.left, right.right) && process(left.right, right.left);
    }
}
```

### [剑指 Offer 27. 二叉树的镜像](https://leetcode-cn.com/problems/er-cha-shu-de-jing-xiang-lcof/)

直接交换每个节点的左右儿子即可。

```java
class Solution {
    public TreeNode mirrorTree(TreeNode root) {
        process(root);
        return root;
    }

    private void process(TreeNode root) {
        if (root == null) {
            return;
        }
        TreeNode tmp = root.left;
        root.left = root.right;
        root.right = tmp;

        process(root.left);
        process(root.right);
    }
}
```

### 用Java实现BitMap

```java
public class BitMap {
    private final byte[] bits;

    public BitMap(int size) {
        bits = new byte[getIndex(size) + 1];
    }

    private int getIndex(int i) {
        return i >> 3; // aka i / 8
    }

    private int getBitIndex(int i) {
        return (i & (8 - 1));  // aka i % 8
    }


    public void set1(int idx) {
        bits[getIndex(idx)] |= (1 << getBitIndex(idx));
    }

    public void set0(int idx) {
        bits[getIndex(idx)] &= ~(1 << getBitIndex(idx));
    }

    public int getVal(int idx) {
        return (bits[getIndex(idx)] & (1 << getBitIndex(idx))) == 0 ? 0 : 1;
    }


    public static void main(String[] args) {
        BitMap bitMap = new BitMap(10000);
        bitMap.set1(1000);
        System.out.println(bitMap.getVal(1000));
    }
}
```

### [下一个排列](https://leetcode-cn.com/problems/next-permutation/)

【思路】我们需要将左边的较小数与右边的较大数交换，并且这个较小数尽量靠右而较大数尽量小。交换完成后较大数右边的序列需要重新排序为升序，使变大的幅度尽可能小。

- 先从右往左遍历，找到第一个逆序的数，这个数就是我们要找的较小数。
- 再从右往左遍历，找到第一个大于上面较小数的数，这个数就是我们要找的较大数。
- 交换他们，然后从较大数的下一个元素开始逆序。

```java
public void nextPermutation(int[] nums) {
        // 左侧较小值的下标
        int leftMinIdx = nums.length - 2;
        while (leftMinIdx >= 0 && nums[leftMinIdx] >= nums[leftMinIdx + 1]) {
            leftMinIdx--;
        }
        if (leftMinIdx >= 0) {
            // 右侧第一个比较小值大的下标
            int rightMaxIdx = nums.length - 1;
            while (rightMaxIdx >= 0 && nums[leftMinIdx] >= nums[rightMaxIdx]) {
                rightMaxIdx--;
            }
            swap(nums, leftMinIdx, rightMaxIdx);
        }
        reverse(nums, leftMinIdx + 1);
    }

    private void swap(int[] arr, int i, int j) {
        int tmp = arr[i];
        arr[i] = arr[j];
        arr[j] = tmp;
    }

    private void reverse(int[] arr, int start) {
        int end = arr.length - 1;
        while (start < end) {
            swap(arr, start, end);
            start++;
            end--;
        }
    }
```



### [最长有效括号](https://leetcode-cn.com/problems/longest-valid-parentheses/)

【思路】动态规划

- dp[i]表示以i结尾处能最长匹配的括号个数
- 如果i处是“(”的话，只能匹配0个
- 如果i出是“)”的话，需要看他前一个的情况
  - 如果前一个是“(”的话，当前最少能匹配的长度是2
  - 如果前一个是")"的话，需要看前一个“)”匹配的最左的"("括号的前一个括号
    - 如果前一个是"("，则匹配的长度最少是dp[i - 1] + 2
    - 如果前一个是”)“，那么不能匹配，当前位置只能匹配0个
  - 此外还要判断前面的即i - dp[i - 1] - 1处匹配的情况。
- 因此dp[i] += dp[i - 1] + 2 + (pre > 0 ? dp[pre - 1] : 0)

```java
class Solution {
    public int longestValidParentheses(String s) {
        char[] str = s.toCharArray();
        int[] dp = new int[str.length];
        int curMax = 0;

        int pre = 0;
        for (int i = 1; i < str.length; i++) {
            if (str[i] == ')') {
                pre = i - dp[i - 1] - 1;
                if (pre >= 0 && str[pre] == '(') {
                    dp[i] += dp[i - 1] + 2 + (pre > 0 ? dp[pre - 1] : 0);
                }
            }
            curMax = Math.max(curMax, dp[i]);
        }

        return curMax;
    }
}
```

### [反转链表 II](https://leetcode-cn.com/problems/reverse-linked-list-ii/)

```java
public ListNode reverseBetween(ListNode head, int left, int right) {
        if (head == null || head.next == null) {
            return head;
        }
        ListNode fakeHead = new ListNode(-1);
        fakeHead.next = head;
        ListNode pre = fakeHead;
        ListNode cur;
        // 先找到起始位置
        for (int i = 0; pre.next != null && i < left - 1; ++i) {
            pre = pre.next;
        }
        cur = pre.next;
        ListNode next;
        // 头插法反转链表
        for (int i = 0; cur != null && i < right - left; ++i) {
            // 将cur.next从链表删除
            next = cur.next;
            cur.next = next.next;
          
          	// 头插法重新插入
            next.next = pre.next;
            pre.next = next;
        }
        return fakeHead.next;
    }
```

### [反转链表](https://leetcode-cn.com/problems/reverse-linked-list/)

```java
public ListNode reverseList(ListNode head) {
        if (head == null || head.next == null) {
            return head;
        }
        ListNode fakeHead = new ListNode();
        fakeHead.next = head;
        ListNode cur = head.next;
        head.next = null;
        ListNode next;
        while (cur != null) {
            next = cur.next;
            cur.next = fakeHead.next;
            fakeHead.next = cur;
            cur = next;
        }
        
        return fakeHead.next;
    }
```

### [全排列](https://leetcode-cn.com/problems/permutations/)

```java
class Solution {
    public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        process(nums, 0, res);
        return res;
    }

    public void process(int[] nums, int i, List<List<Integer>> res) {
        if (i == nums.length) {
            Integer[] arrWarped =
                Arrays.stream(nums).boxed().toArray(Integer[]::new);
            
            res.add(new ArrayList<>(Arrays.asList(arrWarped)));
        }

        for (int j = i; j < nums.length; ++j) {
            swap(nums, i, j);
            process(nums, i + 1, res);
            swap(nums, i, j);
        }

    }

    void swap(int[] arr, int i, int j) {
        int tmp = arr[i];
        arr[i] = arr[j];
        arr[j] = tmp;
    }
}
```

### [K 个一组翻转链表](https://leetcode-cn.com/problems/reverse-nodes-in-k-group/)

```java
class Solution {
    public ListNode reverseKGroup(ListNode head, int k) {
        if(head == null || head.next == null || k <= 0) {
            return head;
        }
        ListNode dummyHead = new ListNode();
        dummyHead.next = head;
        ListNode pre = dummyHead;
        ListNode tail = dummyHead;
        while (true) {
            int count = 0;
            while (tail != null && count != k) {
                count++;
                tail = tail.next;
            }
            if (tail == null) {
                break;
            }
            ListNode curHead = pre.next;
          	// 尾插法
            while (pre.next != tail) {
                ListNode cur = pre.next;
                pre.next = cur.next;
                cur.next = tail.next;
                tail.next = cur;
            }
            pre = curHead;
            tail = curHead;
        }
        return dummyHead.next;
    }
}
```

### [反转链表](https://leetcode-cn.com/problems/fan-zhuan-lian-biao-lcof/)

```JAVA
public static ListNode reverse(ListNode head) {
        ListNode dummyHead = new ListNode();
        dummyHead.next = head;
        ListNode cur = head.next;
        head.next = null;
        ListNode next;

       while (cur != null) {
           next = cur.next;
           cur.next = dummyHead.next;
           dummyHead.next = cur;
           cur = next;
       }

        return dummyHead.next;
    }
```

### [链表排序](https://leetcode-cn.com/problems/7WHec2/)

```java
class Solution {
    public ListNode sortList(ListNode head) {
        if (head == null) {
            return null;
        }
        PriorityQueue<ListNode> minHeap = new PriorityQueue<>(new ListNodeComp());
        while (head != null) {
            minHeap.add(head);
            head = head.next;
        }

        ListNode dummyHead = new ListNode();
        ListNode pre = dummyHead;
        ListNode cur = null;
        while (!minHeap.isEmpty()) {
            cur = minHeap.poll();
            pre.next = cur;
            pre = cur;
        }
        cur.next = null;
        return dummyHead.next;
    }

    private static class ListNodeComp implements Comparator<ListNode> {

        @Override
        public int compare(ListNode o1, ListNode o2) {
            return o1.val - o2.val;
        }
    }
}
```

### [位1的个数](https://leetcode-cn.com/problems/number-of-1-bits/)

假设一个数n的二进制为`b1000`，那么n-1的二进制为`b0111`，那么使用n & (n-1)就可以把原来n最后一个1给消除，重复这个过程就可以知道一个数的bit位了。	

【kng】

- n & (n - 1)可以去掉n最后一个为1的bit位。
- 当n为2的n次方时，i & (n - 1)等价于取模。

```java
public class Solution {
    // you need to treat n as an unsigned value
    public int hammingWeight(int n) {
        int count = 0;
        while (n != 0) {
            count++;
            n &= (n - 1);
        }
        return count;
    }
}
```

### [二进制表示中质数个计算置位](https://leetcode-cn.com/problems/prime-number-of-set-bits-in-binary-representation/)

```java
class Solution {
    public int countPrimeSetBits(int left, int right) {
        if (right < left) {
            return 0;
        }
        int count = 0;
        for (int i = left; i <= right; ++i) {
            if (isPrime(Integer.bitCount(i))) {
                count++;
            }
        }
        return count;
    }

    public boolean isPrime(int n) {
        if (n < 2) {
            return false;
        }
        int c = (int) Math.sqrt(n);
        for (int i = 2; i <= c; ++i) {
            if (n % i == 0) {
                return false;
            }
        }
        return true;
    }
}
```

### [环形链表](https://leetcode-cn.com/problems/linked-list-cycle/)

【思路一】快慢指针。

- 慢指针每次走一步，快指针每次都两步。
  - 两个指针走的时候如果后面有null了，说明链表没有环
  - 否则直到两个指针相遇，否则继续走
- 当两个指针相遇后，将快指针放回到链表头部，然后快慢指针每次都只走一步
- 直到两个指针再次相遇，再次相遇的节点即为入环节点。

```java
public class Solution {
    public boolean hasCycle(ListNode head) {
        if (head == null || head.next == null || head.next.next == null) {
            return false;
        }
        ListNode pFast = head.next.next;
        ListNode pSlow = head.next;
        while (pFast != pSlow) {
            if (pFast.next == null || pFast.next.next == null || pSlow.next == null) {
                return false;
            }
            pFast = pFast.next.next;
            pSlow = pSlow.next;
        }
        pFast = head;
        return true;
    }
}
```

### [环形链表 II](https://leetcode-cn.com/problems/linked-list-cycle-ii/)

```java
		public ListNode detectCycle(ListNode head) {
        if (head == null || head.next == null || head.next.next == null) {
            return null;
        }
        ListNode pFast = head.next.next;
        ListNode pSlow = head.next;
        while (pFast != pSlow) {
            if (pFast == null || pFast.next == null) {
                return null;
            }
            pFast = pFast.next.next;
            pSlow = pSlow.next;
        }
        // 至此一定有环
        pFast = head;
        while (pFast != pSlow) {
            pFast = pFast.next;
            pSlow = pSlow.next;
        }

        return pFast;
    }
```

### [寻找重复数](https://leetcode-cn.com/problems/find-the-duplicate-number/)

【思路一】二分查找

```java
class Solution {
    public int findDuplicate(int[] nums) {
        if (nums == null) {
            return Integer.MIN_VALUE;
        }
        int n = nums.length;
        int left = 1, right = n - 1, ans = -1;
        while (left <= right) {
            int mid = left + ((right - left) >> 1);
            int cnt = 0;
            for (int num : nums) {
                if (num <= mid) {
                    cnt++;
                }
            }
            if (cnt <= mid) {
                left = mid + 1;
            } else {
                right = mid - 1;
                ans = mid;
            }
        }
        return ans;
    }
}


```

【思路二】快慢指针

```java
class Solution {
   public int findDuplicate(int[] nums) {
        if (nums == null) {
            return Integer.MIN_VALUE;
        }
     		// 根据题目条件确认不会溢出
        int slow = nums[0];
        int fast = nums[nums[0]];
        while (slow != fast) {
            slow = nums[slow];
            fast = nums[nums[fast]];
        }
     		// 将其中一个指针放回头部
        fast = 0;
        while (fast != slow) {
            slow = nums[slow];
            fast = nums[fast];
        }
        return fast;
    }
}
```

### [第 k 个缺失的正整数](https://leetcode-cn.com/problems/kth-missing-positive-number/)

```java
class Solution {
    public int findKthPositive(int[] arr, int k) {
        if (arr == null || k <= 0 || arr.length == 0) {
            return -1;
        }
        int c = 1;
        int idx = 0;
        while (true) {
            if (idx == arr.length || k == 0) {
                break;
            }
            if (c < arr[idx]) {
                c++;
                k--;
            } else {
                idx++;
                c++;
            }
        }
        c += k - 1;
        return c;
    }
}
```

