peb (PHP-Erlang Bridge) is an open-source PHP extension to run PHP as an Erlang cnode.

It's similar to the traditional php mysql function , you can easily communicate with an Erlang node.

The following is an example:

```
<?php 
$link = peb_connect('sadly-desktop@sadly-desktop',  'secret'); 
if (!$link) { 
    die('Could not connect: ' . peb_error()); 
} 

$msg = peb_vencode('[~p,~a]', array( 
                                   array($link,'getinfo')
                                  ) 
                 ); 
//The sender must include a reply address.  use ~p to format a link identifier to a valid Erlang pid.

peb_send_byname('pong',$msg,$link); 

$message = peb_receive($link);
$rs= peb_vdecode( $message) ;
print_r($rs);

$serverpid = $rs[0][0];

$message = peb_vencode('[~s]', array(
                                    array('how are you')
                                   )
                      );
peb_send_bypid($serverpid,$message,$link); 
//just demo for how to use peb_send_bypid

peb_close($link); 
?>  
```