# mvcc_read_conflict 
Hyperledger fabric is one of the frameworks to develop enterprise blockchain applications. It became more popular in recent times and many companies started implementing this framework to develop applications. One can start working on hyperledger by following fabric [docs](https://hyperledger-fabric.readthedocs.io/en/release-1.4/).

 Here, in this post i want to share my experience with MVCC_READ_CONFLICT issue.

## Scenario

Once i gave myself a task to transfer a huge data from one database to blockchain database. I initially converted all the data from source database to a JSON object and it is a mysql database. So i had lot of tables in it. Then i extracted that part of data that i want to transfer. I am using v1.4.3 of fabric and that has provided a js file using 'fabric-network' module to submit transactions. Compared to earlier versions of same code, this code made things much simpler. I made an array of arrays where each array is a transaction to be submitted. As the data is huge, i have more transactions to submit. The js script returns a promise after successful submission of transaction. So i thought of using Promise.all(). And it is there i first experienced MVCC_READ_CONFLICT. Below is the code i used. 

```javascript
    var promises = [invoke(['func1','arg1']), invoke(['func2','arg2']), ...so on]
    Promise.all(promises).then(function(data) {
        console.log(data)
    }).catch(() => {
        console.log("Error")
    })
```

## Why MVCC_READ_CONFLICT?

When you submit a transaction, a peer generates a read/write set which is used to get signed by orderer and commit into ledger. During the time one set generation and commit into ledger, another transaction came into play and it changed the read/write set of the pervious transaction. This lead to MVCC_READ_CONFLICT. It is simply two or more processes trying to update same key or value at same time.

## Solution

We have 'BatchTimeout' in configtx.yaml under Orderer section. By decreasing this value to 1s or so, one can improve latency of the code and submitting multiple transactions.

```yaml
Orderer: &OrdererDefaults
    OrdererType: solo
    Addresses:
        - orderer.example.com:7050
    BatchTimeout: 1s
    BatchSize:
        MaxMessageCount: 10
        AbsoluteMaxBytes: 99 MB
        PreferredMaxBytes: 512 KB
```

Now here is the solution i made use of. I have executed a loop and resolved a promise after submitting a transaction and displaying the response. By this i have transfered all the data sequentially.

```javascript
    var promises = [invoke(['func1','arg1']), invoke(['func2','arg2']), ...so on]
    (function loop(i) {
        if (i < promises.length) new Promise((resolve, reject) => {
            return invoke(promises[i]).then((result) => {
                console.log(result)
                resolve()
            })
        }).then(loop.bind(null, i+1))
    })(0);
```
