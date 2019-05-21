<!--
author:   André Dietrich

email:    andre.dietrich@ovgu.de

version:  0.0.2

language: en

narrator: US English Female

comment:  Use LogicEmu to create any kind of logic emulations based on ASCII-diagrams.

@LogicEmu.eval: @LogicEmu._eval_(@uid)

@LogicEmu._eval_
<script>
let code = window.encodeBoard(`@input`);

let iframe = document.getElementById("logic_emu@0");

iframe.contentWindow.location.reload(true);

iframe.contentWindow.location.replace("https://liatemplates.github.io/LogicEmu/docs/index.html#code="+code);

"LIA: stop";
</script>

<iframe id="logic_emu@0" width="100%" height="400px" src=""></iframe>

@end

@LogicEmu.run: @LogicEmu._run_(@uid,```@0```)

@LogicEmu._run_
<script>
let code = window.encodeBoard(`@1`);

let iframe = document.getElementById("logic_emu@0");

iframe.contentWindow.location.reload(true);

iframe.contentWindow.location.replace("https://liatemplates.github.io/LogicEmu/docs/index.html#code="+code);
</script>

<iframe id="logic_emu@0" width="100%" height="400px" src=""></iframe>

@end

@onload
function LZ77Coder() {
  this.lz77MatchLen = function(text, i0, i1) {
    var l = 0;
    while(i1 + l < text.length && text[i1 + l] == text[i0 + l] && l < 255) {
      l++;
    }
    return l;
  };

  this.encodeString = function(text) {
    return arrayToString(this.encode(stringToArray(text)));
  };

  this.decodeString = function(text) {
    return arrayToString(this.decode(stringToArray(text)));
  };

  // Designed mainly for 7-bit ASCII text. Although the text array may contain values
  // above 127 (e.g. unicode codepoints), only values 0-127 are encoded efficiently.
  this.encode = function(text) {
    var result = [];
    var map = {};

    var encodeVarint = function(i, arr) {
      if(i < 128) {
        arr.push(i);
      } else if(i < 16384) {
        arr.push(128 | (i & 127));
        arr.push(i >> 7);
      } else {
        arr.push(128 | (i & 127));
        arr.push(128 | ((i >> 7) & 127));
        arr.push((i >> 14) & 127);
      }
    };

    for(var i = 0; i < text.length; i++) {
      var len = 0;
      var dist = 0;

      var sub = arrayToStringPart(text, i, 4);
      var s = map[sub];
      if(s) {
        for(var j = s.length - 1; j >= 0; j--) {
          var i2 = s[j];
          var d = i - i2;
          if(d > 2097151) break;
          var l = this.lz77MatchLen(text, i2, i);
          if(l > len) {
            len = l;
            dist = d;
            if(l > 255) break; // good enough, stop search
          }
        }
      }

      if(len > 2097151) len = 2097151;

      if(!(len > 5 || (len > 4 && dist < 16383) || (len > 3 && dist < 127))) {
        len = 1;
      }

      for(var j = 0; j < len; j++) {
        var sub = arrayToStringPart(text, i + j, 4);
        if(!map[sub]) map[sub] = [];
        if(map[sub].length > 1000) map[sub] = []; // prune
        map[sub].push(i + j);
      }
      i += len - 1;

      if(len >= 3) {
        if(len < 130) {
          result.push(128 + len - 3);
        } else {
          var len2 = len - 128;
          result.push(255);
          encodeVarint(len2, result);
        }
        encodeVarint(dist, result);
      } else {
        var c = text[i];
        if(c < 128) {
          result.push(c);
        } else {
          // Above-ascii character, encoded as unicode codepoint (not UTF-16).
          // Normally such character does not appear in circuits, but it could in comments.
          result.push(255);
          encodeVarint(c - 128, result);
          result.push(0);
        }
      }
    }
    return result;
  };

  this.decode = function(encoded) {
    var result = [];
    var temp;
    for(var i = 0; i < encoded.length;) {
      var c = encoded[i++];
      if(c > 127) {
        var len = c + 3 - 128;
        if(c == 255) {
          len = encoded[i++];
          if(len > 127) len += (encoded[i++] << 7) - 128;
          if(len > 16383) len += (encoded[i++] << 14) - 16384;
          len += 128;
        }
        dist = encoded[i++];
        if(dist > 127) dist += (encoded[i++] << 7) - 128;
        if(dist > 16383) dist += (encoded[i++] << 14) - 16384;

        if(dist == 0) {
          result.push(len);
        } else {
          for(var j = 0; j < len; j++) {
            result.push(result[result.length - dist]);
          }
        }
      } else {
        result.push(c);
      }
    }
    return result;
  };
}
function arrayToString(a) {
  var s = '';
  for(var i = 0; i < a.length; i++) {
    //s += String.fromCharCode(a[i]);
    var c = a[i];
    if (c < 0x10000) {
       s += String.fromCharCode(c);
    } else if (c <= 0x10FFFF) {
      s += String.fromCharCode((c >> 10) + 0xD7C0);
      s += String.fromCharCode((c & 0x3FF) + 0xDC00);
    } else {
      s += ' ';
    }
  }
  return s;
}
function stringToArray(s) {
  var a = [];
  for(var i = 0; i < s.length; i++) {
    //a.push(s.charCodeAt(i));
    var c = s.charCodeAt(i);
    if (c >= 0xD800 && c <= 0xDBFF && i + 1 < s.length) {
      var c2 = s.charCodeAt(i + 1);
      if (c2 >= 0xDC00 && c2 <= 0xDFFF) {
        c = (c << 10) + c2 - 0x35FDC00;
        i++;
      }
    }
    a.push(c);
  }
  return a;
}
// ignores the utf-32 unlike arrayToString but that's ok for now
function arrayToStringPart(a, pos, len) {
  var s = '';
  for(var i = pos; i < pos + len; i++) {
    s += String.fromCharCode(a[i]);
  }
  return s;
}
function RangeCoder() {
  this.base = 256;
  this.high = 1 << 24;
  this.low = 1 << 16;
  this.num = 256;
  this.values = [];
  this.inc = 8;

  this.reset = function() {
    this.values = [];
    for(var i = 0; i <= this.num; i++) {
      this.values.push(i);
    }
  };

  this.floordiv = function(a, b) {
    return Math.floor(a / b);
  };

  // Javascript numbers are doubles with 53 bits of integer precision so can
  // represent unsigned 32-bit ints, but logic operators like & and >> behave as
  // if on 32-bit signed integers (31-bit unsigned). Mask32 makes the result
  // positive again. Use e.g. after multiply to simulate unsigned 32-bit overflow.
  this.mask32 = function(a) {
    return ((a >> 1) & 0x7fffffff) * 2 + (a & 1);
  };

  this.update = function(symbol) {
    // too large denominator
    if(this.getTotal() + this.inc >= this.low) {
      var last = this.values[0];
      for(var i = 0; i < this.num; i++) {
        var d = this.values[i + 1] - last;
        d = (d > 1) ? this.floordiv(d, 2) : d;
        last = this.values[i + 1];
        this.values[i + 1] = this.values[i] + d;
      }
    }
    for(var i = symbol + 1; i < this.values.length; i++) {
      this.values[i] += this.inc;
    }
  };

  this.getProbability = function(symbol) {
    return [this.values[symbol], this.values[symbol + 1]];
  };

  this.getSymbol = function(scaled_value) {
    var symbol = this.binSearch(this.values, scaled_value);
    var p = this.getProbability(symbol);
    p.push(symbol);
    return p;
  };

  this.getTotal = function() {
    return this.values[this.values.length - 1];
  };

  // returns last index in values that contains entry that is <= value
  this.binSearch = function(values, value) {
    var high = values.length - 1, low = 0, result = 0;
    if(value > values[high]) return high;
    while(low <= high) {
      var mid = this.floordiv(low + high, 2);
      if(values[mid] >= value) {
        result = mid;
        high = mid - 1;
      } else {
        low = mid + 1;
      }
    }
    if(result > 0 && values[result] > value) result--;
    return result;
  };

  this.encodeString = function(text) {
    return arrayToString(this.encode(stringToArray(text)));
  };

  this.decodeString = function(text) {
    return arrayToString(this.decode(stringToArray(text)));
  };

  this.encode = function(data) {
    this.reset();

    var result = [1];
    var low = 0;
    var range = 0xffffffff;

    result.push(data.length & 255);
    result.push((data.length >> 8) & 255);
    result.push((data.length >> 16) & 255);
    result.push((data.length >> 24) & 255);

    for(var i = 0; i < data.length; i++) {
      var c = data[i];
      var p = this.getProbability(c);
      var total = this.getTotal();
      var start = p[0];
      var size = p[1] - p[0];
      this.update(c);
      range = this.floordiv(range, total);
      low = this.mask32(start * range + low);
      range = this.mask32(range * size);

      for(;;) {
        if(low == 0 && range == 0) {
          return null; // something went wrong, avoid hanging
        }
        if(this.mask32(low ^ (low + range)) >= this.high) {
          if(range >= this.low) break;
          range = this.mask32((-low) & (this.low - 1));
        }
        result.push((this.floordiv(low, this.high)) & (this.base - 1));
        range = this.mask32(range * this.base);
        low = this.mask32(low * this.base);
      }
    }

    for(var i = this.high; i > 0; i = this.floordiv(i, this.base)) {
      result.push(this.floordiv(low, this.high) & (this.base - 1));
      low = this.mask32(low * this.base);
    }

    if(result.length > data.length) {
      result = [0];
      for(var i = 0; i < data.length; i++) result[i + 1] = data[i];
    }

    return result;
  };

  this.decode = function(data) {
    if(data.length < 1) return null;
    var result = [];
    if(data[0] == 0) {
      for(var i = 1; i < data.length; i++) result[i - 1] = data[i];
      return result;
    }
    if(data[0] != 1) return null;
    if(data.length < 5) return null;

    this.reset();

    var code = 0;
    var low = 0;
    var range = 0xffffffff;
    var pos = 1;
    var symbolsize = data[pos++];
    symbolsize |= (data[pos++] << 8);
    symbolsize |= (data[pos++] << 16);
    symbolsize |= (data[pos++] << 24);
    symbolsize = this.mask32(symbolsize);

    for(var i = this.high; i > 0; i = this.floordiv(i, this.base)) {
      var d = pos >= data.length ? 0 : data[pos++];
      code = this.mask32(code * this.base + d);
    }
    for(var i = 0; i < symbolsize; i++) {
      var total = this.getTotal();
      var scaled_value = this.floordiv(code - low, (this.floordiv(range, total)));
      var p = this.getSymbol(scaled_value);
      var c = p[2];
      result.push(c);
      var start = p[0];
      var size = p[1] - p[0];
      this.update(c);

      range = this.floordiv(range, total);
      low = this.mask32(start * range + low);
      range = this.mask32(range * size);
      for(;;) {
        if(low == 0 && range == 0) {
          return null; // something went wrong, avoid hanging
        }
        if(this.mask32(low ^ (low + range)) >= this.high) {
          if(range >= this.low) break;
          range = this.mask32((-low) & (this.low - 1));
        }
        var d = pos >= data.length ? 0 : data[pos++];
        code = this.mask32(code * this.base + d);
        range = this.mask32(range * this.base);
        low = this.mask32(low * this.base);
      }
    }

    return result;
  };
}

