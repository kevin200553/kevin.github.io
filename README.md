#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>
#include <windows.h>

// 物品结构体
typedef struct {
    int id;        // 物品编号
    int weight;    // 重量
    double value;  // 价值
} Item;

// 结果结构体
typedef struct {
    int *selected; // 选择的物品编号
    int count;     // 选择物品数量
    double total_value; // 总价值
    int total_weight;   // 总重量
} Result;

// 获取当前时间（毫秒）
double get_time_ms() {
    LARGE_INTEGER freq, counter;
    QueryPerformanceFrequency(&freq);
    QueryPerformanceCounter(&counter);
    return (double)(counter.QuadPart * 1000.0 / freq.QuadPart);
}

// 初始化物品
void init_items(Item *items, int n) {
    srand(42); // Fixed seed for reproducibility
    for (int i = 0; i < n; i++) {
        items[i].id = i + 1;
        items[i].weight = rand() % 100 + 1; // 1~100
        items[i].value = (rand() % 90001 + 10000) / 100.0; // 100.00~999.99
    }
}

// 释放结果内存
void free_result(Result *res) {
    if (res->selected) {
        free(res->selected);
    }
    if (res) {
        free(res);
    }
}

// 输出结果
void print_result(Result *res, Item *items, FILE *file) {
    fprintf(file, "Selected items: ");
    for (int i = 0; i < res->count; i++) {
        fprintf(file, "%d ", res->selected[i]);
    }
    fprintf(file, "\nTotal weight: %d\n", res->total_weight);
    fprintf(file, "Total value: %.2f\n", res->total_value);
    printf("Selected items count: %d\n", res->count);
    printf("Total weight: %d\n", res->total_weight);
    printf("Total value: %.2f\n\n", res->total_value);
}

// 蛮力法
Result* brute_force(Item *items, int n, int capacity) {
    Result *res = (Result*)malloc(sizeof(Result));
    if (!res) {
        perror("malloc failed for res");
        exit(EXIT_FAILURE);
    }
    res->selected = (int*)malloc(n * sizeof(int));
    if (!res->selected) {
        perror("malloc failed for res->selected");
        free(res);
        exit(EXIT_FAILURE);
    }
    res->count = 0;
    res->total_value = 0;
    res->total_weight = 0;
    
    int max_mask = 1 << n;
    double best_value = 0;
    int best_weight = 0;
    int best_mask = 0;
    
    for (int mask = 0; mask < max_mask; mask++) {
        double curr_value = 0;
        int curr_weight = 0;
        for (int i = 0; i < n; i++) {
            if (mask & (1 << i)) {
                curr_weight += items[i].weight;
                curr_value += items[i].value;
            }
        }
        if (curr_weight <= capacity && curr_value > best_value) {
            best_value = curr_value;
            best_weight = curr_weight;
            best_mask = mask;
        }
    }
    
    for (int i = 0; i < n; i++) {
        if (best_mask & (1 << i)) {
            res->selected[res->count++] = items[i].id;
        }
    }
    res->total_value = best_value;
    res->total_weight = best_weight;
    return res;
}

