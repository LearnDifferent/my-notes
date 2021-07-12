快速排序 Quicksort 最新版：

```java
public static void quickSort(int[] arr) {
    quickSort(arr, 0, arr.length - 1);
}

public static void quickSort(int[] arr, int head, int tail) {

    if (head < tail) {
        int h = head;
        int t = tail;
        int pivot = arr[(tail + head) / 2];

        while (h < t) {
            while (arr[h] < pivot) {
                h++;
            }
            while (arr[t] > pivot) {
                t--;
            }
            if (h < t) {
                int temp = arr[t];
                arr[t] = arr[h];
                arr[h] = temp;
                h++;
                t--;
            } else {
                h++;
            }
        }
        quickSort(arr, h, tail);
        quickSort(arr, head, t);
    }
}
```