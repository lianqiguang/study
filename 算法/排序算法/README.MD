# 八大排序算法

排序算法是计算机科学中最基础且重要的算法之一。以下是八种经典的排序算法及其特点：

## 1. 冒泡排序 (Bubble Sort)

- **原理**：重复遍历列表，比较相邻元素并交换顺序错误的元素
- **时间复杂度**：
  - 最好：O(n)（已排序时）
  - 平均/最差：O(n²)
- **空间复杂度**：O(1)
- **稳定性**：稳定

## 2. 选择排序 (Selection Sort)

- **原理**：每次从未排序部分选择最小(或最大)元素放到已排序部分的末尾
- **时间复杂度**：O(n²)（所有情况）
- **空间复杂度**：O(1)
- **稳定性**：不稳定

## 3. 插入排序 (Insertion Sort)

- **原理**：构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置插入
- **时间复杂度**：
  - 最好：O(n)（已排序时）
  - 平均/最差：O(n²)
- **空间复杂度**：O(1)
- **稳定性**：稳定

## 4. 希尔排序 (Shell Sort)

- **原理**：插入排序的改进版，通过将数组分成多个子序列进行插入排序
- **时间复杂度**：取决于增量序列，通常为O(n^(3/2))到O(n²)
- **空间复杂度**：O(1)
- **稳定性**：不稳定

## 5. 归并排序 (Merge Sort)

- **原理**：分治法，将数组分成两半分别排序，然后合并
- **时间复杂度**：O(n log n)（所有情况）
- **空间复杂度**：O(n)
- **稳定性**：稳定

## 6. 快速排序 (Quick Sort)

- **原理**：分治法，选择一个基准元素，将数组分为小于基准和大于基准的两部分，递归排序
- **时间复杂度**：
  - 最好/平均：O(n log n)
  - 最差：O(n²)（当分区极度不平衡时）
- **空间复杂度**：O(log n)（递归栈）
- **稳定性**：不稳定

## 7. 堆排序 (Heap Sort)

- **原理**：利用堆数据结构进行排序，先构建最大堆，然后反复取出堆顶元素
- **时间复杂度**：O(n log n)（所有情况）
- **空间复杂度**：O(1)
- **稳定性**：不稳定

## 8. 基数排序 (Radix Sort)

- **原理**：按位排序，从最低位到最高位依次排序
- **时间复杂度**：O(nk)（k为数字位数）
- **空间复杂度**：O(n+k)
- **稳定性**：稳定

## 总结对比

| 排序算法 | 平均时间复杂度 | 最坏时间复杂度 | 空间复杂度 | 稳定性 |
| :------: | :------------: | :------------: | :--------: | :----: |
| 冒泡排序 |     O(n²)      |     O(n²)      |    O(1)    |  稳定  |
| 选择排序 |     O(n²)      |     O(n²)      |    O(1)    | 不稳定 |
| 插入排序 |     O(n²)      |     O(n²)      |    O(1)    |  稳定  |
| 希尔排序 |   O(n log n)   |     O(n²)      |    O(1)    | 不稳定 |
| 归并排序 |   O(n log n)   |   O(n log n)   |    O(n)    |  稳定  |
| 快速排序 |   O(n log n)   |     O(n²)      |  O(log n)  | 不稳定 |
|  堆排序  |   O(n log n)   |   O(n log n)   |    O(1)    | 不稳定 |
| 基数排序 |     O(nk)      |     O(nk)      |   O(n+k)   |  稳定  |

在实际应用中，快速排序通常是最快的通用排序算法，而归并排序因其稳定性常用于需要稳定排序的场景。基数排序则适用于特定类型的数据（如整数或字符串）。



以下是使用C++实现的八大排序算法代码示例：

### 1. 冒泡排序 (Bubble Sort)

```c++
void bubbleSort(int arr[], int n) {
    for (int i = 0; i < n-1; i++) {
        bool swapped = false;
        for (int j = 0; j < n-i-1; j++) {
            if (arr[j] > arr[j+1]) {
                std::swap(arr[j], arr[j+1]);
                swapped = true;
            }
        }
        if (!swapped) break; // 如果没有交换，说明已经有序
    }
}
```

### 2. 选择排序 (Selection Sort)

```c++
void selectionSort(int arr[], int n) {
    for (int i = 0; i < n-1; i++) {
        int min_idx = i;
        for (int j = i+1; j < n; j++) {
            if (arr[j] < arr[min_idx]) {
                min_idx = j;
            }
        }
        std::swap(arr[i], arr[min_idx]);
    }
}
```

### 3. 插入排序 (Insertion Sort)