// 动态规划法
Result* dynamic_programming(Item *items, int n, int capacity) {
    Result *res = (Result*)malloc(sizeof(Result));
    if (!res) {
        perror("malloc failed for res");
        exit(EXIT_FAILURE);
    }
    res->selected = (int*)malloc(n * sizeof(int));
    if (!res->selected) {
        perror("malloc failed for res->selected");
        free(res);
        exit(EXIT_FAILURE);
    }
    res->count = 0;
    res->total_value = 0;
    res->total_weight = 0;
    
    double **dp = (double**)malloc((n + 1) * sizeof(double*));
    if (!dp) {
        perror("malloc failed for dp");
        exit(EXIT_FAILURE);
    }
    int **keep = (int**)malloc((n + 1) * sizeof(int*));
    if (!keep) {
        perror("malloc failed for keep");
        free(dp);
        exit(EXIT_FAILURE);
    }
    for (int i = 0; i <= n; i++) {
        dp[i] = (double*)calloc(capacity + 1, sizeof(double));
        keep[i] = (int*)calloc(capacity + 1, sizeof(int));
        if (!dp[i] || !keep[i]) {
            perror("calloc failed");
            for (int j = 0; j < i; j++) {
                free(dp[j]);
                free(keep[j]);
            }
            free(dp);
            free(keep);
            free(res->selected);
            free(res);
            exit(EXIT_FAILURE);
        }
    }
    
    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= capacity; w++) {
            if (items[i-1].weight <= w) {
                double include = items[i-1].value + dp[i-1][w - items[i-1].weight];
                if (include > dp[i-1][w]) {
                    dp[i][w] = include;
                    keep[i][w] = 1;
                } else {
                    dp[i][w] = dp[i-1][w];
                }
            } else {
                dp[i][w] = dp[i-1][w];
            }
        }
    }
    
    int w = capacity;
    for (int i = n; i > 0; i--) {
        if (keep[i][w]) {
            res->selected[res->count++] = items[i-1].id;
            res->total_weight += items[i-1].weight;
            res->total_value += items[i-1].value;
            w -= items[i-1].weight;
        }
    }
    
    for (int i = 0; i <= n; i++) {
        free(dp[i]);
        free(keep[i]);
    }
    free(dp);
    free(keep);
    return res;
}

// 贪心法
int compare(const void *a, const void *b) {
    Item *item1 = (Item*)a;
    Item *item2 = (Item*)b;
    double ratio1 = item1->value / item1->weight;
    double ratio2 = item2->value / item2->weight;
    return ratio1 > ratio2 ? -1 : (ratio1 < ratio2 ? 1 : 0);
}

Result* greedy(Item *items, int n, int capacity) {
    Result *res = (Result*)malloc(sizeof(Result));
    if (!res) {
        perror("malloc failed for res");
        exit(EXIT_FAILURE);
    }
    res->selected = (int*)malloc(n * sizeof(int));
    if (!res->selected) {
        perror("malloc failed for res->selected");
        free(res);
        exit(EXIT_FAILURE);
    }
    res->count = 0;
    res->total_value = 0;
    res->total_weight = 0;
    
    Item *sorted = (Item*)malloc(n * sizeof(Item));
    if (!sorted) {
        perror("malloc failed for sorted");
        free(res->selected);
        free(res);
        exit(EXIT_FAILURE);
    }
    memcpy(sorted, items, n * sizeof(Item));
    qsort(sorted, n, sizeof(Item), compare);
    
    int curr_weight = 0;
    for (int i = 0; i < n; i++) {
        if (curr_weight + sorted[i].weight <= capacity) {
            res->selected[res->count++] = sorted[i].id;
            res->total_weight += sorted[i].weight;
            res->total_value += sorted[i].value;
            curr_weight += sorted[i].weight;
        }
    }
    
    free(sorted);
    return res;
}

// 回溯法
void backtrack(Item *items, int n, int capacity, int idx, double curr_value, int curr_weight,
               int *temp, int temp_count, Result *best) {
    if (curr_value > best->total_value) {
        best->count = temp_count;
        best->total_value = curr_value;
        best->total_weight = curr_weight;
        for (int i = 0; i < temp_count; i++) {
            best->selected[i] = temp[i];
        }
    }
    
    if (idx >= n) return;
    
    // 不选当前物品
    backtrack(items, n, capacity, idx + 1, curr_value, curr_weight, temp, temp_count, best);
    
    // 选当前物品（如果可行）
    if (curr_weight + items[idx].weight <= capacity) {
        temp[temp_count] = items[idx].id;
        backtrack(items, n, capacity, idx + 1, curr_value + items[idx].value,
                  curr_weight + items[idx].weight, temp, temp_count + 1, best);
    }
}

Result* backtracking(Item *items, int n, int capacity) {
    Result *res = (Result*)malloc(sizeof(Result));
    if (!res) {
        perror("malloc failed for res");
        exit(EXIT_FAILURE);
    }
    res->selected = (int*)malloc(n * sizeof(int));
    if (!res->selected) {
        perror("malloc failed for res->selected");
        free(res);
        exit(EXIT_FAILURE);
    }
    res->count = 0;
    res->total_value = 0;
    res->total_weight = 0;
    
    int *temp = (int*)malloc(n * sizeof(int));
    if (!temp) {
        perror("malloc failed for temp");
        free(res->selected);
        free(res);
        exit(EXIT_FAILURE);
    }
    backtrack(items, n, capacity, 0, 0, 0, temp, 0, res);
    free(temp);
    return res;
}

