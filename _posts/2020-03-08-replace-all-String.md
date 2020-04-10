---
layout: post
title: replace_all method in rust String
image: /img/hello_world.jpeg
---

## Beginning

I was trying to escape some chars (` `, `=`, `,`). This chars should be escaped in [influx db line protocol](https://v2.docs.influxdata.com/v2.0/reference/syntax/line-protocol/#special-characters)

First what I have done was to search [String doc] but there is only [replace] method. So I have decided to implement it manually.

**First implementation**

Great! Using [replace] from std we can easily build our first implementation:
```rust
#[inline]
fn escape_with_replace(s: &String) -> String {
    s.replace("=", r#"\="#)
        .replace(",", r#"\,"#)
        .replace(" ", r#"\ "#)
}
```
Hym.. Rust is zero cost abstraction language - but [`String::replace`] take `&self` so it need to copy this tree times. Or maybe it is able to optimize it?
**Let's find out.**

## Benchmarking

To measure our implementations we will use this two str.
```
const NO_ESCAPE: &str = r#"Abcdefghijklmnouódsałπ≠²³4tonżðąq"#;
const TO_ESCAPE: &str = r#"asdddas\  =d =das=sddsałπ≠²³4tonż"#;
```

We also need to compare our implementation again something. On stack overflow is [answer](https://stackoverflow.com/a/34606128/5190508) that explain how replace patterns with regex. We used this implementation in `no_escpae_regex` and `to_escape_regex` bench function.

We use some name convention for bench functions:
- bench function that start with `no_escape` will use `NO_ESCAPE` const value to test escaping impl. Test version without escaping.
- bench function that start with `to_escape` will use `TO_ESCAPE` const value to test escaping impl. Test version with escaping.

```
// #[bench] is only available on nightly
#[cfg(all(feature = "nightly", test))]
mod bench {
    const NO_ESCAPE: &str = r#"Abcdefghijklmnouódsałπ≠²³4tonżðąq"#;
    const TO_ESCAPE: &str = r#"asdddas\  =d =das=sddsałπ≠²³4tonż"#;
    use super::*;
    extern crate test;
    use regex::Regex;

    #[bench]
    fn no_escpae_regex(b: &mut test::Bencher) {
        let s = String::from(NO_ESCAPE);
        let re = Regex::new("[, =]").unwrap();
        b.iter(|| re.replace_all(&s, r#"\$0"#).to_string())
    }

    #[bench]
    fn to_escape_regex(b: &mut test::Bencher) {
        let s = String::from(TO_ESCAPE);
        let re = Regex::new("[, =]").unwrap();
        b.iter(|| re.replace_all(&s, r#"\$0"#).to_string())
    }

    #[bench]
    fn no_escape_std_replace(b: &mut test::Bencher) {
        let s = String::from(NO_ESCAPE);
        b.iter(|| escape_with_replace(&s))
    }

    #[bench]
    fn to_escape_std_replace(b: &mut test::Bencher) {
        let s = String::from(TO_ESCAPE);
        b.iter(|| escape_with_replace(&s))
    }
}
```
Guess which implementation do better!

```
test escape::bench::no_escpae_regex                ... bench:         104 ns/iter (+/- 8)
test escape::bench::to_escape_regex                ... bench:         990 ns/iter (+/- 101)

test escape::bench::no_escape_std_replace          ... bench:         279 ns/iter (+/- 27)
test escape::bench::to_escape_std_replace          ... bench:         560 ns/iter (+/- 37)
```
So our implementation is ~2 times better in escaping but make 2.5 times worse in **no escape** scenario compare to regex. That is unacceptable as escape case would be rather rare.

## So let's just call replace only if needed (second impl)
```
#[inline]
fn escape_with_contains_replace(s: String) -> String {
    let s = if s.contains("=") {
        s.replace("=", r#"\="#)
    } else {
        s
    };
    let s = if s.contains(",") {
        s.replace(",", r#"\,"#)
    } else {
        s
    };

    let s = if s.contains(" ") {
        s.replace(" ", r#"\ "#)
    } else {
        s
    };

    s
}

```

Our result in bench:
```
test escape::bench::no_escape_std_contains_replace ... bench:         198 ns/iter (+/- 17)
test escape::bench::to_escape_std_contains_replace ... bench:         592 ns/iter (+/- 31)
```
First of all I am not sure this benches are correct since there is `clone()` call in every iteration. But we are doing slightly better in **no escape** scenario but still 2 times worse than regex.

## Third approach
The [second answer](https://stackoverflow.com/a/34610817/5190508) on our question in stack overflow suggest nice solution. Let's just iterate and collect it into string.

```
fn escape_map_collect(s: String) {
    s.chars()
        .map(|c| match c {
            '=' => r#"\="#,
            ',' => r#"\,"#,
            ' ' => r#"\ "#,
            c => c.as_str(), // unfortunately we can't represent char as &str.
                             // c => c.to_string().as_str() // lifetime problem
        })
        .collect()
}
```
Unfortunately code above does not compile.

So we can't change from [char] to [&str]. But we could use [`push_str`] and [`push`] methods.

```
#[inline]
fn escape_find_push_uo(s: &String) -> String {
    // like `String::contains` but looking for three chars at once
    let escape = s
        .find(|c| match c {
            '=' | ',' | ' ' => true,
            _ => false,
        })
        .map(|_| true)
        .unwrap_or(false);

    // should we escape?
    if escape {
        let mut escaped_string = String::with_capacity(s.capacity());
        for c in s.chars() {
            match c {
                '=' => escaped_string.push_str(r#"\="#),
                ',' => escaped_string.push_str(r#"\,"#),
                ' ' => escaped_string.push_str(r#"\ "#),
                c => escaped_string.push(c),
            }
        }
        escaped_string
    } else {
        s.clone()
    }
}
```

Let _optimize_ this a little. We can use value returned by [`String::find`] function to start replacing from that char.
```rust
#[inline]
fn escape_find_push(s: &String) -> String {
    let opt_begin = s.find(|c| match c {
        '=' | ',' | ' ' => true,
        _ => false,
    });

    // begin contains position where first item was found
    if let Some(begin) = opt_begin {
        let mut escaped_string = String::with_capacity(s.len() + 8);
        escaped_string.push_str(&s[..begin]); // from 0 to first item(without it)
        for c in s[begin..].chars() { // skip copied chars
            match c {
                '=' => escaped_string.push_str(r#"\="#),
                ',' => escaped_string.push_str(r#"\,"#),
                ' ' => escaped_string.push_str(r#"\ "#),
                c => escaped_string.push(c),
            }
        }
        escaped_string
    } else {
        s.clone()
    }
}
```

And add the benches:

```
    // skip rest of bench (..)

    fn no_escape_std_find_push_uo(b: &mut test::Bencher) {
        let s = String::from(NO_ESCAPE);
        b.iter(|| escape_find_push_uo(&s))
    }

    #[bench]
    fn to_escape_std_find_push_uo(b: &mut test::Bencher) {
        let s = String::from(TO_ESCAPE);
        b.iter(|| escape_find_push_uo(&s))
    }

    #[bench]
    fn no_escape_find_push(b: &mut test::Bencher) {
        let s = String::from(NO_ESCAPE);
        b.iter(|| escape_find_push(&s))
    }

    #[bench]
    fn to_escape_find_push(b: &mut test::Bencher) {
        let s = String::from(TO_ESCAPE);
        b.iter(|| escape_find_push(&s))
    }
```

Guess the results!
```
test escape::bench::no_escape_std_find_push_uo     ... bench:          68 ns/iter (+/- 6)
test escape::bench::to_escape_std_find_push_uo     ... bench:         229 ns/iter (+/- 21)

test escape::bench::no_escape_find_push            ... bench:          84 ns/iter (+/- 5)
test escape::bench::to_escape_find_push            ... bench:         204 ns/iter (+/- 38)
```

Oh so we are doing slightly better in **no escape** scenario than regex and ~11 times better in **escape scenario**. That's great!

Are we happy with our implementation? I am not - performance is good but we still can do some improvements:
- we can prevent repeating constants so much
- do we need both `push_str` and `push` calls?
- do we need two ways of copy String? (`clone()` and `escaped_string.push_str(&s[..begin]`)

```
#[inline]
fn escape_find_push2(s: &String) -> String {
    let s_len = s.len();
    let is_escape_char = move |c| match c {
        '=' | ',' | ' ' => true,
        _c => false,
    };
    let begin = s.find(is_escape_char).unwrap_or(s_len);

    // we add extra bytes to prevent unnecessary copy
    let mut escaped_string = String::with_capacity(s_len + 8);
    escaped_string += &s[..begin];
    for c in s[begin..].chars() {
        if is_escape_char(c) {
            escaped_string.push('\\');
            escaped_string.push(c);
        } else {
            escaped_string.push(c);
        }
    }
    escaped_string
}
```

Now the implementation is nice and can be easy generated. So let's do this easy step:

```
#[inline]
fn escape<P>(is_escape_char: P, s: &String) -> String
where
    P: Fn(char) -> bool,
{
    let s_len = s.len();
    let begin = s.find(|c| is_escape_char(c)).unwrap_or(s_len);

    // we add extra bytes to prevent unnecessary copy
    let mut escaped_string = String::with_capacity(s_len + 8);
    escaped_string += &s[..begin];
    for c in s[begin..].chars() {
        if is_escape_char(c) {
            escaped_string.push('\\');
            escaped_string.push(c);
        } else {
            escaped_string.push(c);
        }
    }
    escaped_string
}
```

Will generated version be as good as our `escape_find_push2()`? Of course - Rust is zero cost abstraction!

But let's bench it!
```
    #[bench]
    fn no_escape_general(b: &mut test::Bencher) {
        let s = String::from(NO_ESCAPE);
        b.iter(|| {
            escape(
                |c| match c {
                    '=' | ',' | ' ' => true,
                    _c => false,
                },
                &s,
            )
        })
    }

    #[bench]
    fn to_escape_general(b: &mut test::Bencher) {
        let s = String::from(TO_ESCAPE);
        b.iter(|| {
            escape(
                |c| match c {
                    '=' | ',' | ' ' => true,
                    _c => false,
                },
                &s,
            )
        })
    }

```

Bench results:
```
test escape::bench::no_escape_general              ... bench:          83 ns/iter (+/- 9)
test escape::bench::to_escape_general              ... bench:         202 ns/iter (+/- 15)
```

### All benches
rust version: 1.43.0-nightly (2890b37b8 2020-03-06)
```
test escape::bench::no_escpae_regex                ... bench:         104 ns/iter (+/- 8)
test escape::bench::to_escape_regex                ... bench:         990 ns/iter (+/- 101)

test escape::bench::no_escape_std_replace          ... bench:         279 ns/iter (+/- 27)
test escape::bench::to_escape_std_replace          ... bench:         560 ns/iter (+/- 37)

test escape::bench::no_escape_std_contains_replace ... bench:         198 ns/iter (+/- 17)
test escape::bench::to_escape_std_contains_replace ... bench:         592 ns/iter (+/- 31)

test escape::bench::no_escape_std_find_push_uo     ... bench:          68 ns/iter (+/- 6)
test escape::bench::to_escape_std_find_push_uo     ... bench:         229 ns/iter (+/- 21)

test escape::bench::no_escape_find_push            ... bench:          84 ns/iter (+/- 5)
test escape::bench::to_escape_find_push            ... bench:         204 ns/iter (+/- 38)

test escape::bench::no_escape_find_push2           ... bench:          77 ns/iter (+/- 6)
test escape::bench::to_escape_find_push2           ... bench:         204 ns/iter (+/- 52)

test escape::bench::no_escape_general              ... bench:          83 ns/iter (+/- 9)
test escape::bench::to_escape_general              ... bench:         202 ns/iter (+/- 15)
```

### Conclusions

- `replace()` called many times in a row is not optimal. What make me a little surprised - since replace is `#[inline]` according to doc.
- it's quite easy to implement own escape function with just std
- in my personal opinion `escape<P>` with `|c| match c {..}` closure is more readable. (You don't need to be familiar with regex).
-


## Edit
Two days later I discovered that implementation of `no_escpe_std_replace` and `to_escpe_std_replace` wasn't done correct. Can you found how we can make it better?;

```
#[inline]
fn escape_with_replace2(s: &String) -> String {
    s.replace('=', r#"\="#)
        .replace(',', r#"\,"#)
        .replace(' ', r#"\ "#)
}
```

So the difference is `char` vs `&str` for matches

And bench results:
```
test escape::bench::no_escape_std_replace2         ... bench:         134 ns/iter (+/- 13)
test escape::bench::to_escape_std_replace2         ... bench:         401 ns/iter (+/- 34)
```

If I would found it earlier I will not write this blog post since std version is good enough!.


[String doc]:https://doc.rust-lang.org/std/string/struct.String.html
[replace]:https://doc.rust-lang.org/std/string/struct.String.html#method.replace

