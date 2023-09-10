# Find The Longest Subset With Sum Less Than K

In the course of finding an internship I applied to bunch of the top tech companies. Some refused my application, other ignore it. One gave me a programming challenge (I can't say the name unfortunately) in which I got a couple of interesting problems to solve.

One of the problems consist of finding the longest subset with a sum less than a given integer. Note that we don't need to return the subset itself but only the length of it.

The input data are:
* 'K' the max sum
* 'a' an array of integers

my first approach as always is "don't reinvent the wheel" and Google it to find the optimal algorithm. However this is a challenge in which my programming skills are tested so I grabbed pen and papers and start with the most obvious way.

- start at the "start" position
- start summing up until we go over K
- save the length 
- increment the "start" position
- and repeat 
In python

```python
def length_max_subarray(a, K):
    length = 0
    start = 0
    finish = 0
    while(start<len(a)):
        finish = 0
        s = sum(a[start:start+finish])
        while(start+finish<=len(a) and s<=K ):
            if finish > length:
                length = finish
            finish+=1
            s = sum(a[start:start+finish])
        start+=1
    return length
```
As you can see this is in the order of `O(N2)` of course we can save some steps -- for example we don't need to reset the finish to zero we can directly set it up to length

```python
....
    while(start<len(a)):
        finish = length
        s = sum(a[start:start+finish])
        while(start+finish<=len(a) and s<=K ):
....

```
However this doesn't give us a performance boost we need to find a solution to lower the order of magnitude

## Then it got to me.
Thinking of this again, I got the idea of using a head and tail and keep the current sum between them.

```
head = tail = 0 


-while the tail is less than the length of the array 
   -increment the tail and update the sum
   -if the sum go over the max sum increment the head until the sum is less than K
   - while incrementing the tail check if we have a new max_length
```

The final code:

```python
def length_max_subarray(array, K):
    head, tail = 0, 0
    length = 0
    current_sum = 0
    while(tail<len(array)):
        if current_sum + array[tail]<=K:
            current_sum += array[tail]
            tail+=1
            if tail-head > length:
                length = tail-head
        else:
            current_sum -= array[head]
            head+=1

    return length
```
As you can see this is in the order of `O(N)`.

Unfortunately I was busy with exams so I didn't give my full potential. But it was fun nonetheless :D.

