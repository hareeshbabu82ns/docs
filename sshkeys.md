## Generate SSH Key
```bash
$> ssh-keygen
```
* key pair will be generated under **/home/<username>/.ssh/id_rsa**
* **/home/<username>/.ssh/id_rsa.pub** will be public key to be shared
* useful to connect to servers with SHA key instead password login
* append the *.pub key value into **~/.ssh/authorized_keys** file

* [ssh config from digitalocean](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)
