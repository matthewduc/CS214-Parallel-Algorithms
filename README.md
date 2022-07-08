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
## Project 2
#### Parallel BFS using scan, prefix sum, and effective neighbors to calculate distance from a point of origin

```` C++
int scan_down(int*A, int*B, int* LS, int n, int offset){
   if(n==1){
       B[0] = offset + A[0];
       return 0;
   }
   int m = n/2;
   int L, R;
   L = cilk_spawn scan_down(A, B, LS, m, offset);
   R = scan_down(A+m, B+m, LS+m, n-m, offset+LS[m-1]);
   cilk_sync;
   return L+R;
}

int scan_up(int* A, int* LS, int n){
   if(n==1) return A[0];
   int m = n/2;
   int L, R;
   L = cilk_spawn scan_up(A, LS, m);
   R = scan_up(A+m, LS+m, n-m);
   LS[m-1]=L;
   cilk_sync;
   return L+R;
}

void scan(int* A, int* B, int n){
    int* LS = new int[n-1];
    scan_up(A, LS, n);
    scan_down(A, B, LS, n, 0);
}

bool CAS(bool* p, bool vold, bool vnew) {
    if (*p==vold){
        *p = vnew;
        return true;
    }
    else false;
}

void Filter(int* A, int n, int* B, int* flagsA){
    int* ps = new int[n];
    cilk_for(int i=0;i<n;i++){
        ps[i] = 0;
    }
    scan(flagsA, ps, n);
    cilk_for (int i = 0; i<n;i++){
        B[ps[i]-1] = A[i];
    }
}

int* next_Frontier(int n, int m, int* offset, int*E, int* frontier, int* dist, bool* visited, int &start, int &end){
    int offsetCount, newFrontCount = 0;
    // cilk_for x in current frontier
    cilk_for(int x=start; x<end; x++){
        int current_node = frontier[x];
        // only visit if frontier value is not -1
        if(current_node!=-1) {
            if (start<n) start++;
            int out_neighbors = offset[current_node+1]-offset[current_node];
            int* neighbors = new int[out_neighbors];
            int* effective_neighbors = new int[out_neighbors];
            int* neighbors_flag = new int[out_neighbors];
            
            for(int i=0;i<out_neighbors;i++){
                effective_neighbors[i] = -1;
                neighbors_flag[i] = 0;
                int v = E[offset[current_node]+i];
                neighbors[i] = v;
                if((!visited[v]) && CAS(&visited[v], false,true)){
                    neighbors_flag[i] = 1;
                    
                    int cur_dist = dist[current_node];
                    if (dist[v] == -1) {
                        dist[v] = cur_dist + 1;
                    }
                }
            }
            Filter(neighbors, out_neighbors, effective_neighbors, neighbors_flag);
            cilk_for(int i = 0;i<out_neighbors;i++){
                if(effective_neighbors[i] != -1){
                    frontier[end++] = effective_neighbors[i];
                }
            }
        }
    }
    return frontier;
}

void sequential_BFS(int n, int m, int* offset, int* E, int s, int* dist) {
    for (int i = 0; i < n; i++) dist[i] = -1;

    dist[s] = 0;
    int* q = new int[n];
    q[0] = s;
    int start = 0, end = 1;
    while (start != end) {
        int cur_node = q[start];
        int cur_dis = dist[cur_node];

        for (int i = offset[cur_node]; i < offset[cur_node+1]; i++) {
            int v = E[i];
            if (dist[v] == -1) {
                dist[v] = cur_dis + 1;
                q[end++] = v;
            }
        }
        start++;
    }
}

void BFS(int n, int m, int* offset, int* E, int s, int* dist) {
    cilk_for(int i =0; i<n; i++) dist[i] = -1;
    dist[s] = 0;
    
    bool* visited = new bool[n];
    cilk_for (int i = 0; i < n; i++) visited[i] = false;
    
    int* frontier = new int[n];
    cilk_for (int i = 0; i < n; i++) frontier[i] = -1;
    
    frontier[0] = s;
    
    // while |frontier|
    int start = 0; int end = 1;
    while(start!=end){
        if (end == n) end--;
        frontier = next_Frontier(n, m, offset, E, frontier, dist, visited, start, end);

    }
}
````