// 测试函数
void test_algorithm(const char *name, Result* (*algo)(Item*, int, int),
                    Item *items, int n, int capacity, FILE *fp, FILE *excel_fp) {
    double start_time = get_time_ms();
    Result *res = algo(items, n, capacity);
    double end_time = get_time_ms();
    
    fprintf(fp, "\n=== %s ===\n", name);
    print_result(res, items, fp);
    fprintf(fp, "Execution time: %.2f ms\n", end_time - start_time);
    
    // 记录n=1000时的统计信息到Excel文件
    if (n == 1000 && capacity == 10000) {
        fprintf(excel_fp, "%s,%d,%.2f,%d,%.2f\n", name, res->count, res->total_value, res->total_weight, end_time - start_time);
    }
    
    free_result(res);
}

// 保存n=1000时物品信息到Excel
void save_items_to_excel(Item *items, int n, FILE *fp) {
    printf("Saving items for n=%d\n", n); // Debug
    if (n == 1000) {
        printf("Writing to items_n1000.csv\n"); // Debug
        fprintf(fp, "Item_ID,Weight_Kg,Value_Yuan\n");
        for (int i = 0; i < n; i++) {
            fprintf(fp, "%d,%d,%.2f\n", items[i].id, items[i].weight, items[i].value);
        }
        fflush(fp); // Force flush to ensure data is written
    } else {
        printf("Skipped writing to items_n1000.csv, n=%d\n", n); // Debug
    }
}

int main() {
    int item_counts[] = {10, 20, 30, 50, 100, 500, 1000, 2000, 5000, 10000};
    int capacities[] = {100, 1000, 10000};
    int num_items = sizeof(item_counts) / sizeof(item_counts[0]);
    int num_caps = sizeof(capacities) / sizeof(capacities[0]);
    
    FILE *output_fp = fopen("results.txt", "w");
    if (!output_fp) {
        perror("Failed to open results.txt");
        exit(EXIT_FAILURE);
    }
    FILE *excel_fp = fopen("stats_n1000.csv", "w");
    if (!excel_fp) {
        perror("Failed to open stats_n1000.csv");
        fclose(output_fp);
        exit(EXIT_FAILURE);
    }
    fprintf(excel_fp, "Algorithm,Selected Count,Total Value,Total Weight,Execution Time (ms)\n");
    
    for (int i = 0; i < num_items; i++) {
        int n = item_counts[i];
        printf("Processing n=%d\n", n); // Debug
        Item *items = (Item*)malloc(n * sizeof(Item));
        if (!items) {
            perror("malloc failed for items");
            fclose(output_fp);
            fclose(excel_fp);
            exit(EXIT_FAILURE);
        }
        init_items(items, n);
        
        // Save items only for n=1000
        if (n == 1000) {
            FILE *items_fp = fopen("items_n1000.csv", "w");
            if (!items_fp) {
                perror("Failed to open items_n1000.csv");
                free(items);
                fclose(output_fp);
                fclose(excel_fp);
                exit(EXIT_FAILURE);
            }
            save_items_to_excel(items, n, items_fp);
            fclose(items_fp);
        }
        
        for (int j = 0; j < num_caps; j++) {
            int capacity = capacities[j];
            fprintf(output_fp, "\nTesting with n=%d, capacity=%d\n", n, capacity);
            printf("\nTesting with n=%d, capacity=%d\n", n, capacity);
            
            if (n <= 20) {
                test_algorithm("Brute Force", brute_force, items, n, capacity, output_fp, excel_fp);
                test_algorithm("Backtracking", backtracking, items, n, capacity, output_fp, excel_fp);
            }
            test_algorithm("Dynamic Programming", dynamic_programming, items, n, capacity, output_fp, excel_fp);
            test_algorithm("Greedy", greedy, items, n, capacity, output_fp, excel_fp);
        }
        
        free(items);
    }
    
    fclose(output_fp);
    fclose(excel_fp);
    return 0;
}
# kevin.github.io
