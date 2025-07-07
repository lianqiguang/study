## Rabin-Karp算法实现

```c++
const int PRIME = 101; // 大质数

long calculateHash(const string& str, int len) {
    long hash = 0;
    for (int i = 0; i < len; i++) {
        hash += str[i] * pow(PRIME, i);
    }
    return hash;
}

long recalculateHash(long oldHash, char oldChar, char newChar, int patternLen) {
    long newHash = oldHash - oldChar;
    newHash /= PRIME;
    newHash += newChar * pow(PRIME, patternLen - 1);
    return newHash;
}

bool checkEqual(const string& text, int start, const string& pattern) {
    for (int i = 0; i < pattern.length(); i++) {
        if (text[start + i] != pattern[i]) {
            return false;
        }
    }
    return true;
}

int rabinKarpSearch(const string& text, const string& pattern) {
    int m = pattern.length();
    int n = text.length();
    
    if (m == 0) return 0;
    if (n == 0 || m > n) return -1;
    
    long patternHash = calculateHash(pattern, m);
    long textHash = calculateHash(text, m);
    
    for (int i = 0; i <= n - m; i++) {
        if (patternHash == textHash && checkEqual(text, i, pattern)) {
            return i;
        }
        
        if (i < n - m) {
            textHash = recalculateHash(textHash, text[i], text[i + m], m);
        }
    }
    return -1;
}
```