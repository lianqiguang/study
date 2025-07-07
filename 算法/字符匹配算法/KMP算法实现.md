## KMP算法实现

```c++
#include <vector>
#include <string>
using namespace std;

vector<int> computeLPS(const string& pattern) {
    int n = pattern.length();
    vector<int> lps(n, 0);
    int len = 0;
    int i = 1;
    
    while (i < n) {
        if (pattern[i] == pattern[len]) {
            len++;
            lps[i] = len;
            i++;
        } else {
            if (len != 0) {
                len = lps[len - 1];
            } else {
                lps[i] = 0;
                i++;
            }
        }
    }
    return lps;
}

int kmpSearch(const string& text, const string& pattern) {
    int m = pattern.length();
    int n = text.length();
    
    if (m == 0) return 0;
    if (n == 0 || m > n) return -1;
    
    vector<int> lps = computeLPS(pattern);
    int i = 0; // text指针
    int j = 0; // pattern指针
    
    while (i < n) {
        if (pattern[j] == text[i]) {
            i++;
            j++;
        }
        
        if (j == m) {
            return i - j; // 找到匹配
        } else if (i < n && pattern[j] != text[i]) {
            if (j != 0) {
                j = lps[j - 1];
            } else {
                i++;
            }
        }
    }
    return -1;
}
```