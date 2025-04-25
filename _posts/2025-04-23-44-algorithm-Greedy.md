---
layout: post
title: "알고리즘 - Greedy 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
hidden: true
---

<details>
<summary>2+1 세일</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/11508)

<br>
### 난이도 : ⭐
최대한 비싼 항목이 제외되야하기 때문에 내림 차순 정렬 후 3개씩 홧인


### 코드
```java
import java.util.*;
import java.io.*;

// DFS, O(N^2)
public class Main{
    public static void main(String[] args) throws IOException{
        // 1. 입력 및 초기화
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        int n = Integer.parseInt(stz.nextToken());
        Integer []arr = new Integer[n];
        for(int i=0; i<n; i++){
            stz = new StringTokenizer(br.readLine());
            arr[i] = Integer.parseInt(stz.nextToken());
        }
        Arrays.sort(arr,Collections.reverseOrder());

        int idx = 1;
        int answer = 0;
        for(int num: arr){
            if(idx % 3 == 0) {
                idx++;
                continue;
            }
            answer += num;
            idx++;
        }
        System.out.println(answer);
    }
}
```
</div>
</details>
