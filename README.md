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
