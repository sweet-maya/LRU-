# LRU-
## 原理
LRU（Least recently used，最近最少使用）算法根据数据的历史访问记录来进行淘汰数据，其核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”。
最常见的实现是通过哈希表辅助双向链表实现，使用哈希表来保存缓存数据，并用双向链表保存数据的淘汰顺序，数据的淘汰顺序满足规则：新数据插入到链表头部；每当缓存命中（即缓存数据被访问），则将数据移到链表头部；当链表满的时候，将链表尾部的数据丢弃。插入数据和访问数据的时间复杂度均为O(1)。
## 设计思路
使用hashmap存储key，而hashmap的value指向双向链表实现的lru的node节点。首先预设置lru的容量，如果存储满了，淘汰双向链表的头部节点，每次新增和访问数据，都将新的节点增加到队头，或者把已存在的节点移动到队头，这样每次存储或访问节点对应的时间复杂度均为o（1）。
## 代码实现
### 双向链表节点ListNode
成员：
    key: i32,
    val: i32,
    prev: Option<Rc<RefCell<ListNode>>>,
    next: Option<Rc<RefCell<ListNode>>>,
    
    
方法：
impl ListNode {
    fn new(key: i32, val: i32) -> Self {
        Self {
            key,
            val,
            prev: None,
            next: None,
        }
    }
}

### 双向链表List
struct List {
    head: Option<Rc<RefCell<ListNode>>>,
    tail: Option<Rc<RefCell<ListNode>>>,
}


链表节点中的prev，next以及双向链表中的head tail的类型均为option（rc（refcell（listnode）））类型

   
   以next为例进行说明，首先，next可能为none或具体值，所以是option类型，其次，由于next是对其它节点的引用，所以没有对应节点的所有权，
   采用rc共享所有权，另一方面，因为双向链表节点的变动会导致next字段的变动，但是rc是不可变的，所以需要refcell将其变为可变的。
   
   
   
   fn new() -> Self {
        Self {
            head: None,
            tail: None,
        }
    }
    
    
    fn push_back(&mut self, node: Option<Rc<RefCell<ListNode>>>) {
        if let Some(t) = self.tail.take() {
            if let Some(n) = &node {
                t.borrow_mut().next = Some(n.clone());
                n.borrow_mut().prev = Some(t);
            }
        } else {
            self.head = node.clone();
        }

        self.tail = node;
    }
    
    
    
    push back接口用于实现将新的接点插入到双向链表尾部，在这一步过程中需要判断双向链表是否有节点，也即是判断tail是否为none
    
    使用take转移走self.tail 的内部值并进行判断，由于过程中需要改变next和prev的值，所以需要调用borrow_mut（）来使内部值可变
    
    另外，由于插入节点被tail以及head或前一节点的next同时引用，所以需要clone来增加引用计数。
    
    
    fn pop_front(&mut self) -> Option<Rc<RefCell<ListNode>>> {
        if let Some(h) = self.head.take() {
            if let Some(n) = h.borrow_mut().next.take() {
                n.borrow_mut().prev = None;
                self.head = Some(n);
            } else {
                self.head = None;
                self.tail = None;
            }

            Some(h)
        } else {
            None
        }
    }
    
    
    fn unlink_node(&mut self, node: Option<Rc<RefCell<ListNode>>>) -> Option<Rc<RefCell<ListNode>>> {
        if let Some(n) = node {
            let prev = n.borrow_mut().prev.take();
            let next = n.borrow_mut().next.take();

            if let Some(p) = &prev {
                p.borrow_mut().next = next.clone();
            } else {
                self.head = next.clone();
            }

            if let Some(n) = &next {
                n.borrow_mut().prev = prev;
            } else {
                self.tail = prev;
            }

            Some(n)
        } else {
            None
        }
    }
}



 pop front 用于取出双向链表头部节点并返回，unlink node用于取出双向链表内部节点，并将该节点的前节点和后节点相连
 
 
 ### LRUCache
 
 
 struct LRUCache {
    cap: usize,
    used: usize,
    data: HashMap<i32, Rc<RefCell<ListNode>>>,
    list: List,
}


fn get(&mut self, key: i32) -> i32 {
        if let Some(node) = self.data.get(&key) {
            let val = node.borrow().val;
            
            let node = self.list.unlink_node(Some(node.clone()));
            self.list.push_back(node);

            val
        } else {
            -1
        }
    }
    
    
    fn put(&mut self, key: i32, value: i32) {
        if let Some(node) = self.data.get_mut(&key) {
            node.borrow_mut().val = value;

            let node = self.list.unlink_node(Some(node.clone()));
            self.list.push_back(node);
        } else {
            if self.used == self.cap {
                if let Some(node) = self.list.pop_front() {
                    self.data.remove(&node.borrow().key);
                    self.used -= 1;
                }
            }

            let new_node = Rc::new(RefCell::new(ListNode::new(key, value)));
            self.data.insert(key, new_node.clone());
            self.list.push_back(Some(new_node));
            self.used += 1;
        }
    }
}


get 用于根据key值访问节点，若节点不存在则返回-1
put 用于存储数据，若该过程使得容量溢出，则会涉及数据的淘汰。这两个方法均会涉及到hashmap以及双向链表节点的变化
值得一提的是，在调用双向链表的方法时 传递的参数是clone，这是因为在方法调用完成后，会使参数的引用计数减一，因此需要传递clone来增加引用计数，除此之外，在所有权和生命周期等特性的处理上，与之前并无二异。

 
 ## 改进
 ### 缺点
 偶发性的、周期性的批量操作会导致LRU命中率急剧下降，缓存污染情况比较严重。


### 对策
将最近使用过1次”的判断标准扩展为“最近使用过K次。
维护一个访问历史列表，当数据第一次被访问，加入到访问历史列表；如果数据在访问历史列表里后没有达到K次访问，则按照一定规则（FIFO，LRU）淘汰；当访问历史队列中的数据访问次数达到K次后，将数据索引从历史队列删除，将数据移到缓存队列中，并缓存此数据，缓存队列重新按照时间排序；缓存数据队列中被再次访问后，重新排序；需要淘汰数据时，淘汰缓存队列中排在末尾的数据，即：淘汰“倒数第K次访问离现在最久”的数据。



