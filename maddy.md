To run maddy in rootless mode

~/.config/systemd/user/prod-smtp.service
```
...
        --cap-add=CAP_NET_BIND_SERVICE \
        --userns=keep-id \
         docker.io/foxcpp/maddy:latest
```
