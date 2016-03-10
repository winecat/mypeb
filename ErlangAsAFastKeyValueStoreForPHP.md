# Erlang as a Fast Key Value Store for PHP #

In this post I want to show you some of the neat things that can be done with the PHP-Erlang Bridge extension: A Key Value Store.

Erlang comes packed with a Key Value store in the form of the ETS module. This is database is pretty fast and efficient for storing the Erlang terms in memory.

I tried a proof of concept with the PHP extension and I obtained impressive results: Storing 150.000+ items in the ETS in 1 second! All that running on my Macbook Pro.

What I did was to write a PHP class wrapping the calls to the Erlang ETS module like for example:

```
public function insert($key, $value)
{
 $x = peb_encode("[~a, {~a, ~s}]", array(array(
     $this->name,
     array($key, $value)
   )));
 $result = peb_rpc("ets", "insert", $x, $this->link);
 return peb_decode($result);
}
```

Maps to this call in Erlang:

```
> ets:insert(tablename, {key, value}).
```

You can see the full code example here: http://gist.github.com/324043

So here are the steps:

- Install the PEB extension from source
- Start Erlang with this Command: erl -sname node -setcookie abc
- Create the ETS table: ets:new(test, [set, named\_table, public]).
- Save the gist to a file and run it: php ets.php

So while this is a very simple proof of concept, I just wanted to illustrate some of the cool things that can be done with this extension. For example the speed of encoding/decoding from Erlang to PHP is pretty decent as well as the communication speed.

Please let me know your thoughts about it in the comments.

Article Taken from: http://obvioushints.blogspot.com/2010/03/erlang-as-fast-key-value-store-for-php.html