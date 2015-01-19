---
layout: post
title:  "Pthread Barriers on OSX"
date:   2015-01-18 21:52:24
categories:
tags: c, osx, pthread
---

When I was porting the [mbw][1] to the OSX, I found that `pthread_barrier_t`
and related functions are missing. Fortunately it is not hard to
implement barriers based on `pthread_mutex_t` and `pthread_cond_t`. Usually
barriers are used to ensure that all threads have finished the current step
before they can proceed to the next step. The threads finished fast have
to wait other slow threads to catch up. In this example, I am going to
use the [sense-reverse barrier](http://cs.anu.edu.au/student/comp8320/lectures/aux/comp422-Lecture21-Barriers.pdf)
which is probably the easiest to implement.

The `pthread_barrier_t` contains a mutex protecting the whole data structure
and associated with the conditional variable. The `flag` changes every time
when the number of threads calling the `pthread_barrier_wait` is equal to the
number specified by the `num` variable which is set during initialisation.
Finally the `count` field records the number of threads have arrived at the
barrier, and if it equals `num`, all threads have arrived and they may pass
the barrier.

The `init` function is pretty straightforward, so let us focus on the `wait`
funciton. The first step is to acquire the `mutex` which protects the
whole barrier. Then a thread saves the flag in its local variable and increases
the `count` varibles. If the `count` equals to `num`, it means that the calling
thread is the last thread which needs to wake up other waiting threads.
The last thread resets the barrier counter to 0 and changes the barrier flag
to a different value. After that, the `pthread_cond_broadcast` is used
wake up other threads while the thread is holding the `mutex`. On the
other hand, other threads called `wait` function early execute
`pthread_cond_wait` to block themselves. After they are woken up by
the last thread, they check if the barrier `flag` value is different
from the value saved locally and exit the loop if so.


{% highlight c %}
#define PTHREAD_BARRIER_SERIAL_THREAD   1

typedef struct pthread_barrier {
  pthread_mutex_t         mutex;
  pthread_cond_t          cond;
  volatile uint32_t       flag;
  size_t                  count;
  size_t                  num;
} pthread_barrier_t;

int pthread_barrier_init(pthread_barrier_t *bar, int attr, int num)
{
  int ret = 0;
  if ((ret = pthread_mutex_init(&(bar->mutex), 0))) return ret;
  if ((ret = pthread_cond_init(&(bar->cond), 0))) return ret;
  bar->flag = 0;
  bar->count = 0;
  bar->num = num;
  return 0;
}

int pthread_barrier_wait(pthread_barrier_t *bar)
{
  int ret = 0;
  uint32_t flag = 0;

  if ((ret = pthread_mutex_lock(&(bar->mutex)))) return ret;

  flag = bar->flag;
  bar->count++;

  if (bar->count == bar->num) {
    bar->count = 0;
    bar->flag = 1 - bar->flag;
    if ((ret = pthread_cond_broadcast(&(bar->cond)))) return ret;
    if ((ret = pthread_mutex_unlock(&(bar->mutex)))) return ret;
    return PTHREAD_BARRIER_SERIAL_THREAD;
  }

  while (1) {
    if (bar->flag == flag) {
      ret = pthread_cond_wait(&(bar->cond), &(bar->mutex));
      if (ret) return ret;
      } else { break; }
    }

  if ((ret = pthread_mutex_unlock(&(bar->mutex)))) return ret;
  return 0;
}
{% endhighlight %}

[1]: https://github.com/yyshen/mbw
