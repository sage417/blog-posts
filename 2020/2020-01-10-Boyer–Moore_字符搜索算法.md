---
title: 'Boyer–Moore 字符搜索算法'
date: '2020-01-10 00:00:00'
cover: https://i.loli.net/2020/01/14/zQSP9xfZnBoYLNc.jpg
tags:
    - 工作
    - 代码
    - 算法
categories:
    - 算法
    - 字符串
---

因为字符比较是从右往左比较的，所以第一层循环  `needle.length + 1` <= i < `haystack.length`。

```
                  start=needle.length - 1                                             end=haystack.length - 1
                          +                                                                   +
                          |                                                                   |
                          |                                                                   |
                          v-------------------------------------------------------------------v->
  0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15  16  17  18  19  20  21  22  23
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
| t | h | i | s |   | i | s |   | a |   | s | i | m | p | l | e |   | e | x | a | m | p | l | e |
+-------------------------------+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
+---------------------------+
| e | x | a | m | p | l | e |
+---+---+---+---+---+---+---+

```

第二层循环中i变量表示坏字符的位置、j表示搜索坏字符开始位置

```
                          i
                          +
                          |
                          v
  0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15  16  17  18  19  20  21  22  23
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
| t | h | i | s |   | i | s |   | a |   | s | i | m | p | l | e |   | e | x | a | m | p | l | e |
+-------------------------------+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
+---------------------------+
| e | x | a | m | p | l | e |
+---+---+---+---+---+---+-+-+
                          ^
                          |
                          +
                          j

```

i,j指针向左搜索,如果完全匹配直接返回i即可

```java
for (j = needle.length - 1; needle[j] == haystack[i]; --i, --j) {
    if (j == 0) {
        return i;
    }
}
```

### 坏字符规则和好字符规则

坏字符:从右向左第一个不匹配的字符
好字符:从坏字符下一个字符直到最后的字符

可以认为字符移动有两种策略：坏字符对齐和好字符对齐，然后选择字符移动距离大的策略即可

### wiki实现

```java
    /**
     * Returns the index within this string of the first occurrence of the
     * specified substring. If it is not a substring, return -1.
     *
     * There is no Galil because it only generates one match.
     *
     * @param haystack The string to be scanned
     * @param needle The target string to search
     * @return The start index of the substring
     */
    public static int indexOf(char[] haystack, char[] needle) {
        if (needle.length == 0) {
            return 0;
        }
        int charTable[] = makeCharTable(needle);
        int offsetTable[] = makeOffsetTable(needle);
        for (int i = needle.length - 1, j; i < haystack.length;) {
            for (j = needle.length - 1; needle[j] == haystack[i]; --i, --j) {
                if (j == 0) {
                    return i;
                }
            }
            // i += needle.length - j; // For naive method
            i += Math.max(offsetTable[needle.length - 1 - j], charTable[haystack[i]]);
        }
        return -1;
    }

    /**
     * Makes the jump table based on the mismatched character information.
     */
    private static int[] makeCharTable(char[] needle) {
        final int ALPHABET_SIZE = Character.MAX_VALUE + 1; // 65536
        int[] table = new int[ALPHABET_SIZE];
        for (int i = 0; i < table.length; ++i) {
            table[i] = needle.length;
        }
        for (int i = 0; i < needle.length - 2; ++i) {
            table[needle[i]] = needle.length - 1 - i;
        }
        return table;
    }

    /**
     * Makes the jump table based on the scan offset which mismatch occurs.
     * (bad character rule).
     */
    private static int[] makeOffsetTable(char[] needle) {
        int[] table = new int[needle.length];
        int lastPrefixPosition = needle.length;
        for (int i = needle.length; i > 0; --i) {
            if (isPrefix(needle, i)) {
                lastPrefixPosition = i;
            }
            table[needle.length - i] = lastPrefixPosition - i + needle.length;
        }
        for (int i = 0; i < needle.length - 1; ++i) {
            int slen = suffixLength(needle, i);
            table[slen] = needle.length - 1 - i + slen;
        }
        return table;
    }

    /**
     * Is needle[p:end] a prefix of needle?
     */
    private static boolean isPrefix(char[] needle, int p) {
        for (int i = p, j = 0; i < needle.length; ++i, ++j) {
            if (needle[i] != needle[j]) {
                return false;
            }
        }
        return true;
    }

    /**
     * Returns the maximum length of the substring ends at p and is a suffix.
     * (good suffix rule)
     */
    private static int suffixLength(char[] needle, int p) {
        int len = 0;
        for (int i = p, j = needle.length - 1;
                 i >= 0 && needle[i] == needle[j]; --i, --j) {
            len += 1;
        }
        return len;
    }
```