```c++
void insertionSort(int arr[], int n) {
    for (int i = 1; i < n; i++) {
        int key = arr[i];
        int j = i - 1;
        while (j >= 0 && arr[j] > key) {
            arr[j+1] = arr[j];
            j--;
        }
        arr[j+1] = key;
    }
}
```

### 4. 希尔排序 (Shell Sort)

```c++
void shellSort(int arr[], int n) {
    for (int gap = n/2; gap > 0; gap /= 2) {
        for (int i = gap; i < n; i++) {
            int temp = arr[i];
            int j;
            for (j = i; j >= gap && arr[j - gap] > temp; j -= gap) {
                arr[j] = arr[j - gap];
            }
            arr[j] = temp;
        }
    }
}
```

### 5. 归并排序 (Merge Sort)

```c++
void merge(int arr[], int l, int m, int r) {
    int n1 = m - l + 1;
    int n2 = r - m;
    
    int L[n1], R[n2];
    for (int i = 0; i < n1; i++) L[i] = arr[l + i];
    for (int j = 0; j < n2; j++) R[j] = arr[m + 1 + j];
    
    int i = 0, j = 0, k = l;
    while (i < n1 && j < n2) {
        if (L[i] <= R[j]) arr[k++] = L[i++];
        else arr[k++] = R[j++];
    }
    
    while (i < n1) arr[k++] = L[i++];
    while (j < n2) arr[k++] = R[j++];
}

void mergeSort(int arr[], int l, int r) {
    if (l >= r) return;
    int m = l + (r - l) / 2;
    mergeSort(arr, l, m);
    mergeSort(arr, m+1, r);
    merge(arr, l, m, r);
}
```

### 6. 快速排序 (Quick Sort)

```c++
int partition(int arr[], int low, int high) {
    int pivot = arr[high];
    int i = low - 1;
    
    for (int j = low; j <= high-1; j++) {
        if (arr[j] < pivot) {
            i++;
            std::swap(arr[i], arr[j]);
        }
    }
    std::swap(arr[i+1], arr[high]);
    return i+1;
}

void quickSort(int arr[], int low, int high) {
    if (low < high) {
        int pi = partition(arr, low, high);
        quickSort(arr, low, pi-1);
        quickSort(arr, pi+1, high);
    }
}
```

### 7. 堆排序 (Heap Sort)

```c++
void heapify(int arr[], int n, int i) {
    int largest = i;
    int l = 2*i + 1;
    int r = 2*i + 2;
    
    if (l < n && arr[l] > arr[largest]) largest = l;
    if (r < n && arr[r] > arr[largest]) largest = r;
    
    if (largest != i) {
        std::swap(arr[i], arr[largest]);
        heapify(arr, n, largest);
    }
}

void heapSort(int arr[], int n) {
    for (int i = n/2 - 1; i >= 0; i--)
        heapify(arr, n, i);
    
    for (int i = n-1; i > 0; i--) {
        std::swap(arr[0], arr[i]);
        heapify(arr, i, 0);
    }
}
```

### 8. 基数排序 (Radix Sort)

```c++
int getMax(int arr[], int n) {
    int mx = arr[0];
    for (int i = 1; i < n; i++)
        if (arr[i] > mx) mx = arr[i];
    return mx;
}

void countSort(int arr[], int n, int exp) {
    int output[n];
    int count[10] = {0};
    
    for (int i = 0; i < n; i++)
        count[(arr[i]/exp)%10]++;
    
    for (int i = 1; i < 10; i++)
        count[i] += count[i-1];
    
    for (int i = n-1; i >= 0; i--) {
        output[count[(arr[i]/exp)%10]-1] = arr[i];
        count[(arr[i]/exp)%10]--;
    }
    
    for (int i = 0; i < n; i++)
        arr[i] = output[i];
}

void radixSort(int arr[], int n) {
    int m = getMax(arr, n);
    for (int exp = 1; m/exp > 0; exp *= 10)
        countSort(arr, n, exp);
}
```

### 使用示例

```c++
#include <iostream>
#include <algorithm>

int main() {
    int arr[] = {64, 34, 25, 12, 22, 11, 90};
    int n = sizeof(arr)/sizeof(arr[0]);
    
    // 选择要使用的排序算法
    // bubbleSort(arr, n);
    // selectionSort(arr, n);
    // insertionSort(arr, n);
    // shellSort(arr, n);
    // mergeSort(arr, 0, n-1);
    // quickSort(arr, 0, n-1);
    // heapSort(arr, n);
    // radixSort(arr, n);
    
    std::cout << "Sorted array: \n";
    for (int i = 0; i < n; i++)
        std::cout << arr[i] << " ";
    std::cout << std::endl;
    
    return 0;
}
```

这些实现涵盖了八大经典排序算法，每种算法都有其特定的应用场景和优缺点。在实际应用中，可以根据数据特点和性能要求选择合适的排序算法。