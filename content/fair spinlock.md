
---
title: Fair Spinlock
draft: false
tags:
  - cpp
---
 
Got asked this in an interview recently and it stumped me. A better name might be **ticket lock**.

```cpp
class fair_spinlock
{
  std::atomic<uint32_t> _ticket{};
  std::atomic<uint32_t> _counter{};

 public:
  fair_spinlock() = default;
  ~fair_spinlock() = default;
  fair_spinlock(fair_spinlock&) = delete;
  fair_spinlock& operator=(fair_spinlock&) = delete;
  fair_spinlock(fair_spinlock&&) = delete;
  fair_spinlock& operator=(fair_spinlock&&) = delete;

  void lock()
  {
    auto ticket = _ticket.fetch_add(1, std::memory_order_relaxed);
    while(_counter.load() != ticket)
	{
		_mm_pause();
    }
  }

  void unlock()
  {
    ++_counter;
  }
};
```