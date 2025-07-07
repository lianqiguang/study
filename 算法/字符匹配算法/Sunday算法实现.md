## Sunday算法实现

```c++
#include <unordered_map>

unordered_map<char, int> buildShiftTable(const string& pattern) {
    unordered_map<char, int> shift;
    int m = pattern.length();
    
    for (int i = 0; i < m; i++) {
        shift[pattern[i]] = m - i;
    }
    return shift;
}

int sundaySearch(const string& text, const string& pattern) {
    int m = pattern.length();
    int n = text.length();
    
    if (m == 0) return 0;
    if (n == 0 || m > n) return -1;
    
    unordered_map<char, int> shift = buildShiftTable(pattern);
    int i = 0;
    
    while (i <= n - m) {
        int j = 0;
        while (j < m && text[i + j] == pattern[j]) {
            j++;
        }
        
        if (j == m) {
            return i; // 匹配成功
        }
        
        if (i + m >= n) {
            return -1;
        }
        
        // 计算跳跃距离
        char nextChar = text[i + m];
        i += (shift.find(nextChar) != shift.end()) ? shift[nextChar] : m + 1;
    }
    return -1;
}
```