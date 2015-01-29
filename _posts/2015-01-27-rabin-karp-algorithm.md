---
layout: post
title: Rabin-Karp Algorithm
date: 2015-01-27 20:13:17
---

# Intro

When I was reading the go standard library, I found `string.Index` (which
returns the first index of occursion of string `s` in string `t`) used
Rabin-Karp algorithm.  So I implemented the one.

# Straight Forward Algorithm

When you try to find string `s` in string `t`, a simple way is to compare `s` with
`t[0:len(s)]`, `t[1:len(s)+1]`...


{% highlight go %}
for i := 0; i <= len(t)-len(s); i++ {
	if s == t[i : i+len(s)] {
		return i
	}
}
return -1
{% endhighlight %}

The problem is this algorithm is `O(|s||t|)`. The wasteful part is comparing
`s` with `t[i:i+len(s)]` every time in the loop. Which lead us to Rabin-Karp
algorithm.

# Rabin-Karp Algorithm

The only difference and the key idea of Rabin-Karp algorithm is comparing two
strings using __rolling hash__ value of them. Rolling hash skip computation by
re-uses previous hash value. The following is the definition of it.

$$h_i := t_i p^k + t_{i+1}p^{k-1} + ... + t_{i+k}p^0$$

where $$p$$ is a random prime and $$k$$ is the length of $$s$$. It is a simple
polynomial, thus it is easy to calculate $$h_{i+1}$$ from $$h_i$$.

{% highlight go %}
for i := 1; i < lt-ls+1; i++ {
	ht -= int(t[i-1]) * pow(prime, len(s)-1) // remove the first character
	ht = ht*prime + int(t[i+len(s)-1])       // append the next character
	if hs == ht && s == t[i:i+len(ls)] {
		return i
	}
}
{% endhighlight %}

The source code is available at [gist](https://gist.github.com/shouichi/52e7464cbfe1bcd80c71).
Thanks for reading, happy hacking!

# References

[9. Table Doubling, Karp-Rabin](https://www.youtube.com/watch?v=BRO7mVIFt08)
