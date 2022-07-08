# CS214-Parallel-Algorithms
## Project 1
#### Parallel Quicksort using CILK, Flag position based on pivot and control to sort large arrays
```` C++
size_t Filter(T* A, size_t n, size_t control, size_t pivot, size_t* output, size_t position, size_t* flag, size_t* ps){
    size_t* LS = new size_t[n-1];
    cilk_for(int i=0;i<n;i++){
        flag[i] = 0;
        ps[i] = 0;
        // Less Than Pivot
        if (control == -1){
            if (A[i] < pivot){
                flag[i] = 1;
            }
        }
        // Equal To Pivot
        if (control == 0){
            if (A[i] == pivot){
                flag[i] = 1;
            }
        }
        // Greater Than Pivot
        if (control == 1){
            if (A[i] > pivot){
                flag[i] = 1;
            }
        }
    }
    std::atomic<size_t> increment = position;
    cilk_for (size_t i = 0; i<n;i++){
        if(flag[i] == 1){
            output[increment++] = A[i];
        }
    }
    position = increment;
    return position;
}

void quicksort(T* A, size_t n) {
    size_t leftPointer, middlePointer, rightPointer;
    if (n <= 35000000){
        std::sort(A, A+n);
        return;
    } else {
        // Sample Array A to find a "good" pivot, e.g. 1000
        size_t* B = new size_t[1000];
        cilk_for(int i=0; i<1000; i++){
            B[i] = A[i];
        }
        // Sort B and use middle number as pivot
        std::sort(B, B+1000);
        size_t pivot = B[500];
        // Partition Array into Left, Equal, Right based on pivot
        size_t* output = new size_t[n];
        size_t* flagLess = new size_t[n];
        size_t* equalTo = new size_t[n];
        size_t* flagGreater = new size_t[n];
        size_t* ps = new size_t[n];

        leftPointer = Filter(A, n, -1, pivot, output, 0, flagLess, ps);
        middlePointer = Filter(A, n, 0, pivot, output, leftPointer, equalTo, ps);
        rightPointer = Filter(A, n, 1, pivot, output, middlePointer, flagGreater, ps);
        
        cilk_for(int i=0;i<n;i++){
            A[i] = output[i];
        }
    }
    cilk_spawn quicksort(A, leftPointer);
    quicksort(A+middlePointer, n-middlePointer);
    cilk_sync;
}
````
