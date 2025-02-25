---
title: COW-Trie 多线程正确性测试构造心得
date: 2025-02-22 11:44:56
categories:
  - [C++, 数据结构]
  - [多线程]
  - [助教工作]
tags:
  - C++
  - 助教工作
  - 多线程
---

{% note::实践是检验真理的唯一标准 %}

本学期继续担任 JohnClass 程设的助教，这次分配的任务是 copy-on-write Trie with Concurrency 以及 deque。

本篇着重讨论一下前者。

<!-- more -->

### About

首先我们给出 github 仓库：

{%link COW-Trie with concurrency::https://github.com/No8irCh6En/Cow-Trie-Concurrency-included::https://pic1.imgdb.cn/item/67aabccdd0e0a243d4fe3cda.jpg %}

这个大作业的想法来源是 CMU15445 Database Spring 2023 Project 0 'Cpp Primer'

对于如何理解 COW-Trie 这种数据结构，可以看我们 github 仓库中的 README.md，介绍的应该较为详尽了：

{% folding README.md %}

### Copy-on-Write Trie with Concurrency

> Acknowledgement: CMU15445 Database Spring 2023 Project 0 'Cpp Primer'

In this project, you will implement a key-value store backed by a copy-on-write trie. Tries are efficient ordered-tree data structures for retrieving a value for a given key. To simplify the explanation, we will assume that the keys are variable-length strings, but in practice they can be any arbitrary type.

Each node in a trie can have multiple child nodes representing different possible next characters.

The key-value store you will implement can store string keys mapped to values of any type. The value of a key is stored in the node representing the last character of that key (aka terminal node). For example, consider inserting kv pairs ("ab", 1) and ("ac", "val") into the trie.

<img src="https://notes.sjtu.edu.cn/uploads/upload_b6e9036083fbe8202e19f9d8535e1d81.png" width="300">


The two keys share the same parent node. The value 1 corresponding to key "ab" is stored in the left child, and the value "val" corresponding to key "ac" is stored in the right node.

#### Task1 : Copy-On-Write Trie

In this task, you will need to modify trie.h and trie.cpp to implement a copy-on-write trie. In a copy-on-write trie, operations do not directly modify the nodes of the original trie. Instead, new nodes are created for modified data, and a new root is returned for the newly-modified trie. Copy-on-write enables us to access the trie after each operation at any time with minimum overhead. Consider inserting ("ad", 2) in the above example. We create a new Node2 by reusing two of the child nodes from the original tree, and creating a new value node 2. (See figure below)

<img src="https://notes.sjtu.edu.cn/uploads/upload_a6b0c1197d5b95610b755fc62b51d66b.png" width="500">


If we then insert ("b", 3), we will create a new root, a new node and reuse the previous nodes. In this way, we can get the content of the trie before and after each insertion operation. As long as we have the root object (Trie class), we can access the data inside the trie at that time. (See figure below)

<img src="https://notes.sjtu.edu.cn/uploads/upload_4a69483f918f61563e53641c1e573ac6.png" width="500">



One more example: if we then insert ("a", "abc") and remove ("ab", 1), we can get the below trie. Note that parent nodes can have values, and you will need to purge all unnecessary nodes after removal.

<img src="https://notes.sjtu.edu.cn/uploads/upload_a8b9e344d16516dbcff8586e5db0e255.png" width="400">


Your trie should support 3 operations:

1. Get(key): Get the value corresponding to the key.
2. Put(key, value): Set the corresponding value to the key. Overwrite existing value if the key already exists. Note that the type of the value might be non-copyable (i.e., `std::unique_ptr<int>`). This method returns a new trie.
3. Delete(key): Delete the value for the key. This method returns a new trie.
None of these operations should be performed directly on the trie itself. You should create new trie nodes and reuse existing ones as much as possible.
4. `operator==`: Actually it is not necessary. If you don't need this, just comment out.

To create a new node, you should use the Clone function on the TrieNode class. To reuse an existing node in the new trie, you can copy `std::shared_ptr<TrieNode>`: copying a shared pointer doesn’t copy the underlying data. You should not manually allocate memory by using new and delete in this project. `std::shared_ptr` will deallocate the object when no one has a reference to the underlying object.

For the full specifications of these operations, please refer to the comments in the starter code.
    

#### Task2 : Concurrent Key-Value Store

After you have a copy-on-write trie which can be used in a single-thread environment, implement a concurrent key-value store for a multithreaded environment. In this task, you will need to modify trie/src.hpp. This key-value store also supports 3 operations:

1. Get(key) returns the value.
2. Put(key, value). No return value.
3. Delete(key). No return value.

For the original Trie class, everytime we modify the trie, we need to get the new root to access the new content. But for the concurrent key-value store, the put and delete methods do not have a return value. This requires you to use concurrency primitives to synchronize reads and writes so that no data is lost through the process.

Your concurrent key-value store should concurrently serve multiple readers and a single writer. That is to say, when someone is modifying the trie, reads can still be performed on the old root. When someone is reading, writes can still be performed without waiting for reads.

Also, if we get a reference to a value from the trie, we should be able to access it no matter how we modify the trie. The Get function from Trie only returns a pointer. If the trie node storing this value has been removed, the pointer will be dangling. Therefore, in TrieStore, we return a ValueGuard which stores both a reference to the value and the TrieNode corresponding to the root of the trie structure, so that the value can be accessed as we store the ValueGuard.

To achieve this, we have provided you with the pseudo code for `TrieStore::Get` in src.hpp. Please read it carefully and think of how to implement `TrieStore::Put` and `TrieStore::Remove`.


#### Advice on Concurrency :

1. `std::vector` does not provide built-in thread-safety mechanisms.
2. The granularity of the lock should be carefully considered to prevent the program from effectively becoming single-threaded.


{% endfolding %}

### 复盘错误代码

本篇着重讨论一下加强数据点的过程。

为了防止

在审查自己的代码和同学以前的代码的过程中，我发现：

#### Case 1

`test/trie_store_test3.cpp` 可以有效 TLE 掉将多线程的锁的机制写成本质是单线程的程序。例如：

{% folding FalseCase1.cpp %}

```c++
class TrieStore {
public:
template <class T>
auto Get(std::string_view key, size_t version = -1)
-> std::optional<ValueGuard<T>> {
write_lock_.lock_shared();
size_t _version =
(version == (size_t)-1) ? snapshots_.size() - 1 : version;
Trie _object_trie = snapshots_[_version];
const T* _value_ptr = _object_trie.Get<T>(key);
write_lock_.unlock_shared();
if (_value_ptr)
return ValueGuard<T>(_object_trie, *_value_ptr);
return std::nullopt;
}

template <class T>
size_t Put(std::string_view key, T value) {
write_lock_.lock();
Trie _new_trie = snapshots_.back().Put<T>(key, std::move(value));
snapshots_.push_back(_new_trie);
size_t _size = snapshots_.size() - 1;
write_lock_.unlock();
return _size;
}

size_t Remove(std::string_view key) {
write_lock_.lock();
Trie _new_trie = snapshots_.back().Remove(key);
if (_new_trie == snapshots_.back()) {
write_lock_.unlock();
return snapshots_.size();
}
snapshots_.push_back(_new_trie);
size_t _size = snapshots_.size() - 1;
write_lock_.unlock();
return _size;
}

size_t get_version() {
write_lock_.lock_shared();
int _size = snapshots_.size() - 1;
write_lock_.unlock_shared();
return _size;
}

private:
std::shared_mutex write_lock_;
std::vector<Trie> snapshots_{1};
};
```

{% endfolding %}

毕竟该测试点是多线程测试，单线程能在固定时间内通过反而不对劲。

#### Case 2

但是在我们修改过的接口中，我发现，`std::vector` 容器本身并没有提供线程安全机制。

也就是说，关于对 `std::vector` 进行操作的部分，我们应当用锁进行保护。

没有正确保护 `std::vector` 的一个错误案例如下：

{% folding FalseCase2.cpp %}

```c++
template <class T>
auto TrieStore::Get(std::string_view key, size_t version = -1)
    -> std::optional<ValueGuard<T>> {
    size_t _version = (version == (size_t)-1) ? snapshots_.size() - 1 : version;
    Trie _object_trie = snapshots_[_version];
    const T* _value_ptr = _object_trie.Get<T>(key);
    if (_value_ptr)
        return ValueGuard<T>(_object_trie, *_value_ptr);
    return std::nullopt;
}


template <class T>
size_t TrieStore::Put(std::string_view key, T value) {
    write_lock_.lock();
    Trie _new_trie = snapshots_.back().Put<T>(key, std::move(value));
    snapshots_.push_back(_new_trie);
    size_t _size = snapshots_.size() - 1;
    write_lock_.unlock();
    return _size;
}

size_t TrieStore::Remove(std::string_view key) {
    write_lock_.lock();
    Trie _new_trie = snapshots_.back().Remove(key);
    if (_new_trie == snapshots_.back()) {
        write_lock_.unlock();
        return snapshots_.size();
    }
    snapshots_.push_back(_new_trie);
    size_t _size = snapshots_.size() - 1;
    write_lock_.unlock();
    return _size;
}

size_t TrieStore::get_version() {
    return snapshots_.size() - 1;
}
```

{% endfolding %}

这到底引发的是什么问题呢？

从底层的汇编代码的角度思考可能会更加容易一些：

我们考虑每次 `Get` 中遇到 `std::vector` 时发生的事情，至少要做如下事情：

```asm
leaq %rax (snapshots_,%rsi,8) # snapshots_ 实际地址由链接器填充
mov %rbx, (%rax) # %rbx 现存的就是指定版本的 root
```

如果我们在这两句之间切换了线程，调用了 `Put` 函数。

在大部分时间下不会有影响，除了当恰好 `std::vector` 容器倍增的时候。

当倍增完成之后再切换回 `Get` 函数在的线程时， `%rax` 已经不能代表 `snapshots_[n]` 所对应的地址了。

这会导致不符合预期的行为发生。

### 构造过程

其实想清楚锁为什么有问题之后，构造的方法非常的 plain，就是开很多个线程。

然后一些线程不断地 `Put`，一些线程不断地 `Get`。尽可能卡到出现上述问题就好。

如何判断出的问题确实是 `std::vector` 的呢？

我在自己的 "trie/src.hpp" 中加入这句话：

```c++
template <class T>
auto Trie::Get(std::string_view key) const -> const T* {
    if (!root_) {
        std::cerr << "Why is it here(nullptr)?" << std::endl;
        return nullptr;
    }
    ...
}
```

然后在运行下面的测试程序中，确实跑出了很多 Why is it here(nullptr)?

```c++
template <class TrieStore>
void ContinuePut(TrieStore& store) {
    auto start = std::chrono::high_resolution_clock::now();
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = duration_cast<std::chrono::seconds>(end - start);
    while (duration < std::chrono::seconds(200)) {
        store.Put(std::to_string(RVERSION), 86);
        RVERSION++;
        end = std::chrono::high_resolution_clock::now();
        duration = duration_cast<std::chrono::seconds>(end - start);
    }
}

template <class TrieStore>
void ContinueGet(TrieStore& store, int idx) {
    auto start = std::chrono::high_resolution_clock::now();
    std::this_thread::sleep_for(std::chrono::nanoseconds(400 * idx));
    bool not_found = false;
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = duration_cast<std::chrono::seconds>(end - start);
    std::atomic<int> inserted(0);
    while (inserted <= 100) {
        if (store.template Get<int>(std::to_string(inserted.load())) !=
            std::nullopt) {
            inserted++;
        }
    }
    while (duration < std::chrono::seconds(200)) {
        for (int i = 1; i <= 100; i++) {
            not_found |=
                (store.template Get<int>(std::to_string(i)) == std::nullopt);
        }
        if (not_found) {
            std::cerr << "Error: Can't find existed key." << std::endl;
            exit(0);
        }
        end = std::chrono::high_resolution_clock::now();
        duration = duration_cast<std::chrono::seconds>(end - start);
    }
}
```

从而验证了结果。

到这里都是让人雀跃的，但是我在给别的助教验证的时候，得到了很不好的消息。

### 运行时间的严重差异

在别的助教（我此处称为 F）的电脑上运行相对应的测试程序，发现运行时间和我差别很大。

我将停止时间设置为 200s，在自己的电脑上使用 wsl 运行所花费的时间在 300-400s 基本就能结束。

虽然和 200s 有一些误差，但是我认为常数在 2 以内所以就糊弄过去了。

但是 F 的电脑上相同环境运行却在 1000s 内可能都无法跑完。

这个时间点我是很沮丧的。

不过庆幸的是，我们花了 2 个小时成功找到了一些蛛丝马迹，我们查看了程序运行过程中的 TID

有趣的是在我的计算机上，所有线程的 TID 都在很短的时间内出现过。

但是在 F 的电脑上，长时间内都只出现了 16 个线程的 TID。

我们赶紧检查输入输出流，发现正是上面的 `std::cerr << "..." << std::endl;` 引发的差异

突然想到在 ics 课上老师说过输入输出也会进内核，从而导致线程切换。

于是差异性问题得用解决。

### 如何优化

我们是不能指望小朋友闲着没事在自己的函数里加一句向 `std::cerr` 输出的。

因此我们采取了一个比输出更优的办法：

利用系统内部的线程调度让 `std::this_thread::sleep_for` 起到强行切换线程的作用。如下：

```c++
std::atomic<int> RVERSION;  // real_version, std::atomic guarantees

template <class TrieStore>
void ContinuePut(TrieStore& store) {
    auto start = std::chrono::high_resolution_clock::now();
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = duration_cast<std::chrono::seconds>(end - start);
    while (duration < std::chrono::seconds(200)) {
        store.Put(std::to_string(RVERSION), 86);
        RVERSION++;
        end = std::chrono::high_resolution_clock::now();
        duration = duration_cast<std::chrono::seconds>(end - start);
    }
}

template <class TrieStore>
void ContinueGet(TrieStore& store, int idx) {
    auto start = std::chrono::high_resolution_clock::now();
    std::this_thread::sleep_for(std::chrono::nanoseconds(400 * idx));
    bool not_found = false;
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = duration_cast<std::chrono::seconds>(end - start);
    std::atomic<int> inserted(0);
    while (inserted <= 100) {
        if (store.template Get<int>(std::to_string(inserted.load())) !=
            std::nullopt) {
            inserted++;
        }
        std::this_thread::sleep_for(std::chrono::nanoseconds(50));
        // switch the context
    }
    while (duration < std::chrono::seconds(200)) {
        for (int i = 1; i <= 100; i++) {
            not_found |=
                (store.template Get<int>(std::to_string(i)) == std::nullopt);
        }
        if (not_found) {
            std::cerr << "Error: Can't find existed key." << std::endl;
            exit(0);
        }
        end = std::chrono::high_resolution_clock::now();
        duration = duration_cast<std::chrono::seconds>(end - start);
        std::this_thread::sleep_for(std::chrono::nanoseconds(50));
        // switch the context
    }
}
```

现在就没有问题了！完整代码可以见 "test/trie_store_correctness_test.cpp"

~~后来我发现这个代码太暴力了以至于 OJ 系统会报 RE（使用内存超过4G）~~
