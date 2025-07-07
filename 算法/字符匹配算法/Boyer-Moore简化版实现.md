## Boyer-Moore算法实现

```c++
#include <unordered_map>
#include <algorithm>

unordered_map<char, int> buildBadCharTable(const string& pattern) {
    unordered_map<char, int> badChar;
    int m = pattern.length();
    
    for (int i = 0; i < m - 1; i++) {
        badChar[pattern[i]] = i;
    }
    return badChar;
}

vector<int> buildGoodSuffixTable(const string& pattern) {
    int m = pattern.length();
    vector<int> goodSuffix(m, 0);
    vector<int> suffix(m, 0);
    
    // 构建suffix数组
    suffix[m - 1] = m;
    for (int i = m - 2; i >= 0; i--) {
        int j = i;
        while (j >= 0 && pattern[j] == pattern[m - 1 - i + j]) {
            j--;
        }
        suffix[i] = i - j;
    }
    
    // Case 1: 完全匹配好后缀
    for (int i = 0; i < m; i++) {
        goodSuffix[i] = m;
    }
    
    // Case 2: 部分匹配好后缀
    int j = 0;
    for (int i = m - 1; i >= 0; i--) {
        if (suffix[i] == i + 1) {
            while (j < m - 1 - i) {
                if (goodSuffix[j] == m) {
                    goodSuffix[j] = m - 1 - i;
                }
                j++;
            }
        }
    }
    
    // Case 3: 好后缀的前缀
    for (int i = 0; i < m - 1; i++) {
        goodSuffix[m - 1 - suffix[i]] = m - 1 - i;
    }
    
    return goodSuffix;
}

int boyerMooreSearch(const string& text, const string& pattern) {
    int m = pattern.length();
    int n = text.length();
    
    if (m == 0) return 0;
    if (n == 0 || m > n) return -1;
    
    unordered_map<char, int> badChar = buildBadCharTable(pattern);
    vector<int> goodSuffix = buildGoodSuffixTable(pattern);
    
    int s = 0; // 文本中的偏移量
    while (s <= n - m) {
        int j = m - 1;
        
        // 从右向左匹配
        while (j >= 0 && pattern[j] == text[s + j]) {
            j--;
        }
        
        if (j < 0) {
            return s; // 匹配成功
        } else {
            // 坏字符规则
            int badCharShift = j - badChar[text[s + j]];
            if (badChar.find(text[s + j]) == badChar.end()) {
                badCharShift = j + 1;
            }
            
            // 好后缀规则
            int goodSuffixShift = goodSuffix[j];
            
            // 取两者中较大的值
            s += max(badCharShift, goodSuffixShift);
        }
    }
    return -1;
}
```