# Erlang as Session Storage for PHP #

In the last few days I been playing with the PHP extension mypeb which allows us to connect to Erlang from PHP. As a simple example to show what we can do with this extension I will create a PHP class that will be used as the session\_save\_hanlder for PHP. By deafult PHP stores the sessions in the file system, but if you want to share the sessions over several servers, then we have to resort to using a database or Memcached. I will like to try something different by using this class to interact with an Erlang node that will act as the in memory storage for our sessions using ETS tables.

To modify the session\_save\_hanlder we have to call the function session\_set\_save\_handler and provide there six callbacks that will be used for the following actions: opening and closing a session, reading and writing to the session, destroying the sessions and garbage collect the sessions. You can read more about this function in the PHP manual.

Let's start by creating the open callback. In our example, we will have a method ErlangSessionHandler::open that will connect to the Erlang node.

```
public function open($save_path, $session_name)
{
  if(null === $this->link)
  {
    $this->link = peb_connect($this->host, $this->erlang_cookie, $this->conn_timeout);
    if(!$this->link)
    {
      throw new Exception(sprintf("Can't connect to the erlang node %s using erlang_cookie %s", $this->host, $this->erlang_cookie));
    }
  }
  return $this->link;
}
```

There we use the function **peb\_connect** that expects three parameters, the **host** to connect to, the Erlang **secret cookie** and an optional connection **timeout**. This function will return a resource identifier of the connection or false on failure. For our basic example we will define those three parameters as members of the class ErlangSessionHandler like this:

```
protected $host = 'server@127.0.0.1';
protected $erlang_cookie = 'ABCDEFGHI';
protected $conn_timeout = 5;
```

The method **ErlangSessionHandler::close** is very straightforward:

```
public function close()
{
  if(is_resource($this->link))
  {
    peb_close($this->link);
  }
}
```

When called it will close the connection to the Erlang node by calling: **peb\_close** passing the resource identifier as parameter.

Then we have **ErlangSessionHandler::read**

```
public function read($session_id)
{
  $x = peb_encode("[~s]", array(array($session_id)));
  $result = peb_rpc("session_handler", "read", $x, $this->link);
  $rs = peb_decode($result);
  $data = $rs[0];
  return is_array($data) ? '' : $data;
}
```

This method will be passed the **$session\_id** which we will forward to the Erlang node. To accomplish that first we need to create an Erlang Message by calling the function peb\_encode, which expects a format string and the value we want to encode into that format. In our case we need a list which will contain our session id as only element. Once we encoded the variable we will send it to Erla	ng by calling peb\_rpc. This function works similar to the Erlang rpc:call function. We need to specify the Module and the Function to call as the first two parameters. The third parameter is the message we want to send, and the last parameter is the result identifier. This function will return the result of the RPC call or false on error. The session information will be the first element of the $rs variable. In case of an error in the Erlang side, $data will be an array instead of a string, that's why we return and empty string in that case. Take into account that the session read callback must return an empty string in the case that there is no session information for the provided id.

Now lets see the code for the **ErlangSessionHandler::write** method.

```
public function write($session_id, $session_data)
{
  $x = peb_encode("[~s, ~s]", array(array($session_id, $session_data)));
  $result = peb_rpc("session_handler", "write", $x, $this->link);
  unset($result);
  return true;
}
```

This method expects two parameters, the session id and the information to store. The code here is pretty similar to the one for ErlangSessionHandler::read. We encode the PHP variables as Erlang terms and we send them to the session server via **peb\_rpc**.

Session destroy is also similar to the implementation of read, but we call peb\_rpc("session\_handler", "destroy", $x, $this->link); instead of "read":

```
public function destroy($session_id)
{
  $x = peb_encode("[~s]", array(array($session_id)));
  $result = peb_rpc("session_handler", "destroy", $x, $this->link);
  unset($result);
  return true;
}
```

The code for **ErlangSessionHandler::gc** is also simple:

```
public function gc($max_expire_time)
{
  $x = peb_encode('[~i]', array(array($max_expirte_time)));
  $result = peb_rpc("session_handler", "gc", $x, $this->link);
  $rs = peb_decode($result);
  return $rs;
}
```

Then to use the our class as **session\_handler** we add this to our PHP code.

```
$sh = new ErlangSessionHandler();

session_set_save_handler(
array($sh,"open"),
array($sh,"close"),
array($sh,"read"),
array($sh,"write"),
array($sh,"destroy"),
array($sh,"gc")
);

session_start();
```

Then the final piece of the puzzle is to start the Erlang Session Server that is implemented in the file **session\_handler.erl**.

```
$ erl -sname server
(server@localhost)1> c(session_handler).
(server@localhost)1> session_handler:start().
```

And that's it. We can start playing with our Erlang Session Storage Server.

NOTE:

First I want to make clear that this code is not meant to be used in production systems. Is just an example of what can be done with the mypeb extension.

I'm planning in writing a more robust session server in Erlang using the Mnesia database as a way to provide more reliable storage. With Mnesia we can easily distribute the session data across multiple servers, and in some of them store the sessions to disc.

Regarding the session save handler code, I would like to port it into the mypeb extension as native C code along with some php.ini settings that can provide the Erlang node to connect to, the secret cookie, connection timeout, etc.

As a final step I would like to do a small clean up to the API of the mypeb extension.

Article Taken from: http://obvioushints.blogspot.com/2009/12/erlang-as-session-storage-for-php.html