
---
title: My (poor) attempt at a mpmcqueue
draft: false
tags:
  - cpp
  - lockfree
---
 I wouldn't necessarily trust this implementation. This is lock-free(ish). it's lightly tested and just implemented for fun. The key insight for this sort of queue is a two-step approach in order to **own** a slot: see a slot available and successfully grab it.

If I want to push an element as a producer I have to do the following:
1. see an empty slot
2. successfully reserve the slot
3. commit the write

The argument follows similarly for a consumer.

Extending naturally from a single producer single consumer, we have an atomic index for the write and read. However, unlike an spscqueue, the write and read indexes aren't sufficient for a complete algorithm. In a spscqueue, you use the atomic increment of the write index to commit a `push` for a consumer to see; similarly for a consumer, you use the atomic increment of the read index to commit a `pop` for a producer to see.

This doesn't work for mpmc because you have to synchronize grabbing a slot AND changing the state of a slot.

```cpp
#pragma once

#include <atomic>
#include <cassert>
#include <immintrin.h>

using namespace std;

namespace ab
{
template<typename T>
class mpmcqueue
{
 private:
  static auto constexpr kCacheLineSize = 64;

  size_t _buffer_size{};

  struct alignas(kCacheLineSize) Slot
  {
    T Data{};
    // NOTE: Seq has 4 states: WriteAvailable, PendingWrite, ReadAvailable, PendingRead
    // Seq % 4 == 0 := WriteAvailable
    // Seq % 4 == 1 := PendingWrite
    // Seq % 4 == 2 := ReadAvailable
    // Seq % 4 == 3 := PendingRead
    atomic_size_t Seq{};
  };
  Slot* _buf;

  alignas(kCacheLineSize) atomic<size_t> _writeIndex{};
  alignas(kCacheLineSize) atomic<size_t> _readIndex{};

 public:
  mpmcqueue(size_t size) : _buffer_size(size), _buf(new Slot[size])
  {
    auto isPowerOf2 = (size >= 2 && size & (size - 1)) == 0;
    assert(isPowerOf2);
  }
  ~mpmcqueue()
  {
    delete[] _buf;
  }
  mpmcqueue(mpmcqueue&) = delete;
  mpmcqueue& operator=(mpmcqueue&) = delete;

  void push(T const& item)
  {
    auto writeIdx = _writeIndex.load(memory_order_acquire);
    for(;;)
    {
      auto seq = _buf[idx(writeIdx)].Seq.load(memory_order_acquire);
      if(seq % 4 == 0)  // seq is 0 and write available
      {
        if(_writeIndex.compare_exchange_weak(writeIdx, writeIdx + 1))
        {
          if(_buf[idx(writeIdx)].Seq.compare_exchange_weak(seq, seq + 1)) // seq is what we expect, add 1 for pending write
          {
            _buf[idx(writeIdx)].Data = item;
            _buf[idx(writeIdx)].Seq.store(seq + 2, memory_order_release);  // add 1 more to complete write
            return;
          }
        }
      }
      _mm_pause();
    }
  }

  bool try_push(T const& item, size_t attempts = 10)
  {
    auto writeIdx = _writeIndex.load(memory_order_acquire);
    for(auto _1 = 0; _1 < attempts; ++_1)
    {
      auto seq = _buf[idx(writeIdx)].Seq.load(memory_order_acquire);
      if(seq % 4 == 0)  // seq is 0 and write available
      {
        if(_writeIndex.compare_exchange_weak(writeIdx, writeIdx + 1))
        {
          if(_buf[idx(writeIdx)].Seq.compare_exchange_weak(seq, seq + 1)) // seq is what we expect, add 1 for pending write
          {
            _buf[idx(writeIdx)].Data = item;
            _buf[idx(writeIdx)].Seq.store(seq + 2, memory_order_release);  // add 1 more to complete write
            return true;
          }
        }
      }
      _mm_pause();
    }
    return false;
  }

  void pop(T& item)
  {
    auto readIdx = _readIndex.load(memory_order_acquire);
    for(;;)
    {
      auto seq = _buf[idx(readIdx)].Seq.load(memory_order_acquire);
      if(seq % 4 == 2)  // seq is 2 and read available
      {
        if(_readIndex.compare_exchange_weak(readIdx, readIdx + 1))
        {
          if(_buf[idx(readIdx)].Seq.compare_exchange_weak(seq, seq + 1)) // seq is what we expect, add 1 for pending read
          {
            _buf[idx(readIdx)].Data = item;
            _buf[idx(readIdx)].Seq.store(seq + 2, memory_order_release);  // add 1 more to complete read
            return;
          }
        }
      }
      _mm_pause();
    }
  }
  bool try_pop(T& item, size_t attempts = 10)
  {
    auto readIdx = _readIndex.load(memory_order_acquire);
    for(auto _1 = 0; _1 < attempts; ++_1)
    {
      auto seq = _buf[idx(readIdx)].Seq.load(memory_order_acquire);
      if(seq % 4 == 2)  // seq is 2 and read available
      {
        if(_readIndex.compare_exchange_weak(readIdx, readIdx + 1))
        {
          if(_buf[idx(readIdx)].Seq.compare_exchange_weak(seq, seq + 1)) // seq is what we expect, add 1 for pending read
          {
            _buf[idx(readIdx)].Data = item;
            _buf[idx(readIdx)].Seq.store(seq + 2, memory_order_release);  // add 1 more to complete read
            return true;
          }
        }
      }
      _mm_pause();
    }
    return false;
  }

 private:
  inline size_t idx(size_t i)
  {
    // NOTE: _buffer_size is power of 2
    return i & (_buffer_size - 1);
  }
};
}  // namespace ab
```