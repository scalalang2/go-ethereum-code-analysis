## Overview

트라이 트리는 사전 트리, Word Search 트리 또는 prefix 트리 등으로 불린다. 예를 들어 영어 사전은 26개 종류로 포크가 발생하는 트리라고 생각할 수 있고 전화번호부는 0~10까지 종류로 포크가 발생하는 트리라고 볼 수 있다.

트라이는 말은 무엇을 **"찾다"** 라는 의미의 retrieve 단어를 발음할 때 트라이 라고 부르는데서 유래되었다고 한다.

트라이 트리는 공통 prefix를 사용해서 단어를 저장할 때 저장 공간을 절약하기 때문에 유용하게 쓰인다. 아래 그림을 보면 트라이 트리는 tea, ten, to, in, inn int 단어를 10개의 노드를 사용해서 저장한다.

![trie](picture/trietree.jpg)

이 트리에서 `[in, inn, int]`이 세 단어의 공통 prefix는 "in"이다. 그래서 저장소에는 "in"의 복사본 하나만 저장해서 공간을 아낀다. 반대로, 이런 트라이 구조의 단점은 문자열의 길이가 아주 길고 공통 prefix가 없다면 메모리 공간을 많이 먹는다는 점이다.

트라이 트리의 기본적인 속성은 다음과 같다.

1. 모든 단어들이 첫글자가 동일한 경우에만 루트 노드가 값을 가진다. 그 외 경우에는 루트 노드는 빈 노드이다.

1. 루트부터 시작해서 단말노드까지 edge에 저장된 character값이 더해진다.

1. 모든 리프 노드는 서로 다른 값을 저장하고 있다.

## Operations

트라이의 삽입, 삭제, 순회는 간단하다. 단순히 한 사이클만 돌면 되고 i 번째 반복은 첫 i 번째 단어에 해당하는 서브 트리를 찾는다. 이 트라이 구조를 메모리에 저장하려면 단순히 배열로 구현하거나 동적 포인터 타입을 저장하는 구조체를 하나 정의해서 구현할 수도 있다.

아래는 구조체를 정의해서 구현하는 방법을 C++ 코드로 보여준다.

```C
#define MAX_NUM 26
enum NODE_TYPE{ // "COMPLETED" 는 리프노드를 의미한다.
  COMPLETED,
  UNCOMPLETED
};
struct Node {
  enum NODE_TYPE type;
  char ch;
  struct Node* child[MAX_NUM]; //26-tree->a, b ,c, .....z
};

struct Node* ROOT; // 트리의 루트

struct Node* createNewNode(char ch){
  // 노드를 만든다.
  struct Node *new_node = (struct Node*)malloc(sizeof(struct Node));
  new_node->ch = ch;
  new_node->type == UNCOMPLETED;
  int i;
  for(i = 0; i < MAX_NUM; i++)
    new_node->child[i] = NULL;
  return new_node;
}

void initialization() {
    // 비어있는 루트를 만든다.
    ROOT = createNewNode(' ');
}

int charToindex(char ch) { // 'a'~'z'를 순서대로 0~25 사이로 변환
    return ch - 'a';
}

int find(const char chars[], int len) {
  struct Node* ptr = ROOT;
  int i = 0;

  while(i < len) {
     // i 에 해당하는 글자가 없다면 못 찾은 거니까 종료
     if(ptr->child[charToindex(chars[i])] == NULL) {
         break;
     }
     ptr = ptr->child[charToindex(chars[i])];
     i++;
  }

  return (i == len) && (ptr->type == COMPLETED);
}

void insert(const char chars[], int len) {
  struct Node* ptr = ROOT;
  int i;
  for(i = 0; i < len; i++) {
     // 비어있는 곳이 있다면 새로운 노드를 삽입한다.
     // 그 후 ptr이 가르키는 포인터를 변경한다.
     if(ptr->child[charToindex(chars[i])] == NULL) {
        ptr->child[charToindex(chars[i])] = createNewNode(chars[i]);
     }
     ptr = ptr->child[charToindex(chars[i])];
  }

  ptr->type = COMPLETED;
}
```

## 메모리 최적화 방안

메모리를 최적화 하고 싶다면, 배열 2개를 써서 구현할 수도 있다.

![double](picture/double-array-trie.jpg)

## 어디에 쓰면 좋을까?

트라이는 단순하지만 아주 효율적인 자료구조이다. 그래서 다양하게 활용할 수 있는 방안이 있다.

**(1) 단어 검색**

영어 사전을 구현한다고 생각해보자, 내가 검색하고자 하는 단어길이가 N이라면 정확히 `O(N)`의 수행만으로 결과를 찾을 수 있다. 단어 크기가 20자가 채 되지 않는다면 아무리 많은 단어를 저장해도 검색 속도가 느려지진 않는다.

**(2) 정렬**

트라이 트리 자체가 사전순으로 정렬된 모양이 된다. 그래서 사전 순으로 정렬해서 조회하는게 편하다.

**(3) 다른 자료구조나 알고리즘의 보조 구조체로 자주 쓰인다.**

예시, Suffix Tree

## 트라이의 복잡도

(1) `N`을 문자열의 길이라고 할 때, 삽입과 조회의 시간 복잡도는 `O(N)`이다.

(2) `26^N`의 공간 복잡도를 가진다. 

## 요약

트라이 트리는 매우 중요한 자료 구조인데다 정보 탐색, 믄자열 매칭 등 다양한 응용 분야에 사용된다. 그리고 많은 복잡한 자료 구조의 베이스가 되는 알고리즘이기도 한다. 따라서 좋은 개발자는 트라이 구조를 마스터할 필요가 있다. 실제로 백준에 자주 보이는 문제이기도 하다.