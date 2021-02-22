> **文章已收录Github精选，欢迎Star**：[https://github.com/yehongzhi](https://github.com/yehongzhi/learningSummary)

# 前言

很多人做leetcode题目找不到方向，或者说很难持之以恒，我这里推荐一种方法，从简单难度开始刷，刷完这个标签的简单难度，再换一个标签，这样循序渐进，把做题的量慢慢提高，还有难度逐渐加大。**对于初学者，最重要是趁热打铁，而不是东打一枪西放一炮，趁热打铁才能形成做题的思路**。

还有一个问题是，一开始做题往往我们没有思路，只会想到暴力解法，效率只有惨淡的5%，遇到这种情况是很正常的，因为还没开始形成解题的思维。我们可以先看看题解，看完思路再自己写，千万不要照抄，要自己想出来才能锻炼编程能力。

所谓`talking is cheap, show me code`，那么我们就从字符串开始吧！

# 20. 有效的括号

**题目**：

给定一个只包括 '('，')'，'{'，'}'，'['，']' 的字符串 s ，判断字符串是否有效。

有效字符串需满足：

1.左括号必须用相同类型的右括号闭合。
2.左括号必须以正确的顺序闭合。

**解题思路**：

这道题可以应用于校验JSON格式的括号是否正确。从题目上可以知道有效的括号是有左括号，也会有相同类型的右括号，并且按照正确的顺序闭合。

那么应该采取什么方法校验呢？我马上想到的是通过成对成对地删除有效的括号，从最里面一直往外层删除，最后能删除完，变成空字符串就代表是有效括号返回true，否则返回false。

代码如下：

```java
public boolean isValid(String s) {
    if (s == null) {
        return false;
    }
    if ("".equals(s)) {
        return true;
    }
    //去掉空格字符
    s = s.replace(" ", "");
    //如果是长度是奇数，直接返回false
    if (s.length() % 2 != 0) {
        return false;
    }
    //如果长度等于0表示是有效的括号，返回true，否则返回false
    return dealString(s) == 0;
}

private int dealString(String s) {
    if (s.length() == 0) {
        return 0;
    }
    for (int i = 0; i < s.length() - 1; i++) {
        //获取相邻的两个字符
        String str = String.valueOf(s.charAt(i)) + s.charAt(i + 1);
        if ("{}".equals(str) || "[]".equals(str) || "()".equals(str)) {
            //如果是有效的括号就删除
            int index = s.indexOf(str);
            s = s.substring(0, index) + s.substring(index + 2);
            //删除后的字符串，再递归继续删除
            return dealString(s);
        }
    }
    return s.length();
}
```

一写完代码，感觉思路清晰，代码整洁，还使用了递归，简直so easy！

然而一运行...成年人的崩溃就在一瞬间！

![](https://static.lovebilibili.com/string_sf_1.png)

为什么会这么低的效率呢，其实想想就知道，我每次遍历字符串就只删一个有效的括号，如果出现类似这种"[[{}{}{}{}{}{}]]"，就会遍历非常多次！所以不能这样玩！

怎么删效率比较高呢？最好是不要重复去遍历，一次遍历删完效率是最高的。

关键是怎么找到**最里层**的有效括号，其实就是找到第一个右括号，然后判断左边的括号是否能匹配，能匹配的话就是最里层的有效括号，然后删除掉。删除后，因为字符串的长度变短了两位，所以我们把指针往左边移动两位即可。这样遍历下来，就只会遍历一次。最后还是判断字符串的长度。

代码如下：

```java
public boolean isValid(String s) {
    if (s == null) {
        return false;
    }
    if ("".equals(s)) {
        return true;
    }
    //去掉空格字符
    s = s.replace(" ", "");
    //如果是长度是奇数，直接返回false
    if (s.length() % 2 != 0) {
        return false;
    }
    StringBuilder toCheck = new StringBuilder(s);
    for (int i = 0; i < toCheck.length(); i++) {
        //获取当前的字符
        char current = toCheck.charAt(i);
        //判断是右边的括号才进行处理
        if (current == ')' || current == ']' || current == '}') {
            //如果第一个符号就是右括号，直接返回false
            if (i == 0) {
                return false;
            }
            //如果右边的括号和左边相邻的括号可以匹配，直接删除，并且指针往左边移动两位
            if (isPair(toCheck.charAt(i - 1), current)) {
                toCheck.delete(i - 1, i + 1);
                i -= 2;
            }
        }
    }
    return toCheck.length() == 0;
}

private boolean isPair(char a, char b) {
    if (b == ')') {
        return a == '(';
    }
    if (b == ']') {
        return a == '[';
    }
    if (b == '}') {
        return a == '{';
    }
    return false;
}
```

![](https://static.lovebilibili.com/string_sf_2.png)

执行用时2ms，还不错，那么这题就算pass了！

当然除了我说的这种方式之外，题解里有大佬是使用栈来解决，大家有兴趣可以看看。

![](https://static.lovebilibili.com/string_sf_3.png)

# 344. 反转字符串

**题目**：

编写一个函数，其作用是将输入的字符串反转过来。输入字符串以字符数组 char[] 的形式给出。

不要给另外的数组分配额外的空间，你必须原地修改输入数组、使用 O(1) 的额外空间解决这一问题。

你可以假设数组中的所有字符都是 ASCII 码表中的可打印字符。

**解题思路**：

一看到这道题，直呼是送分题，这反转字符串不就是JavaAPI就有了吗，于是乎直接大胆的，两行代码搞定，好家伙！一下子重拾回作为程序员的信心！

代码如下：

```java
public void reverseString(char[] s) {
    char[] chars = new StringBuilder(new String(s)).reverse().toString().toCharArray();
    System.arraycopy(chars, 0, s, 0, chars.length);
}
```

但是效率呢？成年人的崩溃往往就在一瞬间！

![](https://static.lovebilibili.com/string_sf_4.png)

用了2ms，仅仅击败了9.82%的用户，证明有更快的解法。

而且题目要求原地修改输入数组、使用 O(1) 的额外空间解决，所以上面的解法不符合题目要求。如果不使用额外空间，最直接的方式马上想到头尾交换，第二位跟倒数第二位交换，一直交换到中间，最后整个char[]数组就反转过来了。

代码如下：

```java
public void reverseString(char[] s) {
    //如果长度小于2，直接返回
    if (s.length < 2) {
        return;
    }
    //右边索引，从末尾开始
    int r = s.length - 1;
    //左边索引，从0开始
    int l = 0;
    //如果左边的索引值小于右边的索引值就循环，否则跳出循环
    while (l < r) {
        //交换值
        char temp = s[l];
        s[l] = s[r];
        s[r] = temp;
        //左右索引向中间移动
        l++;
        r--;
    }
}
```

![](https://static.lovebilibili.com/string_sf_5.png)

这个效率不用芜湖，已经起飞了！上面那个算法其实就是双指针，应该是比较简单高效的解法之一了。

# 387.字符串中的第一个唯一字符

**题目**：

给定一个字符串，找到它的第一个不重复的字符，并返回它的索引。如果不存在，则返回 -1。

示例：

```java
s = "leetcode"
返回 0

s = "loveleetcode"
返回 2
```

**提示**：你可以假定该字符串只包含小写字母。

**解题思路**：

一看到不重复，二话不说直接想到HashMap，什么？还有顺序，那就用LinkedHashMap，key保存字符，value保存出现的次数，遍历完字符串之后，找出第一个不重复的字符即可。

不多哔哔，上代码！

```java
public int firstUniqChar(String s) {
    char[] chars = s.toCharArray();
    Map<Character, Integer> map = new LinkedHashMap<>();
    //统计出现的次数
    for (Character ch : chars) {
        Integer count = map.get(ch);
        if (count == null) {
            map.put(ch, 1);
        } else {
            count++;
            map.put(ch, count);
        }
    }
    //找出第一个出现不重复的字符
    for (Map.Entry<Character, Integer> entry : map.entrySet()) {
        Integer count = entry.getValue();
        if (count == 1) {
            Character ch = entry.getKey();
            //返回下标
            return s.indexOf(ch);
        }
    }
    return -1;
}
```

看看效率如何？38ms！成年人的崩溃往往就在一瞬间...

![](https://static.lovebilibili.com/string_sf_6.png)

思路是没错的，用哈希表解决，但是没有利用上提示，提示说只有小写字母，小写字母只有26个，所以使用一个长度为26的数组作为哈希表即可，使用Map集合的话，put方法里面的逻辑非常多，会浪费性能。

所以我们把Map集合改成长度为26的数组即可，上代码：

```java
public int firstUniqChar(String s) {
    char[] chars = s.toCharArray();
    int[] hashArray = new int[26];
    //统计字符出现的次数
    for (Character ch : chars) {
        hashArray[ch - 'a']++;
    }
    //找出只出现一次的字符，然后返回下标
    for (int i = 0; i < chars.length; i++) {
        Character ch = chars[i];
        int count = hashArray[ch - 'a'];
        if (count == 1) {
            return i;
        }
    }
    return -1;
}
```

不用说，效率肯定提升数倍，7ms！

![](https://static.lovebilibili.com/string_sf_7.png)

# 125.验证回文串

**题目**：

给定一个字符串，验证它是否是回文串，只考虑字母和数字字符，可以忽略字母的大小写。

**说明**：本题中，我们将空字符串定义为有效的回文串。

**示例 1**:

```java
输入: "A man, a plan, a canal: Panama"
输出: true
```

**示例 2**:

```java
输入: "race a car"
输出: false
```

**解题思路**：

什么是回文串呢，就是一个字符串从左往右读，然后从右往左读都是一样的，比如"aabaa"，也就是把字符串反转过来跟原来的字符串是相等的，就称为回文串。

题目说只考虑字母和数字字符，并且忽略大小写，然后验证是不是回文串。很简单啦，遍历字符串，然后根据题目过滤出字母和数字字符，然后拼接出新的字符串，最后反转对比一下，如果相等就返回true，不相等就返回false。

不多哔哔，直接上代码！

```java
public boolean isPalindrome(String s) {
    StringBuilder sb = new StringBuilder();
    for (char c : s.toCharArray()) {
        //只拼接数字和字母的字符
        if (Character.isDigit(c) || Character.isLetter(c)) {
            //如果是字母则全部转为小写
            if (Character.isLetter(c)) {
                c = Character.toLowerCase(c);
            }
            sb.append(c);
        }
    }
    //反转字符串，验证是否是回文串
    String s1 = sb.toString();
    return s1.equals(sb.reverse().toString());
}
```

居然用时7ms，我不能接受！

![](https://static.lovebilibili.com/string_sf_8.png)

其实判断回文串根本没必要遍历完整个字符串，可以一边遍历一边判断的！

因为是对称的，所以利用双指针，一个指针从左往右，一个指针从右往左，左右两边取值，对比符合条件(数字或者字母)的字符，如果中途发现不相等直接返回false，如果遍历完都是相等的话，那就返回true。

废话不多说，直接看代码。

```java
public boolean isPalindrome(String s) {
    char[] chars = s.toCharArray();
    int leftIndex = 0;
    int rightIndex = chars.length - 1;
    char left;
    char right;
    while (leftIndex < rightIndex) {
        //过滤掉不是数字或者字母的字符
        while (leftIndex < rightIndex && !Character.isLetterOrDigit(chars[leftIndex])) {
            leftIndex++;
        }
        //左边索引的值
        left = chars[leftIndex];
        //过滤掉不是数字或者字母的字符
        while (leftIndex < rightIndex && !Character.isLetterOrDigit(chars[rightIndex])) {
            rightIndex--;
        }
        //右边索引的值
        right = chars[rightIndex];
        //如果不相等则返回false
        if (Character.toLowerCase(left) != Character.toLowerCase(right)) {
            return false;
        } else {
            //如果相等则继续比较，左指针往右移，右指针往左移
            leftIndex++;
            rightIndex--;
        }
    }
    return true;
}
```

桀桀桀，双指针的威力竟恐怖如斯！

![](https://static.lovebilibili.com/string_sf_9.png)

# 总结

上面这四道题，表面看起来好像有点难度，实际上就是Paper Tiger(纸老虎)，没什么好怕的，做多了之后自然就有思路了。对于初学者，不要怕使用暴力解法，先用暴力解法找找感觉，找找思路，然后看看能不能优化一下暴力解法。

上面讲了四道关于字符串的算法题，因为不可能一篇文章讲完所有的题目，所以如果希望提高自己的编程能力，还需要自己到leetcode上做一做。

这篇文章就讲到这里，最后感谢大家的阅读，希望能给大家带来一些启发。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![img](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！