window.encodeBoard = function(text) {
  var lz77 = (new LZ77Coder()).encodeString(text);
  var range = (new RangeCoder()).encodeString(lz77);
  return '0' + toBase64(range); // '0' = format version
}

function toBase64(text) {
  var result = btoa(text);
  result = result.split('=')[0];
  result = result.replace(new RegExp('\\+', 'g'), '-');
  result = result.replace(new RegExp('/', 'g'), '_');
  return result;
}
@end

-->

# LogicEmu - Template

                                   --{{0}}--
This document makes the logic emulator [logicemu](https://lodev.org/logicemu/)
available in [LiaScript](https://LiaScript.github.io). You can either run
simulations directly or associate them with an code-block, to make them editable
and executable.

__Try it on LiaScript:__

https://liascript.github.io/course/?https://raw.githubusercontent.com/liaTemplates/logicemu/master/README.md

__See the project on Github:__

https://github.com/liaTemplates/logicemu

                                   --{{1}}--
There are three ways to use this template. The easiest way is to use the
`import` statement and the url of the raw text-file of the master branch or any
other branch or version. But you can also copy the required functionionality
directly into the header of your Markdown document, see therefor the [last
slide](#4 "Implementation"). And of course, you could also clone this project
and change it, as you wish.

                                     {{1}}
1. Load the macros via

   `import: https://raw.githubusercontent.com/liaTemplates/logicemu/master/README.md`

2. Copy the definitions into your Project

3. Clone this repository on GitHub

## `@LogicEmu.run`

                                     {{0}}
Use the following block-macro, if you only want to execute a certain simulation.
That means, use the common Markdown code-block notation, add a syntax
highlighting and the macro `@LogicEmu.run` to the head of this block. That's it
the simulation will be automatically executed in LiaScript.

``` text @LogicEmu.run
s******>a****>l
       ^
s******


s**>a**>o**>l
   >    ^
s**>e**>a
       >
s******>e**>l
```


## `@LogicEmu.eval`

                                     {{0}}
If you want to allow editing, then simply attach the macro `@LogicEmu.eval` to
the end of your code-block.

```
"128"  "64"  "32"  "16"  "8"   "4"   "2"   "1"
  l     l     l     l     l     l     l     l
  ^     ^     ^     ^     ^     ^     ^     ^
"carry"l<o<a e o<a e o<a e o<a e o<a e o<a e o<a e o<a e s"carry"
^ ^^^/^ ^^^/^ ^^^/^ ^^^/^ ^^^/^ ^^^/^ ^^^/^ ^^^/
a e * a e * a e * a e * a e * a e * a e * a e *
^^^   ^^^   ^^^   ^^^   ^^^   ^^^   ^^^   ^^^
| |   | |   | |   | |   | |   | |   | |   | |
| |   | |   | |   | |   | |   | |   | |   | |
| ,---+-+---+-+---+-+---+-+---+-+---+-+---+-+-*
| ,---* *---+-+---+-+---+-+---+-+---+-+---+-+-+-*
| | *-------* *---+-+---+-+---+-+---+-+---+-+-+-+-*
| | | *-----------* *---+-+---+-+---+-+---+-+-+-+-+-*
| | | | *---------------* *---+-+---+-+---+-+-+-+-+-+-*
| | | | | *-------------------* *---+-+---+-+-+-+-+-+-+-*
| | | | | | *-----------------------* *---+-+-+-+-+-+-+-+-*
| | | | | | | *---------------------------* *-+-+-+-+-+-+-+-*
| | | | | | | |                               | | | | | | | |
| | | | | | | |                               | | | | | | | |
s s s s s s s s                               s s s s s s s s
"a" "1 6 3 1 8 4 2 1"                         "b" "1 6 3 1 8 4 2 1"
"2 4 2 6        "                             "2 4 2 6        "
"8              "                             "8              "
```
@LogicEmu.eval


## Implementation

                         --{{0}}--
The code shows how the two main macros were implemented by calling the macro
`@LogicEmu._eval_` and `@LogicEmu._run_`. All ASCII diagrams are passed to an
iframe, which contains a reduced version of the original logicemu. `@onload` is
required at the beginning to load all required functionionality that is needed
to encode the input.

````` js
@LogicEmu.eval: @LogicEmu._eval_(@uid)

@LogicEmu._eval_
<script>
let code = window.encodeBoard(`@input`);

let iframe = document.getElementById("logic_emu@0");

iframe.contentWindow.location.reload(true);

iframe.contentWindow.location.replace("https://liatemplates.github.io/LogicEmu/docs/index.html#code="+code);

"LIA: stop";
</script>

<iframe id="logic_emu@0" width="100%" height="400px" src=""></iframe>

@end

@LogicEmu.run: @LogicEmu._run_(@uid,```@0```)

@LogicEmu._run_
<script>
let code = window.encodeBoard(`@1`);

let iframe = document.getElementById("logic_emu@0");

iframe.contentWindow.location.reload(true);

iframe.contentWindow.location.replace("https://liatemplates.github.io/LogicEmu/docs/index.html#code="+code);
</script>

<iframe id="logic_emu@0" width="100%" height="400px" src=""></iframe>

@end

@onload
function LZ77Coder() {
  this.lz77MatchLen = function(text, i0, i1) {
    var l = 0;
    while(i1 + l < text.length && text[i1 + l] == text[i0 + l] && l < 255) {
      l++;
    }
    return l;
  };

  this.encodeString = function(text) {
    return arrayToString(this.encode(stringToArray(text)));
  };

  this.decodeString = function(text) {
    return arrayToString(this.decode(stringToArray(text)));
  };

  // Designed mainly for 7-bit ASCII text. Although the text array may contain values
  // above 127 (e.g. unicode codepoints), only values 0-127 are encoded efficiently.
  this.encode = function(text) {
    var result = [];
    var map = {};

    var encodeVarint = function(i, arr) {
      if(i < 128) {
        arr.push(i);
      } else if(i < 16384) {
        arr.push(128 | (i & 127));
        arr.push(i >> 7);
      } else {
        arr.push(128 | (i & 127));
        arr.push(128 | ((i >> 7) & 127));
        arr.push((i >> 14) & 127);
      }
    };

    for(var i = 0; i < text.length; i++) {
      var len = 0;
      var dist = 0;

      var sub = arrayToStringPart(text, i, 4);
      var s = map[sub];
      if(s) {
        for(var j = s.length - 1; j >= 0; j--) {
          var i2 = s[j];
          var d = i - i2;
          if(d > 2097151) break;
          var l = this.lz77MatchLen(text, i2, i);
          if(l > len) {
            len = l;
            dist = d;
            if(l > 255) break; // good enough, stop search
          }
        }
      }

      if(len > 2097151) len = 2097151;

      if(!(len > 5 || (len > 4 && dist < 16383) || (len > 3 && dist < 127))) {
        len = 1;
      }

      for(var j = 0; j < len; j++) {
        var sub = arrayToStringPart(text, i + j, 4);
        if(!map[sub]) map[sub] = [];
        if(map[sub].length > 1000) map[sub] = []; // prune
        map[sub].push(i + j);
      }
      i += len - 1;

      if(len >= 3) {
        if(len < 130) {
          result.push(128 + len - 3);
        } else {
          var len2 = len - 128;
          result.push(255);
          encodeVarint(len2, result);
        }
        encodeVarint(dist, result);
      } else {
        var c = text[i];
        if(c < 128) {
          result.push(c);
        } else {
          // Above-ascii character, encoded as unicode codepoint (not UTF-16).
          // Normally such character does not appear in circuits, but it could in comments.
          result.push(255);
          encodeVarint(c - 128, result);
          result.push(0);
        }
      }
    }
    return result;
  };

  this.decode = function(encoded) {
    var result = [];
    var temp;
    for(var i = 0; i < encoded.length;) {
      var c = encoded[i++];
      if(c > 127) {
        var len = c + 3 - 128;
        if(c == 255) {
          len = encoded[i++];
          if(len > 127) len += (encoded[i++] << 7) - 128;
          if(len > 16383) len += (encoded[i++] << 14) - 16384;
          len += 128;
        }
        dist = encoded[i++];
        if(dist > 127) dist += (encoded[i++] << 7) - 128;
        if(dist > 16383) dist += (encoded[i++] << 14) - 16384;

        if(dist == 0) {
          result.push(len);
        } else {
          for(var j = 0; j < len; j++) {
            result.push(result[result.length - dist]);
          }
        }
      } else {
        result.push(c);
      }
    }
    return result;
  };
}
function arrayToString(a) {
  var s = '';
  for(var i = 0; i < a.length; i++) {
    //s += String.fromCharCode(a[i]);
    var c = a[i];
    if (c < 0x10000) {
       s += String.fromCharCode(c);
    } else if (c <= 0x10FFFF) {
      s += String.fromCharCode((c >> 10) + 0xD7C0);
      s += String.fromCharCode((c & 0x3FF) + 0xDC00);
    } else {
      s += ' ';
    }
  }
  return s;
}
function stringToArray(s) {
  var a = [];
  for(var i = 0; i < s.length; i++) {
    //a.push(s.charCodeAt(i));
    var c = s.charCodeAt(i);
    if (c >= 0xD800 && c <= 0xDBFF && i + 1 < s.length) {
      var c2 = s.charCodeAt(i + 1);
      if (c2 >= 0xDC00 && c2 <= 0xDFFF) {
        c = (c << 10) + c2 - 0x35FDC00;
        i++;
      }
    }
    a.push(c);
  }
  return a;
}
// ignores the utf-32 unlike arrayToString but that's ok for now
function arrayToStringPart(a, pos, len) {
  var s = '';
  for(var i = pos; i < pos + len; i++) {
    s += String.fromCharCode(a[i]);
  }
  return s;
}
function RangeCoder() {
  this.base = 256;
  this.high = 1 << 24;
  this.low = 1 << 16;
  this.num = 256;
  this.values = [];
  this.inc = 8;

  this.reset = function() {
    this.values = [];
    for(var i = 0; i <= this.num; i++) {
      this.values.push(i);
    }
  };

  this.floordiv = function(a, b) {
    return Math.floor(a / b);
  };

  // Javascript numbers are doubles with 53 bits of integer precision so can
  // represent unsigned 32-bit ints, but logic operators like & and >> behave as
  // if on 32-bit signed integers (31-bit unsigned). Mask32 makes the result
  // positive again. Use e.g. after multiply to simulate unsigned 32-bit overflow.
  this.mask32 = function(a) {
    return ((a >> 1) & 0x7fffffff) * 2 + (a & 1);
  };

  this.update = function(symbol) {
    // too large denominator
    if(this.getTotal() + this.inc >= this.low) {
      var last = this.values[0];
      for(var i = 0; i < this.num; i++) {
        var d = this.values[i + 1] - last;
        d = (d > 1) ? this.floordiv(d, 2) : d;
        last = this.values[i + 1];
        this.values[i + 1] = this.values[i] + d;
      }
    }
    for(var i = symbol + 1; i < this.values.length; i++) {
      this.values[i] += this.inc;
    }
  };

  this.getProbability = function(symbol) {
    return [this.values[symbol], this.values[symbol + 1]];
  };

  this.getSymbol = function(scaled_value) {
    var symbol = this.binSearch(this.values, scaled_value);
    var p = this.getProbability(symbol);
    p.push(symbol);
    return p;
  };

  this.getTotal = function() {
    return this.values[this.values.length - 1];
  };

  // returns last index in values that contains entry that is <= value
  this.binSearch = function(values, value) {
    var high = values.length - 1, low = 0, result = 0;
    if(value > values[high]) return high;
    while(low <= high) {
      var mid = this.floordiv(low + high, 2);
      if(values[mid] >= value) {
        result = mid;
        high = mid - 1;
      } else {
        low = mid + 1;
      }
    }
    if(result > 0 && values[result] > value) result--;
    return result;
  };

  this.encodeString = function(text) {
    return arrayToString(this.encode(stringToArray(text)));
  };

  this.decodeString = function(text) {
    return arrayToString(this.decode(stringToArray(text)));
  };

  this.encode = function(data) {
    this.reset();

    var result = [1];
    var low = 0;
    var range = 0xffffffff;

    result.push(data.length & 255);
    result.push((data.length >> 8) & 255);
    result.push((data.length >> 16) & 255);
    result.push((data.length >> 24) & 255);

    for(var i = 0; i < data.length; i++) {
      var c = data[i];
      var p = this.getProbability(c);
      var total = this.getTotal();
      var start = p[0];
      var size = p[1] - p[0];
      this.update(c);
      range = this.floordiv(range, total);
      low = this.mask32(start * range + low);
      range = this.mask32(range * size);

      for(;;) {
        if(low == 0 && range == 0) {
          return null; // something went wrong, avoid hanging
        }
        if(this.mask32(low ^ (low + range)) >= this.high) {
          if(range >= this.low) break;
          range = this.mask32((-low) & (this.low - 1));
        }
        result.push((this.floordiv(low, this.high)) & (this.base - 1));
        range = this.mask32(range * this.base);
        low = this.mask32(low * this.base);
      }
    }

    for(var i = this.high; i > 0; i = this.floordiv(i, this.base)) {
      result.push(this.floordiv(low, this.high) & (this.base - 1));
      low = this.mask32(low * this.base);
    }

    if(result.length > data.length) {
      result = [0];
      for(var i = 0; i < data.length; i++) result[i + 1] = data[i];
    }

    return result;
  };

  this.decode = function(data) {
    if(data.length < 1) return null;
    var result = [];
    if(data[0] == 0) {
      for(var i = 1; i < data.length; i++) result[i - 1] = data[i];
      return result;
    }
    if(data[0] != 1) return null;
    if(data.length < 5) return null;

    this.reset();

    var code = 0;
    var low = 0;
    var range = 0xffffffff;
    var pos = 1;
    var symbolsize = data[pos++];
    symbolsize |= (data[pos++] << 8);
    symbolsize |= (data[pos++] << 16);
    symbolsize |= (data[pos++] << 24);
    symbolsize = this.mask32(symbolsize);

    for(var i = this.high; i > 0; i = this.floordiv(i, this.base)) {
      var d = pos >= data.length ? 0 : data[pos++];
      code = this.mask32(code * this.base + d);
    }
    for(var i = 0; i < symbolsize; i++) {
      var total = this.getTotal();
      var scaled_value = this.floordiv(code - low, (this.floordiv(range, total)));
      var p = this.getSymbol(scaled_value);
      var c = p[2];
      result.push(c);
      var start = p[0];
      var size = p[1] - p[0];
      this.update(c);

      range = this.floordiv(range, total);
      low = this.mask32(start * range + low);
      range = this.mask32(range * size);
      for(;;) {
        if(low == 0 && range == 0) {
          return null; // something went wrong, avoid hanging
        }
        if(this.mask32(low ^ (low + range)) >= this.high) {
          if(range >= this.low) break;
          range = this.mask32((-low) & (this.low - 1));
        }
        var d = pos >= data.length ? 0 : data[pos++];
        code = this.mask32(code * this.base + d);
        range = this.mask32(range * this.base);
        low = this.mask32(low * this.base);
      }
    }

    return result;
  };
}

window.encodeBoard = function(text) {
  var lz77 = (new LZ77Coder()).encodeString(text);
  var range = (new RangeCoder()).encodeString(lz77);
  return '0' + toBase64(range); // '0' = format version
}

function toBase64(text) {
  var result = btoa(text);
  result = result.split('=')[0];
  result = result.replace(new RegExp('\\+', 'g'), '-');
  result = result.replace(new RegExp('/', 'g'), '_');
  return result;
}
@end
`````

                         --{{1}}--
If you want to minimize loading effort in your LiaScript project, you can also
copy this code and paste it into your main comment header, see the code in the
raw file of this document.

{{1}} https://raw.githubusercontent.com/liaTemplates/logicemu/master/README.md
