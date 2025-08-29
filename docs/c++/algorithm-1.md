---
layout: default
title: C++ 알고리즘 기초
parent: C++
nav_order: 1
---

## C++ 알고리즘 기초 정리

### 1. 알고리즘이란?
> 문제를 해결하기 위한 절차 또는 방법

### 2. 기본 알고리즘 종류

| 분류       | 알고리즘 예시                            | 설명                         |
|------------|------------------------------------------|------------------------------|
| 정렬       | 선택, 버블, 삽입, 퀵, 병합, 힙           | 데이터를 순서대로 정렬       |
| 탐색       | 선형 탐색, 이진 탐색                     | 원하는 데이터를 찾는 방법    |
| 재귀       | 팩토리얼, 피보나치                       | 함수가 자기 자신을 호출     |
| 브루트포스 | 순열, 조합                               | 모든 경우를 시도             |
| 수학       | 소수 판별, 최대공약수(GCD), 최소공배수(LCM) | 수학적 문제 해결 알고리즘    |


### 3.  정렬 예제 (버블 정렬)
```cpp
void bubbleSort(vector<int>& arr) {
    int n = arr.size();
    for (int i = 0; i < n-1; i++) {
        for (int j = 0; j < n-1-i; j++) {
            if (arr[j] > arr[j+1]) {
                swap(arr[j], arr[j+1]);
            }
        }
    }
}
```
---
### 4.  탐색 예제 (이진 탐색)
```cpp
int binarySearch(vector<int>& arr, int target) {
    int left = 0, right = arr.size() - 1;
    while (left <= right) {
        int mid = (left + right) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```
---
### 5.  수학 알고리즘 예제 (최대공약수 GCD)
```cpp
int gcd(int a, int b) {
    if (b == 0) return a;
    return gcd(b, a % b);
}
```
* gcd(20,8) -> 4
* C++17 이상에서는 std::gcd(a, b) 사용 가능

### 6.  재귀 알고리즘 예제 (팩토리얼)
```cpp
int factorial(int n) {
    if (n == 0) return 1;
    return n * factorial(n - 1);
}
```

### 7.  브루트포스 예제 (모든 조합 출력)
```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> v = {1, 2, 3};
    do {
        for (int x : v) std::cout << x << ' ';
        std::cout << '\n';
    } while (std::next_permutation(v.begin(), v.end()));
}
```

 추천 학습 순서
1.  C++ STL 기본 (vector, pair, sort 등)

2.  정렬 알고리즘 직접 구현해보기

3.  탐색 알고리즘 (선형, 이진 탐색)

4.  수학적 알고리즘 (GCD, 소수)

5.  재귀와 브루트포스

6.  문제로 실전 연습 시작 (백준, LeetCode 등)

