# Node.js 进阶

## 事件循环

```
┌───────────────────────────┐
│         timers            │
│  setTimeout, setInterval │
├───────────────────────────┤
│     pending callbacks     │
├───────────────────────────┤
│       idle, prepare       │
├───────────────────────────┤
│          poll             │
│    I/O callbacks          │
├───────────────────────────┤
│         check             │
│    setImmediate           │
├───────────────────────────┤
│    close callbacks        │
└───────────────────────────┘
```

## Cluster 集群

```javascript
const cluster = require('cluster')
const http = require('http')
const numCPUs = require('os').cpus().length

if (cluster.isMaster) {
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork()
  }
  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} died`)
  })
} else {
  http.createServer((req, res) => {
    res.writeHead(200)
    res.end('Hello World')
  }).listen(8000)
}
```

## Worker Threads

```javascript
const { Worker, isMainThread, parentPort } = require('worker_threads')

if (isMainThread) {
  const worker = new Worker(__filename)
  worker.on('message', (msg) => console.log(msg))
  worker.postMessage('Hello')
} else {
  parentPort.on('message', (msg) => {
    parentPort.postMessage(`Echo: ${msg}`)
  })
}
```

## Stream 流

```javascript
const fs = require('fs')
const { Transform } = require('stream')

// 可读流
const readStream = fs.createReadStream('file.txt')

// 可写流
const writeStream = fs.createWriteStream('output.txt')

// 转换流
const upperCase = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase())
    callback()
  }
})

readStream.pipe(upperCase).pipe(writeStream)
```

## 内存管理

```javascript
// 查看内存使用
console.log(process.memoryUsage())

// 垃圾回收
if (global.gc) {
  global.gc()
}

// 监控内存
setInterval(() => {
  const used = process.memoryUsage()
  console.log(`Heap: ${Math.round(used.heapUsed / 1024 / 1024)}MB`)
}, 5000)
```
