---
name: Issue report
about: Issue template
title: ''
labels: ''
assignees: ''

---

# Issue: [TEST-01] Goroutine leaks in the signer process

## Impact

When an Index pool is initiated with two tokens A: B and the weight rate = 1:2, then no user can buy token A with token B.  
The root cause is the error in pow. It seems like the dev tries to implement Exponentiation by squaring.  

https://github.com/sushiswap/trident/blob/9130b10efaf9c653d74dc7a65bde788ec4b354b5/contracts/pool/IndexPool.sol#L286-L291

There's no bracket for `for`.  
The IndexPool is not functional. I consider this is a high-risk issue.  

## Description

The thornode signer process causes goroutine leaks whenever a timeout occurs before a transaction has been signed. In some situations, this could lead to an exhaustion of resources.  

The runWithContext function takes as parameters the signAndBroadcast function, responsible for signing transactions, and a context object with a timeout of five minutes. If the transaction is signed within five minutes and signAndBroadcast returns an error, the error will be received by an unbuffered channel, the function will return, and the goroutine will be destroyed. However, if a transaction is not signed within five minutes, the ctx.Done() channel will be read, and the function will return without cleaning the goroutine resources. The more often this situation occurs, the more resources will be held up by the signer process.

## Proof of Concept

When we initiated the pool with 2:1.  
```js
deployed_code = encode_abi(["address[]","uint136[]","uint256"], [
    (link.address, dai.address),
    (2*10**18,  10**18),
    10**13
])
```

## Recommendations

Short term, make channel ch a buffered channel of size 1. That way, the goroutine will be cleaned and destroyed when the function returns regardless of which case occurs first.  
Long term, run GCatch against goroutine-heavy packages to detect the mishandling of channel bugs. Basic instances of this issue can also be detected by running this Semgrep rule.
