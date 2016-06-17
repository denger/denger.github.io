---
layout: post
title: 使用 C 实现 Java LinkedList
---

以下基于 ADT 链表数据结构实现的C语言版 LinkedList。


{% highlight c %}
/** 
 *  link_list.h 
 *  
 *  Generic sequential link list node structure, the can hold any type data. 
 *   
 *  @author: Denger 
 *  Thu May 13, 21:50:25 
 */  
typedef struct _link_list{  
    int theSize;  
    int modCount;  
    struct _node *node;  
    struct _node *beginMarker;  
    struct _node *endMarker;  
} link_list;  
  
struct _node{  
    void *data;  
    struct _node *prev;  
    struct _node *next;  
};  
  
extern link_list* create_link_list(void);  
  
extern void link_add(link_list *link, void *data);  
  
extern void* link_get(link_list *link, int index);  
  
extern void link_set(link_list *link, int index, void *data);  
  
extern int link_size(link_list *link);  
  
extern void link_remove(link_list *link, int index);  
  
extern void link_destory(link_list *link);  
{% endhighlight c %}

针对以上 link_list.h 的实现：

{% highlight c %}
// link_list.c  
// author denger  
#include "../../include/common.h"  
#include "../../include/link_list.h"  
  
#define MALLOC_ERROR "Failed to allocate menory space!"  
  
/** 
  * Create a link list. 
  */  
link_list* create_link_list(void){  
    link_list *list = (link_list*)malloc(sizeof(link_list));  
    if (list == NULL){  
        puts(MALLOC_ERROR);  
        return NULL;  
    }  
    list->theSize = 0;  
    list->node = (struct _node*)malloc(sizeof(struct _node));  
    if (list->node != NULL){  
        list->node->data = NULL;  
        list->node->next = list->node->prev = NULL; // Initialize pointer NULL  
        list->endMarker = list->beginMarker = list->node;  
    }else{  
        puts(MALLOC_ERROR);  
        return NULL;  
    }  
    return list;  
}  
  
// Add any type to link_list.  
void link_add(link_list *list, void *data){  
    struct _node* node = (struct _node*)malloc(sizeof(struct _node));  
    if (node == NULL){  
        puts(MALLOC_ERROR);  
    }else{  
        node->prev = list->node;  
        list->endMarker = list->node = list->node->next = node;  
        node->data = data;  
        list->theSize++;  
    }  
}  
  
  
// Getter a node in link_list.  
struct _node* _link_get_node(link_list *list, int index){  
    if (index >= list->theSize || index < 0){  
        printf("List index out of bounds : %d\n", list->theSize);  
        return NULL;  
    }  
    int i;  
    struct _node *node;   
    if (index < link_size(list) / 2){  
        node = list->beginMarker->next;  
        for (i = 0; i < index; i++){  
            node = node->next;  
        }  
    }else{  
        node = list->endMarker;    
        int max = link_size(list) - index - 1;  
        for (i = 0; i < max; i++){  
            node = node->prev;  
        }  
    }  
    return node;  
}  
  
void* link_get(link_list *list, int index){  
    struct _node *node = _link_get_node(list, index);  
    if (node == NULL){  
        return NULL;  
    }  
    return node->data;  
}  
  
// return the link list size.  
int link_size(link_list *list){  
    return list->theSize;  
}  
  
// replace node.  
void link_set(link_list *list, int index, void *data){  
    struct _node *node = _link_get_node(list, index);  
    if (node != NULL){  
        node->data = data;  
    }  
}  
  
// remove node.  
void link_remove(link_list *list, int index){  
    struct _node *node = _link_get_node(list, index);   
    if (node == NULL){  
        return;  
    }  
    node->prev->next = node->next;  
    node->next->prev = node->prev;  
    if (node != NULL){  
        if (node->data != NULL){  
            free(node->data);  
        }  
        free(node);  
        node = NULL;  
    }  
    list->theSize--;  
}  
  
// destory link  
void link_destory(link_list *list){  
    int i = 0;  
    struct _node *node = list->beginMarker;  
    for (i = 0; i <= list->theSize; i++){  
        struct _node *next = node->next;  
        if (node != NULL){  
            if (node->data != NULL){  
            free(node->data);  
            }  
            free(node);  
            node = next;  
        }  
    }  
    free(list);  
    list = NULL;  
}  
{% endhighlight c %}

简单测试代码示例：
{% highlight c %}
int main(int argc, char *args[]) {  
    link_list *list  = create_link_list();  
    int i;   
    /* 
    int a = 100, b = 1000, c = 10000; 
    link_add(list, &a); 
    link_add(list, &b); 
    link_add(list, &c); 
 
    printf("\nIndex 0 value: %d\n", *(int*)link_get(list, 0)); 
    printf("\nIndex 1 value: %d\n", *(int*)link_get(list, 1)); 
    printf("\nIndex 2 value: %d\n", *(int*)link_get(list, 2)); 
     
    printf("\nIndex 1 value: %d\n", *(int*)link_get(list, 1)); 
    printf("\nIndex 0 value: %d\n", *(int*)link_get(list, 0)); 
    printf("\nIndex 2 value: %d\n", *(int*)link_get(list, 2)); 
    */   
    for (i = 0; i < 10; i++){  
    int* value = (int*)malloc(sizeof(int));  
    *value = i;  
    link_add(list, value);  
    }  
    printf("The list size: %d\n", link_size(list));  
    for (i = 0; i < link_size(list); i++){  
    printf("index: %d = %d\n", i, *(int*)link_get(list, i));  
    }  
    int* new = (int*)malloc(sizeof(int));   
    *new = 100;  
    link_set(list, 0, new);  
    printf("\n Update the index 0 value: %d\n", *(int*)link_get(list, 0));  
    link_remove(list, 0);  
      
    printf("Remove Before========\n");  
    for (i = 0; i < link_size(list); i++){  
    printf("index: %d = %d\n", i, *(int*)link_get(list, i));  
    }  
    link_destory(list);  
}  
{% endhighlight c %}
