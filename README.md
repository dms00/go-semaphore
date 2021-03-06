# go-semaphore
Simple implementation of the classic semaphore construct using Go's sync primitives

```go
package semaphore

import (
	"sync"
)

type Semaphore struct {
	cnt  int
	cond *sync.Cond
}

func (s *Semaphore) V() {
	s.cond.L.Lock()
	s.cnt++
	s.cond.Signal()
	s.cond.L.Unlock()
}

func (s *Semaphore) P() {
	s.cond.L.Lock()

	for done := false; !done; {
		if s.cnt > 0 {
			s.cnt--
			done = true
		} else {
			s.cond.Wait()
		}
	}
	s.cond.L.Unlock()
}

func NewSemaphore(cnt int) *Semaphore {
	s := new(Semaphore)

	s.cond = sync.NewCond(new(sync.Mutex))
	s.cnt = cnt

	return s
}
```

One of the potential problems with the above implementation is that it suffers from the same issue as the Mutex and Cond structures of Go's sync package, if the user makes a copy of the struct, then things won't work correctly. So, for example, correct operation depends on the user not passing the object by value. Here's an alternative implementation that gets around this potential pitfall:

```go
package semaphore

import (
	"sync"
)

type semaphore struct {
	cnt  int
	cond *sync.Cond
}

func (s *semaphore) v() {
	s.cond.L.Lock()
	s.cnt++
	s.cond.Signal()
	s.cond.L.Unlock()
}

func (s *semaphore) p() {
	s.cond.L.Lock()

	for done := false; !done; {
		if s.cnt > 0 {
			s.cnt--
			done = true
		} else {
			s.cond.Wait()
		}
	}
	s.cond.L.Unlock()
}

func newSemaphore(cnt int) *semaphore {
	s := new(semaphore)

	s.cond = sync.NewCond(new(sync.Mutex))
	s.cnt = cnt

	return s
}

type Semaphore struct {
	sem *semaphore
}

func (s *Semaphore) P() {
	s.sem.p()
}

func (s *Semaphore) V() {
	s.sem.v()
}

func NewSemaphore(cnt int) *Semaphore {
	s := new(Semaphore)
	s.sem = newSemaphore(cnt)

	return s
}
```

Here's a third version that adds a non-blocking P that returns boolean to let caller know if it succeeded, renames P and V to the better-known Acquire and Release, and lastly panics if Release is attempted when there are no semaphores left to release.

```go
package semaphore

import (
	"sync"
)

type semaphore struct {
	initialCnt int
	cnt  int
	cond *sync.Cond
}

func (s *semaphore) v() {
	s.cond.L.Lock()
	defer s.cond.L.Unlock()

	if s.cnt < s.initialCnt {
		s.cnt++
		s.cond.Signal()
	} else {
		panic("No semaphores to release")
	}
}

func (s *semaphore) p() {
	s.cond.L.Lock()
	defer s.cond.L.Unlock()
	
	for done := false; !done; {
		if s.cnt > 0 {
			s.cnt--
			done = true
		} else {
			s.cond.Wait()
		}
	}
}

func (s *semaphore) tryP() bool {
	s.cond.L.Lock()
	defer s.cond.L.Unlock()
	result := false
	
	if s.cnt > 0 {
		s.cnt--
		result = true
	}
	return result
}

func newSemaphore(cnt int) *semaphore {
	return &semaphore{initialCnt: cnt, cnt: cnt, cond: sync.NewCond(new(sync.Mutex))}
}

type Semaphore struct {
	sem *semaphore
}

func (s *Semaphore) TryAcquire() bool {
	return s.sem.tryP()
}

func (s *Semaphore) Acquire() {
	s.sem.p()
}

func (s *Semaphore) Release() {
	s.sem.v()
}

func New(cnt int) *Semaphore {
	s := new(Semaphore)
	s.sem = newSemaphore(cnt)

	return s
}
```
