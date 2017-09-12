### SSH keep forwarding your key
The following can be used to login on servers behind a stepping stone, using the same priv/pub key.

client --> stepping stone --> server

`ssh -A user@steppingstone ssh user@server hostname`

Notice that there will be a socket created in `/tmp` on `steppingstone`. This could be abused by an attacker with access to `steppingstone`!
