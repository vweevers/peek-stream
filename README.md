# peek-stream

Transform stream that lets you peek the first line before deciding how to parse it. Fork of [`mafintosh/peek-stream`](https://github.com/mafintosh/peek-stream) (1.1.3) to add [#8](https://github.com/mafintosh/peek-stream/pull/8).

```
npm install @vweevers/peek-stream
```

[![Test](https://img.shields.io/github/actions/workflow/status/vweevers/peek-stream/test.yml?branch=master&label=test)](https://github.com/vweevers/peek-stream/actions/workflows/test.yml)

## Usage

``` js
var peek = require('@vweevers/peek-stream')
var ldjson = require('ldjson-stream')
var csv = require('csv-parser')

var isCSV = function(data) {
  return data.toString().indexOf(',') > -1
}

var isJSON = function(data) {
  try {
    JSON.parse(data)
    return true
  } catch (err) {
    return false
  }
}

// call parser to create a new parser
var parser = function() {
  return peek(function(data, swap) {
    // maybe it is JSON?
    if (isJSON(data)) return swap(null, ldjson())

    // maybe it is CSV?
    if (isCSV(data)) return swap(null, csv())

    // we do not know - bail
    swap(new Error('No parser available'))
  })
}
```

The above parser will be able to parse both line delimited JSON and CSV

``` js
var parse = parser()

parse.write('{"hello":"world"}\n{"hello":"another"}\n')
parse.on('data', function(data) {
  console.log(data) // prints {hello:'world'} and {hello:'another'}
})
```

Or

``` js
var parse = parser()

parse.write('test,header\nvalue-1,value-2\n')
parse.on('data', function(data) {
  console.log(data) // prints {test:'value-1', header:'value-2'}
})
```

Per default `data` is the first line (or the first `65535` bytes if no newline is found).
To change the max buffer pass an options map to the constructor

``` js
var parse = peek({
  maxBuffer: 10000
}, function(data, swap) {
  ...
})
```

If you want to emit an error if no newline is found set `strict: true` as well.

Normally `data` is written to the swapped stream as-is. To override `data`, pass a third argument to `swap`:

``` js
var parse = peek(function(data, swap) {
  // Strip utf-8 byte order mark
  if (data.byteLength >= 3 && data[0] === 0xef && data[1] === 0xbb && data[2] === 0xbf) {
    data = data.subarray(3)
  }

  if (isCSV(data)) {
    // The csv stream will not receive the BOM
    swap(null, csv(), data)
    return
  }

  swap(new Error('No parser available'))
})
```

## License

MIT
