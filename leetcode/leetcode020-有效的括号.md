# 有效的括号

## 问题描述

给定一个只包括 '('，')'，'{'，'}'，'['，']' 的字符串，判断字符串是否有效。

有效字符串需满足：

左括号必须用相同类型的右括号闭合。
左括号必须以正确的顺序闭合。
注意空字符串可被认为是有效字符串。

示例 1:

``` text
输入: "()"
输出: true
```

示例 2:

``` text
输入: "()[]{}"
输出: true
```

示例 3:

``` text
输入: "(]"
输出: false
```

示例 4:

``` text
输入: "([)]"
输出: false
```

示例 5:

``` text
输入: "{[]}"
输出: true
````

## 解答

思路：遍历字符串，每遇到一个左括号，则压入栈中，栈中存储的是未匹配的左括号，当遇到一个右括号时，只有栈顶的左括号跟其类型一致才能匹配成功，如果匹配成功弹出栈顶，继续遍历字符串。最后栈为空时说明字符串是有效的。

``` java
public static boolean solution(String s) {
    HashMap<Character, Character> map = new HashMap<>();
    map.put(')', '(');
    map.put(']', '[');
    map.put('}', '{');
    Stack<Character> stack = new Stack<>();
    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        if (map.containsKey(c)) {
            char top = stack.empty() ? '*' : stack.pop();
            if (top != map.get(c)) {
                return false;
            }
        } else {
            stack.push(c);
        }
    }
    return stack.empty();
}
```

下面的方法和上面的思路是一致的，不过做了几项简化：使用更简单的数据结构

* 去除HashMap的使用，括号的类型有限，直接使用多个if即可。
* 用数组的索引移动来模拟栈，java中stack的实现是链表，预申请内存空间的数组比一直需要创建节点对象的链表效率要高。而且数组是基本类型减少了容器对基本类型的包装和拆箱消耗。

``` java
public static boolean solution(String s) {
    int S_len=s.length();
    if(S_len==0) return true;
    char[] stack=new char[S_len];
    int top=0;
    stack[top++]=s.charAt(0);
    for (int i=1;i<S_len;++i){
        char x=s.charAt(i);
        if (x==')'){
            if (top==0)return false;
            if (stack[top-1]=='(') {
                top--;
                continue;
            }
            return false;
        }
        if (x==']'){
            if (top==0)return false;
            if ( stack[top-1]=='[') {
                top--;
                continue;
            }
            return false;
        }
        if (x=='}'){
            if (top==0)return false;
            if ( stack[top-1]=='{'){
                top--;
                continue;
            }
            return false;
        }
        stack[top++]=x;
    }
    return top == 0;
}
```
