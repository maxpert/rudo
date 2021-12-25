# RuES - JMES Evaluation as Service 

RuES is a minimal JMES expression evaluation side-car, that uses [JMESPath](https://jmespath.org/) and it can handle 
arbitrary JSON. Which effectively makes it general purpose logical expression evaluation engine, just like 
[some](https://zerosteiner.github.io/rule-engine/) Python libraries that used to evaluate 
logical expression. This in turn can allow you implement complex stuff like Rule engine, 
RBAC, or Policy engines etc. 

Here is what makes RuES special:

 - **Lean and Zippy** - Checkout initial benchmarks below. Under `20 MB` with single CPU one will easily do 10K RPS. 
 - **Zero restarts** - Add/remove rules on fly by making changes in `rules.hjson` without restarting.
 - **HTTP & JSON** - Ubiquitous! No custom protocols, no shenanigans.
 - **UNIX philosophy** - Only evaluates rules, no fancy hooks or integrations. Dead simple!

## Usage

Make sure you have `rules.hjson` in your current working directory when launching `rues`. Given following example
rules:

```hjson
{
  example_one: "value == `2`"
  example_two: "a.b"
}
```

Each rule is exposed under `/eval/{rule_name}` as `POST` endpoint, which in turn can be posted payload to evaluate
the expression. Simple use `curl` to test:

```
> curl -X POST http://localhost:8080/eval/example_one -H 'Content-Type: application/json' -d '{"value": 2}'
{"expression":"value == `2`","rule":"example_one","is_truthy":true,"exp_value":"true"}
> curl -X POST http://localhost:8080/eval/example_two -H 'Content-Type: application/json' -d '{"a": {"b": "Hello"}}'
{"expression":"a.b","rule":"example_two","is_truthy":true,"exp_value":"\"Hello\""}
```

## Additional functions

In addition to [built-in functions](https://jmespath.org/proposals/functions.html#built-in-functions) of JMES, there 
additional are following additional functions:

 - `string[] match(string $regex, string $element)` - Returns an array of all groups of regex matching or a `null` if
   there is no match. Regex specs can be found [here](https://github.com/rust-lang/regex). Regexes are compiled 
   and cached in LRU order.
 - `number from_datetime(string $element, string $format = 'rfc3339')` (To be implemented yet) - Converts 
   datetime in given format to a timestamp. The timestamp then in turn can be used to 
   do comparisons or reformatting. 
 - `string to_datetime(number $element, , string $format = 'rfc3339')` (To be implemented yet) - Converts
   timestamp to a given string format.
 - `bool in_geo_fence(number[] $center, number $radius, number[] $element)` (To be implemented yet) - Returns `true`
   or `false` if the `$element` lies within the `$radius` of `$center`.
 - `number[][] filter_in_geo_fence(number[] $center, number $radius, number[][] $elements) ` (To be implemented yet) - 
   Returns all elements that lie within geo fence of given radius and center.
 - `bool valid_email(string $element)` (To be implemented yet) - Returns `true` or `false` based on email format.
 - `bool match_glob(string $pattern, string $element)` (To be implemented yet) - Returns `true` or `false` 
   if the `$element` is a glob match of the `$pattern`.

## Configuration variables

 - `CONFIG_PATH` - path to rules file, file can be `.json`, `.yaml`, or `.hjson`. Default: `rules.hjson`
 - `BIND_ADDRESS` - service address to bind to. Default: `0.0.0.0:8080`

## Benchmarks

My brief stress testing shows with a single CPU core (single worker), 3 rules, and payload size of 1.6 KB. Server was 
easily able to handle 10K RPS (even with sustained load) under **19 MB of RSS** memory footprint, and a p99 of 4ms.

```
$ cat vegeta_attack.txt | vegeta attack -duration=10s -rate=10000 | vegeta report 
Requests      [total, rate, throughput]         100000, 10000.20, 9999.80
Duration      [total, attack, wait]             10s, 10s, 394.927µs
Latencies     [min, mean, 50, 90, 95, 99, max]  107.266µs, 811.954µs, 285.329µs, 2.128ms, 2.654ms, 4.517ms, 12.373ms
Bytes In      [total, mean]                     9566673, 95.67
Bytes Out     [total, mean]                     166000000, 1660.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:100000  
Error Set:
```

With two CPU cores (two workers), the results were even better:
```
$ cat vegeta_attack.txt | vegeta attack -duration=10s -rate=10000 | vegeta report
Requests      [total, rate, throughput]         100000, 10000.30, 10000.08
Duration      [total, attack, wait]             10s, 10s, 217.653µs
Latencies     [min, mean, 50, 90, 95, 99, max]  111.479µs, 270.125µs, 219.274µs, 413.215µs, 564.181µs, 1.021ms, 8.184ms
Bytes In      [total, mean]                     9566673, 95.67
Bytes Out     [total, mean]                     166000000, 1660.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:100000  
Error Set:
```

All the rules, and data has been shipped under `stress_test`. Feel free to share your results, and I will be more 
than happy to include your results.