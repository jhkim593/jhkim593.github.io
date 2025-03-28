---
layout: post
title: "알고리즘 - 투포인터 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
hidden : true
---

<details>
<summary><strong>부분합</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1806)

<br>
### 난이도 : ⭐⭐


### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    public static void main(String [] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());
        int k = Integer.parseInt(stz.nextToken());

        stz = new StringTokenizer(br.readLine());
        int [] arr = new int[n];
        for(int i=0; i<n; i++){
            arr[i] = Integer.parseInt(stz.nextToken());
        }

        int start = 0;
        int end = 0;
        int sum = arr[0];
        int min = Integer.MAX_VALUE;
        while(true) {
            if(sum < k) {
                end++;
                if(end == arr.length) break;
                sum += arr[end];
            } else {
                min = Math.min(min, end-start+1);
                sum -= arr[start];
                start++;
            }
        }
        System.out.println(min==Integer.MAX_VALUE ? 0 : min);
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>용액</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/2470)

<br>
### 난이도 : ⭐
배열 첫번째 인덱스 , 마지막 인덱스 투포인터를 이용해서 용액 합을 계산함

### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    public static void main(String [] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());

        stz = new StringTokenizer(br.readLine());
        int [] arr = new int[n];
        for(int i=0; i<n; i++){
            arr[i] = Integer.parseInt(stz.nextToken());
        }

        int min = Integer.MAX_VALUE;
        int start = 0;
        int end = n-1;

        int saveStart =0;
        int saveEnd =0;
        while(start != end){
            int num = arr[start] + arr[end];
            if(min > Math.abs(num)){
                saveStart = start;
                saveEnd = end;
                min = Math.abs(num);
            }
            if(num > 0) {
                end--;
            } else if (num < 0){
                start++;
            } else break;
        }
        System.out.print(arr[saveStart] + " "+ arr[saveEnd]);
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>소수의 연속합</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1644)

<br>
### 난이도 : ⭐⭐

`for (int j = 2; j * j <= i; j++)` 를 통해 소수를 구함

### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    public static void main(String [] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());
        int []arr = new int[4000000];

        int index = 0;
        loop:
        for(int i=2; i<=n; i++) {
            for (int j = 2; j * j <= i; j++) {
                if (i % j == 0) continue loop;
            }
            arr[index++] = i;
        }

        int start =0;
        int end = 0;
        int count =0;
        int sum=0;
        while(true){
            if(sum <= n){
                if(sum == n) count++;

                if(arr[end] == 0){
                    break;
                }
                sum+=arr[end++];
            } else {
                sum-=arr[start++];
            }
        }
        System.out.println(count);
    }
}
```
</div>
</details>
